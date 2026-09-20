# Nominal Sum Types via `enum union` and `switch` Expressions

| Field           | Value                                                                                                              |
| ---             | ---                                                                                                                |
| DIP:            |                                                                                                                    |
| Review Count:   |                                                                                                                    |
| Authors:        | Jared Hanson                                                                                                       |
| Implementation: | [https://github.com/dlang/dmd/pull/23744](https://www.google.com/search?q=https://github.com/dlang/dmd/pull/23744) |
| Status:         |                                                                                                                    |

## Abstract
Add nominal `enum union` declarations and `switch` expressions to the D programming language, providing algebraic data types (discriminated unions) with unboxed layouts, closed stack-based polymorphism, and static exhaustiveness checking to enable powerful programming patterns.

## Contents
- [Rationale](#rationale)
- [Prior Work](#prior-work)
- [Description](#description)
- [Variant Kinds](#variant-kinds)
  - [Tuple Variants](#tuple-variants)
  - [Struct Variants](#struct-variants)
  - [Bare Type Variants](#bare-type-variants)
  - [Alias Variants](#alias-variants)
- [Enum Union Members](#enum-union-members)
- [.init and Default Construction](#init-and-default-construction)
- [In-Memory Layout](#in-memory-layout)
- [Memory Safety](#memory-safety)
- [Implicit Construction](#implicit-construction)
- [Niche Optimization (not yet implemented)](#niche-optimization-not-yet-implemented)
- [Switch Expressions](#switch-expressions)
- [Switch Expression Patterns](#switch-expression-patterns)
  - [Destructuring Patterns](#destructuring-patterns)
  - [Variable Patterns](#variable-patterns)
  - [Type Name Patterns](#type-name-patterns)
  - [Exhaustiveness & Redundancy Checking](#exhaustiveness--redundancy-checking)
  - [Default Arms](#default-arms)
  - [Pattern Guards](#pattern-guards)
- [Side-Effects](#side-effects)
- [Metaprogramming](#metaprogramming)
  - [Traits](#traits)
- [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
- [Reference](#reference)
- [Copyright & License](#copyright--license)
- [History](#history)

## Rationale
Sum types (also known as tagged unions, discriminated unions, or algebraic data types) are a foundational construct in type-safe programming. They allow expressing that a value is one of several distinct variants, with static guarantees that all cases are handled. This eliminates invalid state representations and missing branch errors at compile time.

While D supports raw unions, they are untagged, inherently `@system` to access, and lack compiler-managed tag coordination, destructor synthesis, and branch exhaustiveness. Library solutions like `std.sumtype` implement tagged unions via complex template metaprogramming. However, library implementations suffer from slow compilation throughput, opaque diagnostic errors, and the inability to exploit compiler memory layout optimizations such as niche optimization.

## Prior Work
* Rust enums, the main inspiration for this feature.
* Swift enums, also a large source of inspiration, and its switch expressions.
* C# unions and switch expressions.
* Java switch expressions.
* Odin unions.
* Zig's `union(enum)`.
* D's `std.sumtype` and `std.variant`.
* [Richard (Rikki) Cattermole's DIP for type unions and match expressions](https://forum.dlang.org/post/nhbiwarfrlqqffegkhsf@forum.dlang.org).

## Description
This DIP proposes 2 new constructs for the D language: `enum union` as a language-level discriminated union type, and switch expressions which are used to inspect these unions at runtime.

An enum union is declared using the `enum union` keyword sequence. Variant declarations in an enum union must start with the keyword `case`, and take a comma-separated list of names or types, delimited by a semicolon:
```d
enum union NetworkPacket
{
    case Data(const(ubyte)[]);
    case Ping(ulong);
    case Reset(ushort);
    case Heartbeat();
    case EndOfStream();
}

// OR alternatively
enum union NetworkPacket
{
    case Data(const(ubyte)[]),
         Ping(ulong),
         Reset(ushort),
         Heartbeat(),
         EndOfStream();
}
```

In the second case, if the enum union **only** contains variant declarations, the trailing semicolon may be omitted.
```d
enum union NetworkPacket
{
    case Data(const(ubyte)[]),
         Ping(ulong),
         Reset(ushort),
         Heartbeat(),
         EndOfStream() // Ok
}

enum union NetworkPacket
{
    case Data(const(ubyte)[]),
         Ping(ulong),
         Reset(ushort),
         Heartbeat(),
         EndOfStream() // Error

        void doSomething() {} 
}
```

Each variant declaration represents one of the possible values that an enum union may take on. There are different kinds of variants that serve different functions.

## Variant Kinds

### Tuple-like Variants
Tuple-like variants (or "tuple variants" for short) consist of a name and a list of parameters required to construct that variant. Each parameter may be named, but it is not required. It is allowed to mix named and unnamed (positional) parameters when declaring a tuple variant:
```d
enum union DrawCommand
{
    // Unnamed positional parameters
    case MoveTo(double, double);
    case LineTo(double, double);

    // Named positional parameters
    case Circle(double x, double y, double radius);
    case Text(string content, double x, double y, ubyte fontSize);

    // Mixed named and unnamed positional parameters
    case Arc(double, double, double radius, double sweepAngle);
}
```

If a variant has zero parameters (like `Heartbeat` and `EndOfStream` in the `NetworkPacket` example above), it is a unit variant. Unit variants must be declared with empty parentheses (`case Heartbeat()`).

Tuple-like and unit variants **do not** have their own standalone types. They are lowered as members of the containing `enum union` aggregate, and calling `DrawCommand.Circle(...)` invokes a synthesized static factory function returning a `DrawCommand`.

### Struct Variants
Struct variants are declared as a struct declaration block within the variant list:
```d
enum union PaymentEvent
{
    case CardCharge {
        string token;
        ulong amountCents;
        string currency;
        bool require3DSecure;
    },
    
    BankTransfer {
        string iban;
        string bic;
        ulong amountCents;
        string reference;
    },
    
    RefundIssued {
        ulong originalTxId;
        ulong refundAmountCents;
        string reason;
    },
}
```

Struct variants are only allowed to declare fields; they cannot declare methods, constructors, destructors, or invariants.

Unlike tuple variants, struct variants define an exported nominal struct type under the namespace of the union (`PaymentEvent.CardCharge`). Instances of struct variants implicitly convert to the enclosing `enum union`.
```d
PaymentEvent.CardCharge charge = PaymentEvent.CardCharge("my token", 1_000_000_000, "CAD", true);

void takesCardCharge(PaymentEvent.CardCharge c);
void takesPaymentEvent(PaymentEvent p);
takesCardCharge(charge);   // OK
takesPaymentEvent(charge); // OK

auto event = PaymentEvent.RefundIssued(42, 42, "");
takesCardCharge(event); // Error
```

### Bare Type Variants
Bare-type variants directly embed a type as a case in the enum union without wrapping it in a named constructor:
```d
enum union ConfigValue
{
    case bool;
    case long;
    case double;
    case string;
    case string[];
}
```

Bare type variants must be unique within an aggregate. Because they have no tag identifier, bare type variants are initialized via direct assignment:
```d
ConfigValue c = false;             // Holds variant `bool`
c = ["some", "cool", "strings"];   // Holds variant `string[]`
c = [["some", "cool", "strings"]]; // Error: no variant `string[][]` in enum union `ConfigValue`
```

Target selection follows standard D overload resolution rules:
```d
enum union Nums 
{
    case int;
    case long;
}

Nums n1 = 0;   // Ok: 0 is typed as `int` (MATCH.exact for `int`, MATCH.convert for `long`)
Nums n2 = 0L;  // Ok: MATCH.exact for `long`
```

Ambiguities occur only when an assigned value requires conversions of equal rank to multiple variants:
```d
enum union Pointers
{
    case int*;
    case long*;
}

// Error: `Pointers.__ctor` called with argument types `(typeof(null))` matches multiple overloads after qualifier conversion:
Pointers p = null; 
```

Bare type variants are accessed and eliminated using `switch` expressions.

### Alias Variants
Alias variants are a shorthand syntax that allows aliasing an external type to a different name while also declaring it as a variant in the enum union. This is useful for embedding types that are in different modules, but have conflicting names, or for renaming an embedded external type:
```d
enum union KeyInput
{
    case Windows = sys.platform.win32.events.Win32KeyEvent;
    case Wayland = sys.platform.linux.wayland.WaylandKeyEvent;
    case Darwin  = sys.platform.darwin.cocoa.CocoaKeyEvent;
}

auto input = KeyInput.Windows(32, true);
```

This is semantically equivalent to:
```d
enum union KeyInput
{
    alias Windows = sys.platform.win32.events.Win32KeyEvent;
    case Windows;
    ...etc.
}
```

They're also useful for embedding multiple instantiations of a templated type, while giving each instantiation a name:
```d
struct RingBuffer(T)
{
    string channelId;
    T[] items;
}

struct Future(T)
{
    ulong taskId;
    T result;
    bool isReady;
}

enum union WorkerTask
{
    case IngestQueue  = RingBuffer!string;
    case PacketStream = RingBuffer!(ubyte[]);
    case MetricResult = Future!double;
}

auto task = WorkerTask.PacketStream("eth0", [[0xAA, 0xBB], [0xCC]]);
```

## Enum Union Members
Like other aggregates in D, enum unions can contain members, member functions, constructors, destructors, aliases, etc.
```d
enum union NetworkMessage
{
    case Heartbeat();
    case Text(string content, string encoding);
    case Binary(ubyte[]);
    case Status(int statusCode, string statusText);

    ulong timestamp;
    uint sequenceNumber;

    this(string rawInput, uint seq = 0, ulong ts = 0)
    {
        if (rawInput == "ping")
            this = Heartbeat;
        else if (rawInput.length == 0)
            this = Status(400, "Empty Payload");
        else
            this = Text(rawInput, "UTF-8");

        this.sequenceNumber = seq;
        this.timestamp = ts;
    }

    size_t byteLength() const
    {
        return switch (this)
        {
            case Heartbeat()          => 0,
            case Text(content, ...)   => content.length,
            case Binary(bytes)        => bytes.length,
            case Status(status, text) => status.sizeof + text.length,
        };
    }

    bool isControlFrame() const => switch (this) // Shorthand method syntax is okay too
    {
        case Heartbeat => true,
        default        => false,
    };
}
```

Inside constructors and member functions, `this` refers to the union aggregate itself, not the active variant. Member fields may be accessed via `this.<field>`. Inside constructors, definite assignment analysis ensures that `this` has been assigned a variant on all execution paths before member fields are accessed or the constructor returns.

## .init and Default Construction
Every enum union provides an `.init` value, which is the `.init` state of its first declared variant (in syntactic order).
```d
enum union Option(T)
{
    case None();
    case Some(T);
}

Option!int opt;
assert(opt.__tag == 0);
assert(opt == Option!int.None);
```

If all payload types in the enum union disable default construction (`@disable this();`), default construction is disabled for the enum union as well.

## In-Memory Layout
An enum union's layout is equivalent to the layout of a struct with a ubyte member for a tag (unless it is optimized away), an empty unit struct **iff** the union contains 1 or more unit variants, and a struct declaration per named variant to carry their fields. Any member fields declared of the enum union come after the union payload.
```d
enum union NetworkMessage
{
    case Heartbeat();
    case Text(string content, string encoding);
    case Binary(ubyte[]);
    case Status(int statusCode, string statusText);

    ulong timestamp;
    uint sequenceNumber;
}
```

Lowers to:
```d
struct NetworkMessage
{
    ubyte __tag;

    // Created once and shared by every unit variant.
    private struct __UnitPayload {}

    private struct __TextPayload
    {
        string content;
        string encoding;
    }

    private struct __BinaryPayload
    {
        ubyte[] __payload;
    }

    struct Status
    {
        int statusCode;
        string statusText;
    }

    private union
    {
        __UnitPayload   __heartbeat;
        __TextPayload   __text;
        __BinaryPayload __binary;
        Status          __status;
    }

    ulong timestamp;
    uint sequenceNumber;
}
```

## Memory Safety
Enum unions guarantee memory integrity across variant transformations:
* **Value-Copy Pattern Bindings**: Pattern match bindings extract payloads by value into the arm's lexical scope. This isolates bound variables from the parent aggregate, preventing aliasing hazards where an active reference could be corrupted by a concurrent re-tagging or reassignment of the parent union during arm evaluation. Once D has a sound method of tracking ownership and borrowing, like Rikki's DFA analyzer, then binding by ref can be allowed.

* **RAII Lifecycle Dispatch**: If any variant contains an elaborate destructor (~this()), the compiler synthesizes an aggregate destructor that inspects the discriminant tag and invokes the destructor of the active variant.

* **Deterministic Re-Tagging**: Reassigning a sum type from variant A to variant B executes active destruction of A, writes the new discriminant tag, and blits payload B in an exception-safe sequence, eliminating use-after-free, memory leaks, and type confusion.

* **Rejection of Non-Copyable Payloads**: Because pattern extraction relies on value-copy isolation to remain @safe without a borrow checker, and because DMD currently lacks general definite assignment analysis and sub-field dynamic drop flags to safely relocate unboxed union members without risking double-destruction on scope exit, move-only types are rejected at declaration time.

## Implicit Construction
Enum unions are implicitly constructed in the following cases: the struct-style construction via assignment discussed previously, when a function takes an enum union as an argument, and when a function returns an enum union:
```d
enum union Option(T)
{
    case None = typeof(null);
    case Some = T;
}

// 1. Implicit construction on function return
Option!string findUsername(int userId)
{
    if (userId <= 0)
        // Implicitly constructs typeof(null) -> Option!string.None via MATCH.exact
        return null;

    // Implicitly constructs string -> Option!string.Some via MATCH.exact
    return "guest_user";
}

void sendAlert(Option!string recipient)
{
    switch (recipient)
    {
        case Some name => notifyUser(name),
        case None      => broadcastToAll(),
    }
}

void main()
{
    // 2. Implicit construction on function parameter passing
    sendAlert("ops_team"); // Lowers to sendAlert(Option!string("ops_team"))
    sendAlert(null);       // Lowers to sendAlert(Option!string(null))
}
```

Implicit construction only performs one level of conversion; nested sum types are not implicitly constructed across multiple levels of nesting.

## Niche Optimization (not yet implemented)
When an enum union contains unit variants alongside non-nullable references, pointers (`T*`), or class references, the compiler can exploit invalid or unused bit patterns to encode the unit state:

* `Option!(int*)`: The null pointer address `0x0` represents `None`.
* `sizeof(Option!(int*)) == 8` (on 64-bit platforms), incurring zero byte overhead for the tag.

Furthermore, modern operating systems leave page zero (`0x0000` through `0x0FFF`) unmapped, providing up to 4,096 distinct niche addresses that can represent multiple unit variants within pointer-sized types without allocating an auxiliary tag byte.

## Switch Expressions
Switch expressions inspect and eliminate enum unions using exhaustive pattern matching:
```d
enum union NetworkPacket
{
    case Data(const(ubyte)[]);
    case Ping(ulong);
    case Reset(ushort);
    case Heartbeat();
    case EndOfStream();
}

string describePacket(NetworkPacket pkt)
{
    return switch (pkt)
    {
        // Positional payload extraction binding variables by value
        case Data(bytes)     => format("Data payload (%d bytes)", bytes.length),
        case Ping(timestamp) => format("Ping probe: timestamp=%d", timestamp),
        case Reset(code)     => format("Connection reset with code %d", code),

        // Parameterless unit variants match directly by tag name
        case Heartbeat       => "Keep-alive heartbeat received",
        case EndOfStream     => "End of transmission stream",
    };
}

```

Every arm must start with the `case` keyword (or `default`) and produce a single value. All arms must unify to a common type via a Least Upper Bound (LUB) calculation. Arms that diverge via `throw` or `assert(0)` evaluate to `noreturn`, which unifies with any type.

Only one unguarded catch-all pattern is permitted per variant. Duplicate unguarded patterns are rejected at compile time:
```d
switch (pkt)
{
    case Data(bytes)  => ...,
    case Data(bytes2) => ..., // Error: redundant match arm; pattern is unreachable
}
```

Within a switch expression over an enum union, the union's variants are automatically introduced into lexical scope via an implicit `with (typeof(subject))`.

## Switch Expression Patterns

### Destructuring Patterns
Destructuring patterns unpack payload fields for unit, tuple, and struct variants. They also allow fields to be omitted using `...` syntax:
```d
enum union U
{
    case Unit();
    case Struct { int n; double d; string s; };
}

int result = switch (U.Struct(42, 6.9, "asdf"))
{
    case Unit()         => 0,
    case Struct(n, ...) => n * 2, // Ignores fields d and s
};
```

The ... syntax allows ALL fields to be omitted:
```d
    case Unit(...) => "unit", // This is valid because ... means "0 or more fields"
    case Struct(...) => "No access to Struct's fields here",
```

The `...` syntax can also be prefixed with a variable name:
```d
    case Struct(n, rest...) => typeof(rest).stringof, // AliasSeq!(double, string)
    case Unit(rest...) => typeof(rest).stringof,      // This is also valid, AliasSeq!()
```

This syntax transforms the remaining fields in the struct into an AliasSeq, similar to `T...` template paramter syntax. This syntax is also supported for tuple and unit variants.

### Variable Patterns
Variable patterns take the form `case Type name =>` and bind the entire active payload to a local variable by value:

```d
enum union A
{
    case int;
    case Unit();
    case Tuple(int n, string);
    case Struct { double d; bool b; };
    case MyStruct = ExternalStruct;
}

switch (A.Tuple(42, "asdf"))
{
    case int n      => ...,
    case Unit u     => ...,
    case Tuple t    => ...,
    case Struct s   => ...,
    case MyStruct m => ...,
}
```

In the case of the variable u declared for the Unit arm, u is equivalent to a unit struct with no fields or members.

### Type Name Patterns
Type Name patterns match strictly on the variant type or tag name without binding an identifier:
```d
switch (...)
{
    case int      => ...,
    case Unit     => ...,
    case Tuple    => ...,
    case Struct   => ...,
    case MyStruct => ...,
}
```

### Exhaustiveness & Redundancy Checking
Switch expressions are statically verified to be exhaustive at compile time using Luc Maranget's matrix reduction algorithm. Missing variants produce a compile-time error specifying the missing pattern(s):
```d
// Error: switch expression is not exhaustive; missing pattern 'EndOfStream'
switch (pkt)
{
    case Data(b)   => "data",
    case Ping(ts)  => "ping",
    case Reset(c)  => "reset",
    case Heartbeat => "heartbeat",
}
```

### Default Arms
Switch expressions may have exactly 1 `default` arm:
```d
switch (pkt)
{
    case Data(bytes) => "data",
    default          => "other packet",
}
```

The default arm represents a catch-all "fallback" for variants that do not have a match arm. Thus, a switch expression may omit arms for any number of variants as long as it has a default arm.

Note that default arms may not declare a variable. The following is invalid:
```d
switch (...)
{
    case int => ...,
    // default val => ..., Error
}
```

### Pattern Guards
Pattern arms may specify a guard condition using `if (condition)`:
```d
switch (pkt)
{
    case Data(bytes) if (bytes.length > 10) => "large data",
    case Data(bytes)                        => "small data",
    default                                 => "other",
}
```

Guarded arms are refutable filters; they do not satisfy exhaustiveness for that variant. A variant with guarded arms must still provide an unguarded fallback pattern or rely on a `default` arm:
```d
//Error: switch expression is not exhaustive
switch (...)
{
    case Data(bytes) if (bytes.length > 10) => ...,
    case Data(bytes) if (bytes.length == 0) => ...,
    case Data(bytes) if (bytes.length == 42) => ...,
}

// Ok
switch (...)
{
    case Data(bytes) if (bytes.length > 10) => ...,
    case Data(bytes) if (bytes.length == 0) => ...,
    case Data(bytes) if (bytes.length == 42) => ...,
    case Data(bytes) => ...,

    // OR
    default => ...,
}
```

Duplicate guarded arms are allowed, but only the first one (in syntactic order) will execute; any duplicate guarded arms are dead code:
```d
switch (...)
{
    case Data(bytes) if (bytes.length > 10) => "greater than 10",
    case Data(bytes) if (bytes.length > 10) => assert(0), // This will never execute
    ...
}
```

## Side-Effects
While switch expressions are expressions, not statements like D's regular switch and final switch constructs, they are also allowed in statement position:
```d
NetworkPacket pkt = ...;

// Ok
switch (pkt)
{
    case Data(bytes)     => writeln("Got Data"),
    case Ping(timestamp) => writeln("Got Ping"),
    case Reset(code)     => writeln("Got Reset"),
    case Heartbeat       => writeln("Got Heartbeat"),
    case EndOfStream     => writeln("Got EndOfStream"),
}
```

However, the compiler analyzes each arm of the switch expression to determine whether they have side-effects or not. If the switch expression is in statement position and none of the arms have side-effects, the compiler will report an error:
```d
// Error: switch expression has no effect; use `cast(void)` to discard its value
switch (pkt)
{
    case Data(bytes)     => format("Data payload (%d bytes)", bytes.length),
    case Ping(timestamp) => format("Ping probe: timestamp=%d", timestamp),
    case Reset(code)     => format("Connection reset with code %d", code),
    case Heartbeat       => "Keep-alive heartbeat received",
    case EndOfStream     => "End of transmission stream",
}

// Ok
cast(void)switch (pkt)
{
    case Data(bytes)     => format("Data payload (%d bytes)", bytes.length),
    case Ping(timestamp) => format("Ping probe: timestamp=%d", timestamp),
    case Reset(code)     => format("Connection reset with code %d", code),
    case Heartbeat       => "Keep-alive heartbeat received",
    case EndOfStream     => "End of transmission stream",
}; // Terminating semicolon required
```

## Metaprogramming

### Traits

New traits will be added for working with enum unions and their variant cases:
- `__traits(allVariants, E)` takes an enum union E and returns a sequence of symbols for each of its variants, in lexical order.
- `__traits(hasVariant, E, Key)` takes an enum union E and a string OR a type Key. If the argument is a string, returns true if E declares a unit, tuple, record, or aliased variant whose identifier equals Key. If it's a type, returns true if E declares a bare type whose canonical base type matches Key.
- `__traits(getVariant, E, Key)` similar to `hasVariant`, except it directly resolves to the variant symbol or canonical type (for bare variants). If Key does not exist in E, it is a compile error.
- `__traits(variantTag, V)` takes a symbol of one of the variants of an enum union, and returns a numeric value representing its tag.
- `__traits(variantParams, V)` takes a symbol of one of the variants of an enum union, and returns the parameter tuple for its constructor. E.g.:
```d
struct ExternalStruct
{
    int n;
}

enum union Vals
{
    case Unit();                        // returns AliasSeq!()
    case Tuple(int, string);            // returns AliasSeq!(int, string)
    case Struct { bool b; double d; };  // returns AliasSeq!(bool, double)
    case int;                           // returns AliasSeq!()
    case MyStruct = ExternalStruct;     // returns AliasSeq!()
}
```
For unit variants, bare type variants, and alias variants, and empty list is returned.
- `__traits(variantParamNames, V)` like `variantParams`, this trait takes a symbol of one of the variants of an enum union, but returns the parameter *names* tuple for its constructor, instead of the types. Unnamed parameters are represented as an empty string.
```d
enum union Vals
{
    case Unit();                          // returns AliasSeq!()
    case Tuple(int n, string);            // returns AliasSeq!("n", "")
    case TupleWithNames(int n, string s); // returns AliasSeq!("n", "s")
    case Struct { bool b; double d; };    // returns AliasSeq!("b", "d")
    case int;                             // returns AliasSeq!()
    case MyStruct = ExternalStruct;       // returns AliasSeq!()
}
```
- `__traits(variantDeclarationOf, V)` this is intended for metaprogramming. It takes a variant symbol V, and transforms it into a variant declaration as it would appear inside an enum union:
```d
enum union A
{
    case int;
    case Unit();
    case Tuple(int n, string);
    case Struct { double d; bool b; };
    case MyStruct = ExternalStruct;
}

enum union B
{
    case Unit();

    static foreach (V; __traits(allVariants, A))
        static if (__traits(identifier, V) == "Unit")
            case __traits(variantDeclarationOf, V, "Unit_A");
        else
            case __traits(variantDeclarationOf, V);
}

Now the variants in B are equivalent to if it was declared as:
enum union B
{
    case Unit();
    case int;
    case Unit_A();
    case Tuple(int n, string);
    case Struct { double d; bool b; };
    case MyStruct = ExternalStruct;
}
```
Variants with identical signatures are automatically merged by `variantDeclarationOf`. Therefore, if the "Unit_A" string argument were omitted (or if an empty string was provided) for the duplicate `case Unit()` from A, it would not be a compile error; the compiler would simply discard one of the duplicates.

This allows powerful metaprogramming patterns like:
```d
/// Tests if two variant declarations match identically across all structural dimensions.
template IsExactMatch(alias V1, alias V2)
{
    // 1. Bare types and Alias variants: compare underlying types via `is(...)`
    static if ((__traits(variantKind, V1) == "bare" && __traits(variantKind, V2) == "bare") ||
               (__traits(variantKind, V1) == "alias" && __traits(variantKind, V2) == "alias"))
    {
        static if (__traits(variantKind, V1) == "alias")
            enum bool IsExactMatch = (__traits(identifier, V1) == __traits(identifier, V2)) && is(V1 == V2);
        else
            enum bool IsExactMatch = is(V1 == V2); // Bare types have no identifier token
    }
    // 2. Tuple, Record, and Unit variants: compare identifier, types, and parameter names
    else static if (__traits(variantKind, V1) == __traits(variantKind, V2))
        enum bool IsExactMatch =
            (__traits(identifier, V1) == __traits(identifier, V2)) &&
            is(__traits(variantParams, V1) == __traits(variantParams, V2)) &&
            (__traits(variantParamNames, V1) == __traits(variantParamNames, V2));
    else
        enum bool IsExactMatch = false;
}

/// Resolves naming: preserves unique/exact variants via "", or namespaces schema conflicts.
template ResolveName(alias V, A, B)
{
    static if (__traits(variantKind, V) == "bare")
        // Bare types never have identifiers or aggregate parents; deduplicated by canonical type
        enum string ResolveName = "";
    else
    {
        enum string id = __traits(identifier, V);
        static if (__traits(hasVariant, A, id) && __traits(hasVariant, B, id))
        {
            alias VA = __traits(getVariant, A, id);
            alias VB = __traits(getVariant, B, id);
            static if (IsExactMatch!(VA, VB))
                // Exact declaration match: pass "" so compiler collapses them
                enum string ResolveName = "";
            // Divergent schemas: prefix using provenance
            else static if (__traits(isSame, __traits(parent, V), A))
                enum string ResolveName = A.stringof ~ "_" ~ id;
            else
                enum string ResolveName = B.stringof ~ "_" ~ id;
        }
        else
            // Unique named variant: retain original name
            enum string ResolveName = "";
    }
}

/// Merges two enum union types: collapses exact matches and disambiguates schema conflicts.
enum union Merge(A, B)
if (is(A == enum union) && is(B == enum union))
{
    static foreach (V; AliasSeq!(__traits(allVariants, A), __traits(allVariants, B)))
        case __traits(variantDeclarationOf, V, ResolveName!(V, A, B));
}

template isInUnion(alias V, Target)
{
    static if (__traits(variantKind, V) == "bare")
        alias Key = V;
    else
        enum string Key = __traits(identifier, V);

    static if (__traits(hasVariant, Target, Key))
        enum bool isInUnion = IsExactMatch!(V, __traits(getVariant, Target, Key));
    else
        enum bool isInUnion = false;
}

/// Yields an enum union containing only variants declared identically in both A and B.
enum union Intersect(A, B)
if (is(A == enum union) && is(B == enum union))
{
    static foreach (V; __traits(allVariants, A))
    {
        static if (isInUnion!(V, B))
            case __traits(variantDeclarationOf, V);
    }
}

/// Yields an enum union containing all variants of A except those shared with B.
enum union Difference(A, B)
if (is(A == enum union) && is(B == enum union))
{
    static foreach (V; __traits(allVariants, A))
    {
        static if (!isInUnion!(V, B))
            case __traits(variantDeclarationOf, V);
    }
}

/// Retains variants unique to either A or B, disambiguating colliding schemas via Merge.
alias SymmetricDifference(A, B) = Merge!(Difference!(A, B), Difference!(B, A));
```

## Breaking Changes and Deprecations

Because `enum union` reuses existing keywords (`enum`, `union`, `case`, `switch`) in a previously illegal syntactic sequence, no user code or symbols are broken. Prefix `switch (...) { ... }` in expression contexts is distinguished from statement switches by the presence of fat-arrow `=>` arms and comma separators.

## Reference

## Copyright & License

Copyright (c) 2026 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://www.google.com/search?q=https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
