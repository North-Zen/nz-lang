# 1. Basic: Null safety, input and output

Part of chapter [1. Basic](01-basic.md#1-basic).

## 1.7 Null safety

`null` is only valid for optional types `T?`.

```nz fragment error="null"
string name = null;
```

A `T?` value must be checked before it is used. The compiler recognises several forms of check:

```nz fragment
string? name = null;

var length = name?.Length;
var shown = name ?? "anonymous";

if (name != null) {
    int n = name.Length;
}

if (name is string s) {
    Console.WriteLine(s);
}
```

Using it without a check is rejected:

```nz fragment error="possibly null"
string? name = null;
int n = name.Length;
```

Operators on an optional number, `char`, `bool` or enum are **lifted**. An arithmetic, bitwise, shift or unary operator gives `null` when an operand is `null`, otherwise the plain operator's result as an optional (`int? + int` is an `int?`, `byte? + byte` an `int?`); the operation runs only when every operand has a value, with the plain operator's promotion and checked arithmetic. `<`, `>`, `<=` and `>=` give a `bool`: `false` when an operand is `null` (so `a <= b` is `false` when both are `null`, while `a == b` is `true`). On `bool?`, `&` and `|` follow a three-valued table: `false & null` is `false`, `true | null` is `true`, and the other combinations with `null` give `null`; `&&` and `||` take only `bool`. Compound assignment and `++`/`--` lift too (`a += 1` leaves a `null` `a` as it is). A `string?` is not lifted, but `+` with a string operand is concatenation: a null operand counts as `""`, so `s + "x"` on a null `s` is `"x"` (a `string`, never null), and `s += "x"` on a null `string?` stores `"x"`. An object is not lifted: check it first.

```nz fragment output="6 [] True False [] False\n"
int? a = 5;
int? none = null;
bool? unknown = null;
a += 1;
Console.WriteLine($"{a} [{a + none}] {a > 5} {none < 5} [{unknown & true}] {unknown & false}");
```

Two spelling rules involve `?`. In `x is int?` the `?` belongs to the type only when the next token cannot start an expression, so `x is int ? a : b` is a conditional, not a test against `int?`. With a space, `x is int? [1] : b` is a conditional whose arm is the collection expression `[1]`. And `?[` (no space) is one token: in a type, `string?[] names` and `new string?[3]` are arrays of optional strings; after a value, `a?[0]` is null-conditional indexing (below), so `x is int?[0]` indexes the `bool` `x is int`, which is an error.

```nz fragment output="yes\n"
int x = 1;
string r = x is int ? "yes" : "no";
Console.WriteLine(r);
```

`a?[i]` reads an element of an array, a `List<T>`, a `string` or a span that may be null: it is `null` when `a` is, and then the index is not evaluated; otherwise it is the element, as an optional (`int?`, `char?`, `string?`). An array reached through `?.` works the same way (`box?.Items?[0]`).

```nz fragment output="none\n2 3\n"
List<int>? l = null;
int? first = l?[0];
Console.WriteLine(first == null ? "none" : "some");
l = [2, 3];
Console.WriteLine($"{l?[0]} {l?[^1]}");
```

The full rules, including when a check stops being valid, are in [Null Safety](+/Null-Safety.md).

## 1.8 Input and output

This section is not written yet.

Outline:

- Writing to the console
- Reading from the console
- Reading and writing files
- Paths and directories

## 1.9 The `using` keyword

**`using` has no cleanup method yet.** `using (var r = ...) { ... }` evaluates its value once, makes it visible in the block and frees it, as an owned local, when the block ends; there is no interface for a user-defined cleanup method. Use `defer` inside the block for other cleanup.

Previous: [1.5 Operators](01-basic-operators-control-flow.md#15-operators) · [Contents](README.md) · Next: [2. Programming paradigms](02-programming-paradigms.md#2-programming-paradigms)
