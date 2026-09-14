# Nominal Sum Types via `enum union` and `switch` Expressions

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            |                                                                 |
| Review Count:   |                                                                 |
| Authors:        | Jared Hanson                                                    |
| Implementation: | https://github.com/dlang/dmd/pull/23744                         |
| Status:         |                                                                 |

## Abstract

Add nominal `enum union` declarations and `switch` expressions to the D programming language, providing algebraic data types (discriminated unions) with unboxed layout, bounded polymorphism, and static exhaustiveness checking. An `enum union` lowers to an aggregate struct containing an anonymous union payload and a discriminant tag. Switch expressions lower to ternary expressions and are used to pattern match against these unions at run time to enable powerful programming patterns.

## Contents

* [Rationale](https://www.google.com/search?q=%23rationale)
* [Prior Work](https://www.google.com/search?q=%23prior-work)
* [Description](https://www.google.com/search?q=%23description)
* [Breaking Changes and Deprecations](https://www.google.com/search?q=%23breaking-changes-and-deprecations)
* [Reference](https://www.google.com/search?q=%23reference)
* [Copyright & License](https://www.google.com/search?q=%23copyright--license)
* [History](https://www.google.com/search?q=%23history)

## Rationale

Sum types (also known as tagged unions, discriminated unions, or algebraic data types) are a foundational construct in type-safe programming. They allow expressing that a value is one of several distinct variants, with static guarantees that all cases are handled. This eliminates invalid state representations and missing branch errors at compile time.

While D supports raw unions, they are untagged, inherently `@system` to access, and lack compiler-managed tag coordination, destructor synthesis, and branch exhaustiveness. Library solutions like `std.sumtype` implement tagged unions via complex template metaprogramming. However, library implementations suffer from slow compilation throughput, opaque diagnostic errors, and the inability to exploit compiler memory layout optimizations such as niche optimization.

Prior Work:

- Rust enums, the main inspiration for this feature.
- Swift enums, also a large source of inspiration, and its switch expressions.
- C# unions and switch expressions.
- Java switch expressions.
- Odin unions.
- Zig's union(enum).
- D's `std.sumtype` and `std.variant`.
- [Richard (Rikki) Cattermole's DIP for type unions and match expressions](https://forum.dlang.org/post/nhbiwarfrlqqffegkhsf@forum.dlang.org).

## Description

This DIP proposes 2 new constructs for the D language: `enum union` as a language-level discriminated union type, and switch expressions which are used to inspect these unions at runtime.

An enum union is declared using the `enum union` keyword:
```d
enum union NetworkPacket
{
    case Data(const(ubyte)[]),
    case Ping(ulong),
    case Reset(ushort),
    case Heartbeat(),
    case EndOfStream(),
}
```

Variant declarations in an enum union must start with the keyword `case`; they represent one of the possible values that an enum union may take on. There are different types of variants that serve different functions.

## Variant Kinds

### Tuple Variants
Tuple-like variants (or "tuple variants" for short) consist of a name and a list of parameters required to construct that variant. Each parameter may be named, but it is not required. It is allowed to mix named and unnamed (positional) parameters when declaring a tuple variant:
```d
enum union DrawCommand
{
    // Unnamed positional parameters
    case MoveTo(double, double),
    case LineTo(double, double),

    // Named positional parameters
    case Circle(double x, double y, double radius),
    case Text(string content, double x, double y, ubyte fontSize),

    // Mixed named and unnamed positional parameters
    case Arc(double, double, double radius, double sweepAngle),
}
```

If a tuple-like variant has 0 parameters (like `Heartbeat` and `EndOfStream` in the previous example above), it is declared with an empty argument as shown. These are referred to as unit variants, and they are equivalent to other unit types in D like `void` and `typeof(null)`.

Tuple-like and unit variants **do not** have their own type. They are the same type as the containing enum union.

### Struct Variants
Struct variants are declared as a normal struct declaration:
```d
enum union PaymentEvent
{
    case CardCharge {
        string token;
        ulong amountCents;
        string currency;
        bool require3DSecure;
    },

    case BankTransfer {
        string iban;
        string bic;
        ulong amountCents;
        string reference;
    },

    case RefundIssued {
        ulong originalTxId;
        ulong refundAmountCents;
        string reason;
    },
}
```

Note that struct variants are only allowed to declare fields; not methods, constructors, destructors, or any other type of declaration.

Unlike tuple variants, struct variants are type declarations. They're also subtypes of the enum union:
```d
auto charge = PaymentEvent.CardCharge("my token", 1_000_000_000, "CAD", true);
assert(is(typeof(charge): PaymentEvent));
```

### Bare Type Variants
Bare-type variants directly embed an external type as a case in the enum union without needing to wrap it in a tuple variant:
```d
enum union ConfigValue
{
    bool,
    long,
    double,
    string,
    string[],
}
```

They are useful for defining a type which may be a value of one of several different subtypes. Since they do not have names, bare type variants are initialized via direct assignment, similar to struct assignment constructor syntax:
```d
ConfigValue c = false; // Union holds a value of type bool
c = ["some", "cool", "strings"]; // Union now holds a value of type string[]
c = [1, 2, 3]; // Error, no variant `int[]` in enum union `ConfigValue`
```

When it is ambiguous which type would be initialized by this assignment, the compiler requires the user to disambiguate:
```d
enum union Nums 
{
    case int,
    case long,
}

Nums n = 0; // Error, 0 is ambiguous between variants `int` and `long` of enum union `Nums`
Nums n = 0L; // Ok
```

Bare type variants can only be _accessed_ via switch expressions, which will be discussed later in this DIP.

### Alias Variants
Alias variants are a shorthand syntax that allows aliasing an external type to a different name while also declaring it as a variant in the enum union. This is useful for embedding types that are in different modules, but have conflicting names, or for renaming an embedded external type:

```d
enum union KeyInput
{
    case Windows = sys.platform.win32.events.Win32KeyEvent,
    case Wayland = sys.platform.linux.wayland.WaylandKeyEvent,
    case Darwin  = sys.platform.darwin.cocoa.CocoaKeyEvent,
}

auto input = KeyInput.Windows(32, true);
```

This is semantically equivalent to:
```d
enum union KeyInput
{
    alias Windows = sys.platform.win32.events.Win32KeyEvent;
    case Windows,

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
    case IngestQueue  = RingBuffer!string,
    case PacketStream = RingBuffer!(ubyte[]),
    case MetricResult = Future!double,
}

auto task = WorkerTask.PacketStream("eth0", [[0xAA, 0xBB], [0xCC]]);
```

## Enum Union Members

Enum unions are treated as struct declarations internally, which contain a union with the declared variant cases, and a `__tag` value to track which variant is currently active.

Like other aggregates in D, enum unions can contain members, member functions, constructors, destructors, aliases, etc.
```d
enum union NetworkMessage
{
    case Heartbeat(),
    case Text(string content, string encoding),
    case Binary(ubyte[]),
    case Status(int statusCode, string statusText); // Terminating semicolon delimits variants

    ulong timestamp;
    uint sequenceNumber;

    this(string rawInput, uint seq = 0, ulong ts = 0)
    {
        if (rawInput == "ping")
            this = Heartbeat;
        else if (rawInput.length == 0)
            this = Status(400, "Empty Payload");
        else
            this = Text(rawInput);

        this.sequenceNumber = seq;
        this.timestamp = ts;
    }

    size_t byteLength()
    {
        return switch (this)
        {
            case Heartbeat            => 0,
            case Text(content, ...)   => content.length,
            case Binary(bytes)        => bytes.length,
            case Status(status, text) => status.sizeof + text.length,
        };
    }

    // Can use shorthand method syntax too
    bool isControlFrame() => switch (this)
    {
        case Heartbeat => true,
        default        => false,
    };
}
```

Inside constructors and member functions, `this` refers to the union itself, not the currently active variant. Member fields may be accessed with `this.<field>`, but not the fields of individual variants.

Inside constructors, the compiler uses definite assignment analysis to ensure that the union has been properly initialized on all code paths.

## .init and Default Construction

Every enum union provides a `.init` value, which by default is the `.init` value of its first declared variant (in syntactic order). If that variant has an `@disable`'d init value, then the `.init` value of the second variant will be used. If all variants disable `.init`, the enum union will also have a disabled `.init`.
```d
enum union Option(T)
{
    case None,
    case Some(T),
}

Option!int opt;
assert(opt.__tag == 0);
assert(opt == Option!int.None);
```

* If all types in the enum union disable default construction (`@disable this();`), default construction will be disabled for the union as well.

## In-Memory Layout

An enum union's layout is equivalent to the layout of a struct defined as follows:
```d
struct EnumUnion
{
    ubyte __tag;

    struct __UnitStruct {}
    struct 

    union {
        UnitStruct _0;
    }
}

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
}
```

Any member fields declared come after the union payload.

## Memory Safety

The enum union guarantees memory integrity across variant transformations:

* **Value-Copy Pattern Bindings**: Pattern match bindings extract payloads by value into the arm's lexical scope. This isolates bound variables from the parent aggregate, preventing aliasing hazards where an active reference could be corrupted by a concurrent re-tagging or reassignment of the parent union during arm evaluation. Once D has a sound method of tracking ownership and borrowing, like Rikki's DFA analyzer, then binding by ref can be allowed.

* **RAII Lifecycle Dispatch**: If any variant contains an elaborate destructor (`~this()`), the compiler synthesizes an aggregate destructor that inspects the discriminant tag and invokes the destructor of the active variant.

* **Deterministic Re-Tagging**: Reassigning a sum type from variant `A` to variant `B` executes active destruction of `A`, writes the new discriminant tag, and blits payload `B` in an exception-safe sequence, eliminating use-after-free, memory leaks, and type confusion.

* **Rejection of Non-Copyable Payloads**: Because pattern extraction relies on value-copy isolation to remain `@safe` without a borrow checker, and because DMD currently lacks definite assignment analysis and sub-field dynamic drop flags to safely relocate unboxed union members without risking double-destruction on scope exit, move-only types are rejected at declaration time.


## Implicit Construction

Enum unions are implicitly constructed in the following cases: the struct-style construction via assignment discussed previously, when a function takes an enum union as an argument, and when a function returns an enum union:
```d
enum union Option(T)
{
    case None = typeof(null),
    case Some(T),
}

Option!ConfigValue getConfigVal(string name) {
    string[string] config = readConfig("config.csv");
    if (auto val = name in config) with (typeof(return)) {
        return Some(ConfigValue(*val)); // Implicitly constructs an Option!ConfigValue
    }

    return null;
}

void applyConfigVal(Option!ConfigValue c);
applyConfigVal(null); // Implicitly constructs Option!ConfigValue.None
```

**NOTE:** Only one level of implicit construction takes place. In the example above, the ConfigValue passed to Some is not able to be implicitly constructed.

## Niche Optimization (not yet implemented)

When an enum union contains unit variants alongside non-nullable references, pointers (`T*`), class references, or bounded scalars (such as `bool`), the compiler exploits invalid bit patterns to encode the unit state:

* `Option!(int*)`: The null pointer address `0x0` represents `None`.
* `sizeof(Option!(int*)) == 8` (on 64-bit platforms), incurring zero byte overhead for the tag.

## Switch Expressions

Switch expressions are the main way to interact with enum unions. They use pattern matching to match the possible variants:
```d
enum union NetworkPacket
{
    case Data(const(ubyte)[]),
    case Ping(ulong),
    case Reset(ushort),
    case Heartbeat(),
    case EndOfStream(),
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

There may only be **one** pattern per variant. The following will not compile:
```d
switch (pkt)
{
    case Data(bytes) => ...,
    case Data(bytes2) => ..., // Error: redundant match arm. Pattern is unreachable
}
```

Every arm must start with the `case` keyword, and every arm is required to produce a value. All arms must unify to the same type via a LUB (Least Upper Bound) calculation. Arms may not contain statements; only a single expression that produces the value for that arm. Thus, the following will not compile:
```d
    case Data(bytes) => {
        writeln("Received Data payload");
        ...
        return format(...);
    }
```

However, statement blocks can be emulated using an immediately-called delegate literal:
```d
    case Data(bytes) => {
        writeln(...);
        ...etc.
        return format(...);
    }(),
```
**NOTE:** when using switch expressions with an enum union, the union's variants are automatically inserted into the switch expression's scope for convenient access.

There are multiple forms of patterns for matching against values in different ways.

## Switch Expression Patterns

### Destructuring Patterns
As shown above, destructuring patterns destructure the enum union's variants. Destructuring patterns can be used for unit, tuple, and structure variants.

Destructuring patterns allow fields to be omitted using `...` syntax:
```d
enum union U
{
    case Unit(),
    case Struct { int n; double d; string s; }
}

switch (U.Struct(42, 6.9, "asdf"))
{
    case Unit() => 1,
    case Struct(n, ...) => n * 2, // Ignores fields d and s
}
```

The `...` syntax allows ALL fields to be omitted:
```d
    case Unit(...) => 1, // This is valid because ... means "0 or more fields"
    case Struct(...) => "No access to Struct's fields here",
```

The `...` syntax can also be prefixed with a variable name:
```d
    // case Unit(rest...) => 1, This is also valid, rest = AliasSeq!()
    case Struct(n, rest...) => typeof(rest).stringof, // AliasSeq!(double, string)
```

This syntax transforms the remaining fields in the struct into an AliasSeq, similar to `T...` template syntax. This syntax is also supported for tuple and unit variants.

### Variable Patterns
Variable patterns are of the form `case Type name =>`. Their syntax mirrors the declaration of a local variable. These patterns can be used for any type of variant:
```d
enum union A
{
    case int,
    case Unit(),
    case Tuple(int n, string),
    case Struct { double d; bool b; },
    case MyStruct = ExternalStruct,
}

switch (A.Tuple(42, "asdf"))
{
    case int n => ...,
    case Unit u => ...,
    case Tuple t => ...,
    case Struct s => ...,
    case MyStruct m => ...,
}
```

In the case of the variable `u` declared for the `Unit` arm, `u` is equivalent to a unit struct with no fields or members.

### Type Name Patterns
Type Name patterns are the simplest form of pattern. They are of the form `case Type =>`, with no identifier. They are also supported for any type of variant:
```d
switch (...)
{
    case int => ...,
    case Unit => ...,
    case Tuple => ...,
    case Struct => ...,
    case MyStruct => ...,
}
```

### Exhaustiveness
Switch expressions are required to be exhaustive over the variants in the enum union. The following will fail to compile with an error listing the variants not covered:
```d
// Error: switch expression is not exhaustive; missing patterns ...
switch (...)
{
    case Unit => "unit"
}
```

### Default Arms
Switch expressions may have exactly 1 default arm:
```d
switch (...)
{
    case int => ..., // Only want to explicitly handle the int case
    default => ...,  // Cover all other cases
}
```

The default arm represents a catch-all "fallback" for variants that do not have a match arm. Thus, a switch expression may omit arms for any number of variants as long as it has a default arm.

Note that default arms may not access the active variant. The following is invalid:
```d
switch (...)
{
    case int => ...,
    // default val => ..., Error
}
```

### Pattern Guards
Switch arms may have a **Pattern Guard** which is declared with the following syntax:
```d
switch (...)
{
    case Data(bytes) if (bytes.length > 10) => ...,
    case Data(bytes) => ...,
    ...etc.
}
```

The expression inside the `if (...)` must evaluate to a bool, and the arm will only be taken if it evaluates to true (otherwise, it's skipped).

Guarded arms **do not** contribute to exhaustiveness; thus, while there **must** be exactly 1 unguarded arm for each variant, and there may be any number of guarded arms, the switch expression is considered inexhaustive if there are only guarded arms for a given variant (unless it has a default arm):
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
}; // Ending semicolon required
```

## Metaprogramming

### Traits

New traits will be added for working with enum unions and their variant cases:
- `__traits(allVariants, E)` takes an enum union E and returns a sequence of symbols for each of its variants, in lexical order.
- `__traits(hasVariant, E, Key)` takes an enum union E and a string OR a type Key. If the argument is a string, returns true if E declares a unit, tuple, record, or aliased variant whose identifier equals Key. If it's a type, returns true if E declares a bare type whose canonical base type matches Key.
- `__traits(getVariant, E, Key)` similar to `hasVariant`, except it directly resolves to the variant symbol or canonical type (for bare variants). If Key does not exist in E, it is a compile error.
- `__traits(variantTag, V)` takes a symbol of one of the variants of an enum union, and returns a numeric value representing its tag.
- `traits(variantParams, V)` takes a symbol of one of the variants of an enum union, and returns the parameter tuple for its constructor. E.g.:
```d
struct ExternalStruct
{
    int n;
}

enum union Vals
{
    case Unit(),                        // returns AliasSeq!()
    case Tuple(int, string),            // returns AliasSeq!(int, string)
    case Struct { bool b; double d; },  // returns AliasSeq!(bool, double)
    case int,                           // returns AliasSeq!()
    case MyStruct = ExternalStruct,     // returns AliasSeq!()
}
```
For unit variants, bare type variants, and alias variants, and empty list is returned.
- `traits(variantParamNames, V)` like `variantParams`, this trait takes a symbol of one of the variants of an enum union, but returns the parameter *names* tuple for its constructor, instead of the types. Unnamed parameters are represented as an empty string.
```d
enum union Vals
{
    case Unit(),                          // returns AliasSeq!()
    case Tuple(int n, string),            // returns AliasSeq!("n", "")
    case TupleWithNames(int n, string s), // returns AliasSeq!("n", "s")
    case Struct { bool b; double d; },    // returns AliasSeq!("b", "d")
    case int,                             // returns AliasSeq!()
    case MyStruct = ExternalStruct,       // returns AliasSeq!()
}
```
- `traits(variantDeclarationOf, V)` this is intended for metaprogramming. It takes a variant symbol V, and transforms it into a variant declaration as it would appear inside an enum union:
```d
enum union A
{
    case int,
    case Unit(),
    case Tuple(int n, string),
    case Struct { double d; bool b; },
    case MyStruct = ExternalStruct,
}

enum union B
{
    case Unit(),

    static foreach (V; __traits(allVariants, A))
        static if (__traits(identifier, V) == "Unit")
            case __traits(variantDeclarationOf, V, "Unit_A");
        else
            case __traits(variantDeclarationOf, V);
}

Now the variants in B are equivalent to if it was declared as:
enum union B
{
    case Unit(),
    case int,
    case Unit_A(),
    case Tuple(int n, string),
    case Struct { double d; bool b; },
    case MyStruct = ExternalStruct,
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

template IsInUnion(alias V, Target)
{
    static if (__traits(variantKind, V) == "bare")
        alias Key = V;
    else
        enum string Key = __traits(identifier, V);

    static if (__traits(hasVariant, Target, Key))
        enum bool IsInUnion = IsExactMatch!(V, __traits(getVariant, Target, Key));
    else
        enum bool IsInUnion = false;
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

Copyright (c) 2026 by the D Language Foundation[cite: 1]

Licensed under [Creative Commons Zero 1.0](https://www.google.com/search?q=https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
