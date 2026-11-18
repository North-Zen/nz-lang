# 1. Basic: Operators and control flow

Part of chapter [1. Basic](01-basic.md#1-basic).

## 1.5 Operators

```nz fragment
int a = 7;
int b = 2;
var sum = a + b;
bool greater = a > b && b != 0;
int bits = a & b;
var answer = greater ? "yes" : "no";
a++;
```

Besides the usual arithmetic, comparison, logical and bitwise operators, NZ supports `>>>` (a logical shift right), unary `+`, `??` (coalesce), `??=`, `?.` (optional chaining) and `is`.

`>>` keeps the sign and `>>>` does not. Both take an `int` or a `long` on the left and an `int` count on the right; the count is masked to the width of the type, so a shift never fails at run time (`1 >>> 33` is `1 >>> 1`). Unary `+` leaves a number as it is.

```nz fragment output="15 15 0 -4 2147483644\n-3 -3 -1\n"
Console.WriteLine($"{-16 >>> 28} {-1L >>> 60} {1 >>> 33} {-8 >> 1} {-8 >>> 1}");
int x = -3;
Console.WriteLine($"{+x} {+ +x} {1 - +2}");
```

**Compound assignment.** `+= -= *= /= %=` work as before. `&= |= ^=` take an integer target and a value that widens to it, or two `bool`s; `<<= >>= >>>=` take an integer target and an `int` count. `x op= y` means `x = x op y`, but the target and its index are evaluated once. On `bool`, both sides are always evaluated (no short circuit), like `&`, `|` and `^`. The value must convert to the target's type implicitly (a constant that fits counts, so `b += 1` works on a `byte`); the operation runs in the promoted type and the result is converted back, checked: `byte b = 250; b += 10;` stops the program with ``nz: value does not fit in `byte` ``. `++` and `--` work the same way on every number type.

**`unchecked` and `checked`.** `unchecked(expr)` and `unchecked { ... }` make the integer `+ - *`, negation, compound assignments, `++`/`--` and integer casts written inside wrap around or keep the low bits instead of stopping the program; `checked(...)` and `checked { ... }` restore the default. Four things stay checked inside `unchecked`: `/` and `%` (a zero divisor, and the smallest value divided by -1), a cast from a real to an integer, and a cast from an integer to `char` (a `char` must be a Unicode scalar value). A function called inside keeps its own checking. Inside `unchecked` a constant folds the same way, so `unchecked((byte)300)` is 44. Both words are keywords: nothing can be named `checked` or `unchecked`.

```nz fragment output="4 -2147483648 255\n"
int big = 2147483647;
byte b = 0;
unchecked { b--; }
Console.WriteLine($"{unchecked((byte)260)} {unchecked(big + 1)} {b}");
```

On a small type, a compound assignment or `++`/`--` computes in the promoted type and checks the narrowing back, in the target (a variable, a field or an array element):

```nz fragment output="20 20 22\n8\n"
byte b = 1;
b += 1;
b++;
--b;
b |= 8;
b <<= 2;
b >>= 1;
Console.WriteLine($"{b} {b++} {++b}");
byte[] bs = new byte[2];
bs[1] += 7;
bs[1]++;
Console.WriteLine($"{bs[1]}");
```

```nz fragment error="cannot apply compound assignment to `byte` and `int`"
byte b = 1;
b += 300;
```

A `char` is not an integer, so `c += 1` is an error: write `c += (char)1` or `c++`.

**`??=`.** `x ??= v` sets `x` to `v` only when `x` is `null`. The target is a variable, a field or an element of type `T?`. The value is evaluated only when the target is `null`, and after the target (and its index) has been evaluated. The value must convert to `T` (the result has type `T`; an integer constant that fits counts, so `byte? b; b ??= 0;` works) or to `T?` (the result has type `T?`, so `x ??= null` is legal and changes nothing).

```nz fragment output="10 1\n5 8 7\n"
int? a = null;
int calls = 0;
a ??= (calls += 1) + 9;
a ??= (calls += 1) + 9;
Console.WriteLine($"{a} {calls}");

int? b = null;
b ??= 5;
int c = b;
int? d = null;
int e = (d ??= 7) + 1;
Console.WriteLine($"{c} {e} {d}");
```

With a small integer type the constant rule of 1.1.1 applies to the value: `byte? b = null; b ??= 0;` works, and `b ??= 300` is an error.

```nz fragment output="0 5 65535\nTrue 1 7 False\n"
byte? o = null;
o ??= 0;
byte? p = 5;
p ??= 9;
ushort? w = null;
w ??= 65535;
Console.WriteLine($"{o} {p} {w}");
byte? a = 1;
byte? n = null;
bool e = a == 1;
var s = a ?? 7;
var t = n ?? 7;
bool z = a == n;
Console.WriteLine($"{e} {s} {t} {z}");
```

```nz fragment error="cannot assign `int` to `byte?`"
byte? q = null;
q ??= 300;
```

A target that is not optional is an error:

```nz fragment error="the left side of `??=` must be optional, found `int`"
int x = 1;
x ??= 2;
```

**When `??=` counts as a null check.** Only the statement form narrows: a `??=` that is a whole statement directly inside a block makes a local variable non-null for the rest of that block (see [Null Safety](+/Null-Safety.md)). It does not narrow a field, an element or a `ref` parameter, and it does not narrow inside an expression (an initializer, a `&&` operand, an interpolation hole).

```nz fragment error="possibly null"
int? a = args.Length;
if (args.Length == 0) {
    a ??= 5;
}
int b = a;
```

When the value of `??=` is not used it is dropped, and so is an object the value stored into the target itself; nothing is leaked. `a ??= b ??= 4` assigns from the right.

The order of precedence, from lowest:

<!-- GEN:precedence -->
| Precedence | Operators | Associativity |
|---|---|---|
| lowest | `=` `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` `>>>=` `??=` | right |
| | `? :` | right |
| | `??` | right |
| | `\|\|` | left |
| | `&&` | left |
| | `\|` | left |
| | `^` | left |
| | `&` | left |
| | `==` `!=` | left |
| | `<` `>` `<=` `>=` | left |
| | `<<` `>>` `>>>` | left |
| | `+` `-` | left |
| | `*` `/` `%` | left |
| | `switch` (expression) | left |
| | `..` (range) | none |
| | `!` `+` `-` `~` `^` `++` `--` `(T)x` (prefix) | right |
| highest | `.` `?.` `()` `[]` `++` `--` `?` (postfix) | left |
<!-- /GEN -->

`is` binds at the same level as the relational operators.

In the grammar of `switch`, the operand of a `switch` expression is a range expression: a prefix operator, a cast, a range or a postfix operator binds tighter than `switch`, and a binary operator looser. So `-n switch { ... }` is `(-n) switch { ... }`, `(int)d switch { ... }` is `((int)d) switch { ... }`, `a..b switch { ... }` is `(a..b) switch { ... }`, and `1 + n switch { ... }` is `1 + (n switch { ... })`:

```nz fragment output="20 2 7 11 -10\n"
int n = 3;
double d = 2.9;
int a = -n switch { 3 => 10, -3 => 20, _ => 30 };
int b = ~n switch { 3 => 1, _ => 2 };
int c = (int)d switch { 2 => 7, _ => 8 };
int e = 1 + n switch { 3 => 10, _ => 0 };
int f = -(n switch { 3 => 10, _ => 0 });
Console.WriteLine($"{a} {b} {c} {e} {f}");
```

### The `sizeof` operator

`sizeof(T)` is the size in bytes of a built-in numeric type, `bool`, `char` or an enum, as an `int` constant (`bool` 1, `char` 4; an enum has the size of its underlying type, so 4 for a plain enum and 1 for `enum B : byte`).

```nz fragment output="1 2 4 8 1 4\n"
Console.WriteLine($"{sizeof(byte)} {sizeof(short)} {sizeof(int)} {sizeof(long)} {sizeof(bool)} {sizeof(char)}");
```

## 1.6 Control flow

```nz fragment output="big\ni = 0\ni = 1\ni = 2\n1\n"
int n = 3;

if (n > 2) {
    Console.WriteLine("big");
} else {
    Console.WriteLine("small");
}

while (n > 0) {
    n -= 1;
}

for (int i = 0; i < 3; i++) {
    Console.WriteLine($"i = {i}");
}

var items = new List<int>();
items.Add(1);
foreach (var x in items) {
    Console.WriteLine($"{x}");
}
```

### 1.6.1 Conditionals

`switch` is used as an expression, with `_` meaning "anything else":

```nz fragment
int n = 2;
var label = n switch { 1 => "one", 2 => "two", _ => "other" };
```

A `switch` takes a scrutinee of any of the ten integer types. A pattern is an integer constant written in the scrutinee's type, and one outside the type's range is an error:

```nz fragment output="b200 sb-5 us u ul l n\n"
byte b = 200; sbyte sb = -5; ushort us = 60000; uint u = 3000000000; ulong ul = 10000000000; long l = -3; nint n = 7;
string r1 = b switch { 200 => "b200", _ => "?" };
string r2 = sb switch { -5 => "sb-5", _ => "?" };
string r3 = us switch { 60000 => "us", _ => "?" };
string r4 = u switch { 3000000000 => "u", _ => "?" };
string r5 = ul switch { 10000000000 => "ul", _ => "?" };
string r6 = l switch { -3 => "l", _ => "?" };
string r7 = n switch { 7 => "n", _ => "?" };
Console.WriteLine($"{r1} {r2} {r3} {r4} {r5} {r6} {r7}");
```

```nz fragment error="is outside the range of `byte`"
byte b = 5;
string r = b switch { 300 => "x", _ => "?" };
```

On a union, a `switch` arm names a case type (`Cat c => ...`), and `switch` also has a statement form with `case` and `default` sections; both are in section [1.6.1](#161-conditionals).

#### Patterns

**Type patterns.** On a union, a `switch` arm or an `is` test names a case: `Cat c` binds the case to a new name, `Cat` alone only tests it, and `_` takes anything else. A binding borrows the case: it can be read and changed, not moved out. A type that is not a case of the union is an error (`` `Fish` is not a case of `Pet` ``). The union itself is a type test of the value: on a `Pet?`, `o is Pet p` tests for null and binds the union (`n.Left is Tree l` in a recursive union), and on a `Pet` it always matches. On any other type, a type pattern is the tested type itself (`int i` on an `int?` tests for null); a type pattern between classes works with `is` (see Inheritance) and waits in `switch`.

**Constant, relational and logical patterns**. A constant (`0`, `'a'`, `"hi"`, `null`, a named constant or an enum member) matches an equal value; `< c`, `<= c`, `> c` and `>= c` compare numbers, `char`s and enum values with a constant; `not`, `and` and `or` combine patterns, in that order of precedence, and parentheses group them. `is` takes every pattern, and `x is not null` proves `x` is not null where it holds:

```nz output="name digit space other\nTrue False 3 -1\n"
package main;

import std.io.Console;

string Kind(char c) => c switch {
    >= 'a' and <= 'z' or >= 'A' and <= 'Z' or '_' => "name",
    >= '0' and <= '9' => "digit",
    ' ' or '\t' or '\n' => "space",
    _ => "other",
};

int Length(string? s) {
    if (s is not null) {
        return s.Length;
    }
    return -1;
}

void Main(string[] args) {
    Console.WriteLine($"{Kind('x')} {Kind('7')} {Kind(' ')} {Kind('+')}");
    char c = 'q';
    int? n = 3;
    Console.WriteLine($"{c is >= 'a' and <= 'z'} {n is null or > 5} {Length("abc")} {Length(null)}");
}
```

A `switch` on an optional is allowed: `null` matches the missing value, and every other pattern except `var`, `_` and `not ...` needs a value. A constant pattern on an `int?` tests the value, and an `int` constant matches a `double`.

**Property, positional and `var` patterns**. `{ X: 0, Y: > 1 }` matches a value whose fields match; `Point { X: 0 }` also tests the type (on a union, the case; on an optional, not null), and `{ }` matches any value that is not null. `(p, q)` matches a tuple element by element, and `Pair(p, q)` a positional `record struct` by its parameters (tuples and positional records, no user `Deconstruct`). `var x` matches every value, `null` included, and binds it; a name after a property or positional pattern binds the matched value (`{ X: > 0 } p`). A variable under `or` or `not` is an error.

```nz output="origin y-axis 3 far\n1 2 Fizz 4 Buzz Fizz 7 8 Fizz Buzz 11 Fizz 13 14 FizzBuzz \nbig 5 bad\n"
package main;

import std.io.Console;

class Point { public int X = 0; public int Y = 0; }

record struct Pair(int A, int B);

Point P(int x, int y) { Point p = new Point(); p.X = x; p.Y = y; return p; }

string Where(in Point p) => p switch {
    { X: 0, Y: 0 } => "origin",
    { X: 0 } => "y-axis",
    { X: > 5 } => "far",
    { X: var x } => $"{x}",
};

string Fizz(int n) => (n % 3, n % 5) switch {
    (0, 0) => "FizzBuzz",
    (0, _) => "Fizz",
    (_, 0) => "Buzz",
    _ => $"{n}",
};

string Show(Result<int, string> r) => r switch {
    Ok<int>(> 10) => "big",
    Ok<int>(var v) => $"{v}",
    Err<string>(var e) => e,
};

void Main(string[] args) {
    Console.WriteLine($"{Where(P(0, 0))} {Where(P(0, 4))} {Where(P(3, 1))} {Where(P(9, 9))}");
    for (int i = 1; i <= 15; i++) {
        Console.Write($"{Fizz(i)} ");
    }
    Console.WriteLine();
    Console.WriteLine($"{Show(new Ok<int>(40))} {Show(new Ok<int>(5))} {Show(new Err<string>("bad"))}");
}
```

Property patterns read fields (computed properties come with properties); the members of built-in types (`Count`, `Length`) and the objects behind `Shared` and `Arc` handles are not supported yet. The extended form `{ A.X: 0 }` is not supported yet: write `{ A: { X: 0 } }`.

**Guards**. `when` after a pattern adds a condition, which can read the pattern's variables. It runs only when the pattern matched, once, in arm order; when it is false, matching goes on with the next arm. It works on `case` labels too. A guard cannot move, replace or change the value its `switch` tests, nor the list or object that holds it (the later arms still test it), even when only `_` follows; lend the value with `in` instead. The parts inside the tested value and the object behind a `Shared` handle are not held: a guard may change them, and the later arms then see the change:

```nz output="old young cat\nbig small other\n"
package main;

import std.io.Console;

class Dog { public int Age = 0; }
class Cat { }

union Pet(Dog, Cat);

string Kind(Pet p) => p switch {
    Dog d when d.Age > 3 => "old",
    Dog => "young",
    Cat => "cat",
};

string Size(int k) {
    switch (k) {
        case var v when v > 3:
            return "big";
        case 1 or 2:
            return "small";
        default:
            return "other";
    }
}

void Main(string[] args) {
    Dog old = new Dog();
    old.Age = 7;
    Console.WriteLine($"{Kind(old)} {Kind(new Dog())} {Kind(new Cat())}");
    Console.WriteLine($"{Size(5)} {Size(2)} {Size(0)}");
}
```

**Exhaustiveness.** A `switch` expression that does not cover every value is a warning, and at run time a value that no arm matches stops the program with `nz: SwitchExpressionException: Non-exhaustive switch expression failed to match its input.`:

```nz warning="`Dog` not handled"
package main;

import std.io.Console;

class Cat { }
class Dog { }

union Pet(Cat, Dog);

int Legs(in Pet p) => p switch { Cat => 4 };

void Main(string[] args) {
    Console.WriteLine($"{Legs(new Cat())}");
}
```

On an enum, naming every member is not enough: an enum can hold values that have no name (`(Color)2`), so the switch also needs `_`:

```nz warning="but not every value it can hold"
package main;

import std.io.Console;

enum Color { Red, Green }

string Name(Color c) => c switch { Color.Red => "red", Color.Green => "green" };

void Main(string[] args) {
    Console.WriteLine(Name(Color.Green));
}
```

The check looks inside patterns: integer and `char` ranges, `null`, tuples, records, property patterns and union cases at any depth. When some value is left, the warning names one:

```nz warning="for example `(false, false)` is not handled"
package main;

import std.io.Console;

int Code(bool a, bool b) => (a, b) switch {
    (true, _) => 1,
    (false, true) => 2,
};

void Main(string[] args) {
    Console.WriteLine(Code(true, false));
}
```

A guarded arm covers nothing. A `string` or floating-point constant takes nothing away either, so such a `switch` needs `_`. The check stops after 100,000 steps of work: a `switch` that ends with `_` or `var x` is then accepted, only the reachability of its earlier arms left unchecked; one that does not is an error that asks for `_`.

An arm that an earlier arm already covers is an error: after `Cat => 4, Dog => 4` on a `Pet`, `_ => 0` is ``pattern `_` is already handled by an earlier arm``, and after `< 10`, the arm `< 5` is too. A `_` after `{ }` on a class is allowed, because `null` is left.

#### The `switch` statement

**The `switch` statement** has sections that start with one or more `case` labels or `default`, and end with `break`, `return`, `continue` or another way out. Control cannot fall from one section into the next; several labels can share one section:

```nz output="none few many\n"
package main;

import std.io.Console;

string Size(int n) {
    switch (n) {
        case 0:
            return "none";
        case 1:
        case 2:
            return "few";
        default:
            break;
    }
    return "many";
}

void Main(string[] args) {
    Console.WriteLine($"{Size(0)} {Size(2)} {Size(9)}");
}
```

The labels take the patterns of the `switch` expression (`case Ok<int> ok:`), and the sections share one declaration space. A statement that does not cover every value is no warning: nothing runs when no label matches. `goto case`, a pattern variable in a section with more than one label, and a variable declared in one section and used in another are not supported yet.

**Temporaries.** A value made on the spot that a `switch` or an `is` tests lives until the end of the statement that holds it, so a binding into it is valid for the whole statement; for an `if`, that is both branches.

### 1.6.2 Loops

`break` is only valid inside a loop or a `switch` statement, and `continue` inside a loop:

```nz fragment error="only valid inside a loop"
break;
```

#### `do ... while`

`do { ... } while (condition);` runs its body once before the first test.

```nz fragment output="1\n"
int n = 0;
do {
    n++;
} while (n < 1);
Console.WriteLine($"{n}");
```

#### Labeled loops

A loop (`while`, `do`, `for`, `foreach`) can carry a label, and `break label;` or `continue label;` then leaves or continues that loop from inside nested loops. Every `defer` and drop of the scopes left runs once.

```nz fragment output="1\n0 0\n0 1\n1 0\ndone\n"
int n = 0;
do {
    n++;
} while (n < 1);
Console.WriteLine($"{n}");

outer: for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 2) {
            continue outer;
        }
        if (i == 1 && j == 1) {
            break outer;
        }
        Console.WriteLine($"{i} {j}");
    }
}
Console.WriteLine("done");
```

A label is allowed only before a loop. Labels have their own names (a label may share a variable's name), a label may not repeat the label of an enclosing loop, and an unused label is fine.

Previous: [1.4 Data Structures](01-basic-data-structures.md#14-data-structures) · [Contents](README.md) · Next: [1.7 Null safety](01-basic-null-safety-io.md#17-null-safety)
