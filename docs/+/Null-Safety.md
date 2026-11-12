# Null Safety

The main claim of NZ: **a non-optional type never holds `null`, and a `T?` cannot be used without a check.** This page details how the compiler enforces it.

## Basic rules

- `null` can only be assigned to a `T?`.
- A non-optional variable or field must have an initial value (numbers, `bool` and `char` default to zero).
- The elements of `new T[n]` must have a default value, so `new string[3]` is rejected; `new string?[3]` is accepted.

```nz fragment error="default"
var a = new string[3];
```

```nz fragment
var a = new string?[3];
var b = new int[3];
```

## What makes a `T?` safe to use (narrowing)

The compiler tracks, syntactically, the local variables and parameters (except `ref` parameters) that are certainly not `null` at a given point.

| Form | Safe in |
|---|---|
| `if (x != null)` | the `then` branch |
| `if (x == null) ... else` | the `else` branch |
| `if (x is T v)` | the `then` branch, and `v` has a non-optional type |
| `x != null && ...` | the right operand of `&&` |
| `x == null \|\| ...` | the right operand of `\|\|` |
| `x != null ? ... : ...` | the first arm of the ternary |
| `while (x != null)` | the loop body |
| `x ??= v;` as a single statement inside a block | the rest of that block, for local variables only |

`?.` and `??` are always allowed on an optional value.

`?.` needs a value on its left. A type name is not a value, so `Counter?.Hit()` is an error, as in C# (write `Counter.Hit()`):

```nz error="`?.` cannot be used on the type `Counter`"
package main;

class Counter {
    public static void Hit() { }
}

void Main(string[] args) {
    Counter?.Hit();
}
```

```nz fragment
string? s = null;
bool ok = s != null && s.Length > 0;
bool empty = s == null || s.Length == 0;
var n = s != null ? s.Length : 0;
```

## When a check stops applying

The compiler withdraws the "certainly not null" status when the value may change.

**Assignment**, including inside a block that was already checked:

```nz fragment error="possibly null"
string? s = null;
if (s != null) {
    s = null;
    int n = s.Length;
}
```

**A loop** that writes the variable, because the next iteration reads it again:

```nz fragment error="possibly null"
string? s = null;
if (s != null) {
    for (int i = 0; i < 3; i++) {
        int n = s.Length;
        s = null;
    }
}
```

**A `ref` argument**, because the called function may write it:

```nz error="possibly null"
package main;

void Clear(ref string? p) {
    p = null;
}

void Main(string[] args) {
    string? s = null;
    if (s != null) {
        Clear(s);
        int n = s.Length;
    }
}
```

**`defer`**: a deferred statement runs when the block that contains it ends, so it does not inherit checks from outside. A name written by a block's `defer` is no longer considered checked after that block ends, even if `??=` narrows it again inside the block. Inside the block, after the `defer` statement, the name is still considered checked because the deferred code has not run yet.

```nz fragment error="possibly null"
int? a = args.Length + 1;
if (a != null) {
    {
        defer { a = null; }
    }
    int b = a;
}
```

```nz fragment error="possibly null"
int? a = args.Length + 1;
if (a != null) {
    {
        defer { a = null; }
        a ??= 7;
    }
    int b = a;
}
```

```nz fragment output="1\n"
int? a = args.Length + 1;
if (a != null) {
    defer { a = null; }
    int b = a;
    Console.WriteLine($"{b}");
}
```

A `defer` cannot directly contain another `defer`; write `defer { defer X; }`:

```nz fragment error="cannot be the direct body of `defer`"
int x = 0;
defer defer x = 1;
```

**A condition that writes the checked variable**, for example `s != null && (s = null) == null`, proves nothing.

## `??=` as a check

`x ??= v` assigns `x` only when `x` is `null`, and `v` is evaluated only then. As a single statement directly inside a block, `??=` on a local variable makes that variable certainly not null until the block ends.

```nz fragment output="5\n"
int? a = null;
a ??= 5;
int b = a;
Console.WriteLine($"{b}");
```

`??=` does not narrow when it is inside a branch, a loop, an inner block or an expression (`if (c) { a ??= 5; }`, `(a ??= 5)`, `c && (a ??= 5) > 0`), or when its target is a field, an element or a `ref` parameter:

```nz fragment error="possibly null"
int? a = args.Length;
if (args.Length == 0) {
    a ??= 5;
}
int b = a;
```

```nz fragment error="possibly null"
int? a = null;
{
    a ??= 5;
}
int b = a;
```

```nz fragment error="possibly null"
int? a = null;
bool ok = args.Length == 0 && (a ??= 5) > 0;
int b = a;
```

The value of `??=` itself has type `T` when `v` has type `T`, so `int d = (c ??= 7) + 1;` is accepted.

## Lifted operators

As in C#, operators on `T?` where `T` is a number, `char`, `bool` or an enum are **lifted**. An arithmetic, bitwise, shift or unary operator gives `null` when an operand is `null`, and otherwise the plain operator's result as an optional (`int? + int` is an `int?`). The operation on the values runs only when every operand has a value, with the same promotion and checked arithmetic. `<`, `>`, `<=` and `>=` give a `bool`: `false` when an operand is `null`. On `bool?`, `&` and `|` follow C#'s three-valued table (`false & null` is `false`, `true | null` is `true`); `&&` and `||` take only `bool`. Compound assignment and `++`/`--` are lifted too. The result stays optional, so it must be checked before it is used as a `T`:

```nz fragment output="11 [] False\n"
int? a = 10;
int? b = null;
a++;
bool small = b < 5;
Console.WriteLine($"{a} [{a * b}] {small}");
```

```nz fragment error="possibly null"
int? a = 10;
int b = a + 1;
```

A `string?` is not lifted, but `+` with a string operand is concatenation, as in C#: a `null` operand counts as `""`. So `s + "x"` on a null `s` is `"x"` (a `string`, never `null`), and `s += "x"` on a null `string?` stores `"x"`. Other uses of a `string?` (`-`, `.Length`) still need a check. Objects and structs are not lifted, and an optional one cannot be concatenated yet.

```nz fragment output="[x] [x] [ab]\n"
string? s = null;
string? t = null;
t += "x";
string? u = "a";
Console.WriteLine($"[{s + "x"}] [{t}] [{u + "b" + s}]");
```

## Current limits

- **Fields are never narrowed**, only local variables and parameters. Copy to a local variable first.
- **`ref` parameters are never narrowed either**, because they can alias a field or another `ref` argument that is written in the middle of the block. Use `?.`, `??`, or copy to a local variable first.

```nz error="possibly null"
package main;

void LengthOf(ref string? p) {
    if (p != null) {
        int n = p.Length;
    }
}

void Main(string[] args) {
}
```

```nz
package main;

int LengthOf(ref string? p) {
    string? copy = p;
    if (copy != null) {
        return copy.Length;
    }
    return p?.Length ?? 0;
}

void Main(string[] args) {
}
```
- **An early return is not recognized yet**: `if (x == null) { return; } x.Length` is not accepted yet.
