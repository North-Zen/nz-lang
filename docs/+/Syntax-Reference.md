# Syntax Reference

This page is the reference for the syntax of NZ code: the lexical structure, the built-in types, variables, expressions, statements and patterns. Declarations (types and their members, generics, interfaces, delegates, attributes, packages and the rest) are in [Syntax Reference: Declarations](Syntax-Declarations.md). The [language guide](../README.md) explains the same features as a tour, with more examples.

Every construct has a short entry with its syntax, a short explanation and a small example. Every example in an `nz` block is compiled by the real compiler when the tests run, and the examples with an output are also run.

# 1. Lexical structure

## 1.1 Source files

A source file is UTF-8 text with the extension `.nz`. It starts with a `package` line, then `import` lines, then declarations. Statements end with `;`, and blocks use `{ }`. Whitespace and comments only separate tokens.

```nz output="Hello, NZ\n"
package main;

import std.io.Console;

void Main(string[] args) {
    Console.WriteLine("Hello, NZ");
}
```

- `package` and `import` declare and use packages (see [Declarations](Syntax-Declarations.md)). Top-level statements without a `Main` are not supported.

## 1.2 Comments

`// text` runs to the end of the line. `/* text */` can span lines and does not nest: the first `*/` ends it. A run of `///` lines before a declaration is a documentation comment, written in markdown.

```nz fragment output="3\n"
// a line comment
int x = 3; /* a block comment */
/* a block comment
   over several lines */
Console.WriteLine($"{x}");
```

- Documentation comments are markdown, not XML. A `///` run that belongs to no declaration gives a warning.

## 1.3 Identifiers

An identifier is letters, digits and `_`, and does not start with a digit. Names are case-sensitive.

```nz fragment output="6\n"
int count = 1;
int _hidden2 = 5;
Console.WriteLine($"{count + _hidden2}");
```

A name with a letter outside ASCII is refused:

```text
int café = 1;    // Error: non-ASCII identifier
```

- The first letter of a function, method, field or property name also decides its visibility (see [Declarations](Syntax-Declarations.md)).

## 1.4 Keywords

The compiler's keyword table, generated from the compiler:

<!-- GEN:keywords -->
NZ has 63 keywords: `import` `package` `void` `public` `private` `class` `struct` `enum` `interface` `var` `new` `this` `if` `else` `while` `for` `foreach` `in` `break` `continue` `return` `switch` `case` `default` `is` `ref` `using` `defer` `unsafe` `true` `false` `null` `abstract` `base` `const` `do` `nameof` `operator` `override` `params` `protected` `readonly` `sealed` `sizeof` `static` `type` `union` `virtual` `where` `async` `await` `move` `lock` `yield` `extern` `volatile` `fixed` `stackalloc` `try` `catch` `finally` `throw` `derive`.
<!-- /GEN -->

Some of them are reserved and not available as features yet (`async`, `await`, `lock`, `yield`, `abstract`, `override`, `virtual`, `sealed`, `protected`, `base`, `move`, `volatile`, `derive`): using them as a name gives "`async` is reserved for a future feature".

`checked` and `unchecked` are keywords: nothing can be named with them.

```nz fragment error="`checked` is a keyword; choose another name"
int checked = 1;
```

Contextual words are keywords only in one position and ordinary names elsewhere: `as`, `and`, `or`, `not`, `when` (in patterns), `get`, `set`, `init`, `value` (in properties), `required`, `record`.

```nz fragment output="12\n"
int when = 3;
int record = 4;
int value = 5;
Console.WriteLine($"{when + record + value}");
```

- `package`, `import`, `defer`, `type`, `union` and `move` are NZ keywords. `namespace`, `object`, `decimal`, `dynamic`, `goto` and `typeof` are not NZ keywords today.

## 1.5 Integer literals

Decimal `42`, hexadecimal `0xFF`, binary `0b1010`, with `_` between digits (`1_000_000`). The suffixes `U`, `L` and `UL` (any case, also `LU`) choose an unsigned or long type.

Without a suffix, a literal has the first type that holds its value among `int`, `uint`, `long`, `ulong`. So `3000000000` is a `uint`. A literal that does not fit `ulong` is an error.

```nz fragment output="255 10 1000000 4 5 6 3000000000 -2147483648\n"
int a = 0xFF;
int b = 0b1010;
int c = 1_000_000;
uint d = 4u;
long e = 5L;
ulong f = 6UL;
var g = 3000000000;
Console.WriteLine($"{a} {b} {c} {d} {e} {f} {g} {-2147483648}");
```

```nz fragment error="must stand between two digits"
int x = 1__;
```

- There is no `M` suffix (no `decimal`).

## 1.6 Real literals

A real literal has a fraction (`1.5`), an exponent (`1e5`, `2.5e-3`) or the suffix `F`. With `F` it is a `float`, otherwise a `double`.

```nz fragment output="1.5 0.0025 100000 2.5\n"
double a = 1.5;
double b = 2.5e-3;
double c = 1e5;
float f = 2.5F;
Console.WriteLine($"{a} {b} {c} {f}");
```

```nz fragment error="invalid suffix"
double d = 1.5d;
```

- There are no `decimal` literals.

## 1.7 Character literals

`'a'` is a `char`. The escapes are `\n \t \r \0 \\ \" \'` and `\u{...}` with 1 to 6 hexadecimal digits that name a Unicode scalar value.

```nz fragment output="A 10 128512 '\n"
char c = 'A';
char n = '\n';
char smile = '\u{1F600}';
char q = '\'';
Console.WriteLine($"{c} {(int)n} {(int)smile} {q}");
```

```nz fragment error="unknown escape sequence"
char c = '\x41';
```

- A code point is always written with braces, `\u{41}`; the forms with exactly four or eight digits and no braces, `\x`, `\a`, `\b`, `\f` and `\v` are not supported. A `char` holds any Unicode scalar value, so `'\u{1F600}'` is one `char` (see 2.5).

## 1.8 String literals

`"text"` is a `string`, with the same escapes as characters.

```nz fragment output="tab\there \"q\" \\\n"
string s = "tab\there \"q\" \\";
Console.WriteLine(s);
```

- A string is UTF-8 (see 2.6).

## 1.10 Raw string literals

A raw string starts with `"""` and ends at the next `"""`. It has no escapes. Over several lines, the line of the closing `"""` gives the indentation that is removed from every line.

```nz fragment output="raw \"quoted\" \\n\none\n  two\n"
string r = """raw "quoted" \n""";
Console.WriteLine(r);
string m = """
    one
      two
    """;
Console.WriteLine(m);
```


## 1.11 Interpolated strings

`$"text {expr} text"` inserts values. `{{` and `}}` are literal braces. A hole may end with a format: `F` (fixed point), `D` (digits, zero padded) or `X` (hexadecimal), each with an optional precision of 0 to 99.

```nz fragment output="x = 5, {braces}, 005 FF 1.500\n"
int x = 5;
Console.WriteLine($"x = {x}, {{braces}}, {x:D3} {255:X} {1.5:F3}");
```

A conditional directly in a hole needs parentheses, because its `:` would start a format:

```nz fragment output="b\n"
Console.WriteLine($"{(args.Length > 0 ? "a" : "b")}");
```

- Formatting never depends on the culture (it is always the invariant culture).

## 1.12 UTF-8 literals

`"text"u8` gives a pointer to static, read-only, NUL-terminated UTF-8 bytes. It is only valid inside `unsafe`.

```nz output="104 105 0\n"
package main;

import std.io.Console;

void Main(string[] args) {
    unsafe {
        byte* p = "hi"u8;
        Console.WriteLine($"{p[0]} {p[1]} {p[2]}");
    }
}
```

- The type is `byte*`, and it needs `unsafe`.

## 1.13 Boolean and null literals

`true` and `false` are `bool`. `null` is the empty value of an optional type `T?` (and of a pointer); a non-optional type never holds `null`.

```nz fragment output="True False True\n"
bool t = true;
string? s = null;
Console.WriteLine($"{t} {false} {s == null}");
```

```nz fragment error="null"
string s = null;
```

- `null` is valid only for `T?`; a plain `string` or class type cannot be `null`.

## 1.14 Operators and punctuators

```text
+  -  *  /  %  ++  --
==  !=  <  >  <=  >=
&&  ||  !
&  |  ^  ~  <<  >>  >>>
=  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=  >>>=  ??=
?.  ?[  ??  ?  :  =>  ..  ^
.  ,  ;  ( )  [ ]  { }
```

`?` is also the postfix error-propagation operator (4.37). `=>` is used by `switch` expression arms and expression bodies.

- `->` (member access through a pointer) is not supported; `?` after an expression can be NZ's propagation operator.

## 1.15 Precedence and associativity

From lowest to highest, generated from the compiler's parser:

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

`is` binds at the level of the relational operators. The operand of a `switch` expression is a range expression, so a prefix operator or a cast binds tighter than `switch`, and a binary operator binds looser. Operands are evaluated from left to right.

```nz fragment output="14 20 2 True\n"
int a = 2 + 3 * 4;
int b = (2 + 3) * 4;
int c = 10 - 4 - 4;
bool d = 1 < 2 == true;
Console.WriteLine($"{a} {b} {c} {d}");
```


# 2. Built-in types

## 2.1 Integer types

| Type | Size | Range | Default |
|---|---|---|---|
| `sbyte` | 1 byte | -128 to 127 | `0` |
| `byte` | 1 byte | 0 to 255 | `0` |
| `short` | 2 bytes | -32,768 to 32,767 | `0` |
| `ushort` | 2 bytes | 0 to 65,535 | `0` |
| `int` | 4 bytes | -2,147,483,648 to 2,147,483,647 | `0` |
| `uint` | 4 bytes | 0 to 4,294,967,295 | `0` |
| `long` | 8 bytes | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | `0` |
| `ulong` | 8 bytes | 0 to 18,446,744,073,709,551,615 | `0` |

```nz fragment output="-128 255 -32768 65535 2147483647 4294967295 -9223372036854775808 18446744073709551615\n4 8\n"
sbyte a = -128;
byte b = 255;
short c = -32768;
ushort d = 65535;
int e = int.MaxValue;
uint f = uint.MaxValue;
long g = long.MinValue;
ulong h = ulong.MaxValue;
Console.WriteLine($"{a} {b} {c} {d} {e} {f} {g} {h}");
Console.WriteLine($"{sizeof(int)} {sizeof(long)}");
```

- Arithmetic is checked by default (2.16).

## 2.2 Native-size integers

`nint` and `nuint` have the width of a pointer on the target. A constant given to them must fit `int` or `uint`, so a program means the same on every target.

```nz fragment output="5 7\n"
nint i = 5;
nuint j = 7;
Console.WriteLine($"{i} {j}");
```

```nz fragment error="`sizeof(nint)` is not supported yet"
int s = sizeof(nint);
```

- A constant above the `int`/`uint` range is refused even on a 64-bit target.

## 2.3 Floating-point types

`float` is 4 bytes (IEEE 754 single precision), `double` is 8 bytes (double precision). Their default is `0`.

```nz fragment output="2.5 Infinity 0.33333334 True\n"
double d = 10.0 / 4;
double inf = 1.0 / 0.0;
float f = 1.0F / 3;
Console.WriteLine($"{d} {inf} {f} {double.IsNaN(0.0 / 0.0)}");
```

```nz fragment error="unknown type `decimal`"
decimal m = 1;
```

- A `double` cast to `float` that is too large stops the program instead of giving an infinity (2.15).

## 2.4 `bool`

`bool` holds `true` or `false`. Its default is `false`. It does not convert to or from a number.

```nz fragment error="cannot cast `bool` to `int`"
bool t = true;
int n = (int)t;
```


## 2.5 `char`

A `char` is one Unicode scalar value (U+0000 to U+10FFFF without the surrogates), stored in 4 bytes. Its default is `'\0'`. A `char` converts implicitly to `int`, `uint`, `long`, `ulong`, `nint`, `nuint`, `float` and `double`; an integer never converts to `char` without a cast.

```nz fragment output="65 65 66 4\n"
char c = 'A';
int n = c;
double d = c;
Console.WriteLine($"{n} {d} {'A' + 1} {sizeof(char)}");
```

```nz fragment error="cannot assign `char` to `ushort`"
char c = 'A';
ushort u = c;
```

- An NZ `char` is a whole code point, so `sizeof(char)` is 4 and `char` does not convert to `ushort`. `c += 1` is an error; `c++` works.

## 2.6 `string`

A `string` is an immutable sequence of UTF-8 bytes. `Length` counts bytes, and `s[i]` is the `char` that starts at byte `i`. `foreach (char c in s)` gives the characters. `==` compares bytes, and comparisons and case changes are ordinal (they never depend on a culture). A string is copied freely, never moved.

```nz fragment output="6 é True -31\n97\n233\n"
string s = "héllo";
Console.WriteLine($"{s.Length} {s[1]} {s == "héllo"} {"B".CompareTo("a")}");
foreach (char c in "aé") {
    Console.WriteLine($"{(int)c}");
}
```

- Strings are UTF-8, so `Length` and positions count bytes; comparisons are ordinal, so `"B".CompareTo("a")` is negative; a `string` is never `null` (use `string?`).

## 2.7 Optional types `T?`

`T?` holds a `T` or `null`, for every type `T`. A `T?` must be checked before its value is used: `is`, `== null`, `!= null` and an early `return` narrow it. `??` gives a fallback. For value types, `.HasValue` and `.Value` work.

```nz fragment output="4\n7 True 4\n"
int? a = null;
int? b = 4;
if (b is int v) {
    Console.WriteLine($"{v}");
}
Console.WriteLine($"{a ?? 7} {b.HasValue} {b.Value}");
```

```nz fragment error="possibly null"
string? s = null;
int n = s.Length;
```

- `T?` is the only way to hold `null`, for classes and strings too, and the compiler refuses a use that is not checked. The details are in [Null Safety](Null-Safety.md).

## 2.8 Arrays

`T[]` is a one-dimensional array on the heap, created with `new T[n]`, an initializer or a collection expression. `Length` gives its size; an index out of range stops the program.

```nz fragment output="3 3 5 6 8 0\n"
int[] a = new int[3];
int[] b = { 1, 2, 3 };
int[] c = [4, 5];
var d = new[] { 6, 7 };
var e = new int[] { 8 };
Console.WriteLine($"{a.Length} {b[2]} {c[1]} {d[0]} {e[0]} {a[0]}");
```

```nz fragment error="expected an expression"
int[,] grid = new int[2, 3];
```

- An array of a class type owns its elements; see [Ownership](Ownership.md).

## 2.9 Spans

`Span<T>` is a borrowed view over contiguous elements (an array, a list or a stack buffer). Today a span is a parameter type: an array or list passed to it is borrowed for that call.

```nz output="6\n"
package main;

import std.io.Console;

int Sum(Span<int> xs) {
    int s = 0;
    foreach (int x in xs) {
        s += x;
    }
    return s;
}

void Main(string[] args) {
    int[] a = [1, 2, 3];
    Console.WriteLine($"{Sum(a)}");
}
```

```nz fragment error="a span cannot be stored in a local variable"
int[] a = [1, 2, 3];
Span<int> s = a;
```

- A span lives for one call; an `in Span<T>` is read-only.

## 2.10 `Index` and `Range`

`^n` is an index counted from the end, and `a..b` is a range (both ends optional, the end excluded). They have the types `Index` and `Range`.

```nz fragment output="4 1..^1\n"
int[] a = [1, 2, 3, 4, 5];
Index i = ^2;
Range r = 1..^1;
Console.WriteLine($"{a[i]} {r}");
```


## 2.11 Tuples

`(T1, T2, ...)` is a value tuple of two or more elements. Elements may have names; every element is also `Item1`, `Item2`, and so on. `var (a, b) = t;` deconstructs it, and `_` discards an element.

```nz fragment output="1 a 3 4 1 a 4\n"
(int, string) t = (1, "a");
(int x, int y) p = (3, 4);
var (q, r) = t;
var (_, w) = p;
Console.WriteLine($"{t.Item1} {t.Item2} {p.x} {p.y} {q} {r} {w}");
```

```nz fragment error="cannot assign to this expression"
int x = 0;
int y = 1;
(x, y) = (y, x);
```

- A tuple that holds an owned value is moved as a whole.

## 2.12 Pointers

Inside `unsafe`, `T*` is a pointer, `&x` takes an address, `*p` reads or writes through it, and `p[i]` indexes it. `null` is a valid pointer.

```nz output="7 7 8\n"
package main;

import std.io.Console;

void Main(string[] args) {
    unsafe {
        int x = 5;
        int* p = &x;
        *p = 7;
        Console.WriteLine($"{x} {*p} {sizeof(int*)}");
    }
}
```

```nz fragment error="can only be used inside `unsafe`"
int x = 5;
int* p = &x;
```

- The ownership rules still apply to managed values inside `unsafe`.

## 2.14 Default values

`default` and `default(T)` give the default of a type: `0` for numbers, `false`, `'\0'`, `""` for a `string`, and `null` for an optional. A bare `default` takes its type from the target.

```nz fragment output="0 0 False 0 True\n"
var x = default(int);
int y = default;
bool b = default;
char c = default;
int? n = default;
Console.WriteLine($"{x} {y} {b} {(int)c} {n == null}");
```

```nz fragment output="True True\n"
string s = default;
string? t = default;
Console.WriteLine($"{s == ""} {t == null}");
```

- In NZ a non-optional type never holds `null`, so the default of `string` is `""`, as the default of `int` is `0`. An empty `T?` is never turned into a `T` by itself: write `s ?? ""`.

## 2.15 Conversions

**Implicit conversions** exist only where every value of the source fits the target: `sbyte` to `short`, `int`, `long`, `nint`, `float`, `double`; `byte` to every wider integer type and the reals; `int` to `long`, `nint`, `float`, `double`; `uint` to `long`, `ulong`, `nuint`, `float`, `double`; `long` and `ulong` to `float` and `double`; `float` to `double`; and `char` as in 2.5. An integer constant converts to any integer type that holds it (`byte b = 7;`).

When two operands meet, both are converted to one type first: `double` if either is `double`, else `float`, else `ulong`, `long`, `uint` or `int`. `ulong` with a signed type has no common type.

```nz fragment output="5 5 3 4\n"
int x = 5;
long y = x;
double z = x;
float f = 3;
uint u = 5;
int i = -1;
long r = u + i;
Console.WriteLine($"{y} {z} {f} {r}");
```

```nz fragment error="no integer type holds both; cast one side"
ulong a = 5;
int b = 1;
var c = a + b;
```

**Explicit conversions** are casts, `(T)x`. A cast is checked: a value that does not fit stops the program, and a constant that does not fit is a compile-time error. The pairs: integer to integer, `float` or `double`; real to integer (truncated toward zero), `char` or the other real type; `char` to integer or real; integer to `char`; an enum to and from those types; and any type to itself.

```nz fragment output="200 2 -2 A 65\n"
int i = 200;
double d = 2.9;
Console.WriteLine($"{(byte)i} {(int)d} {(int)-d} {(char)65} {(int)'A'}");
```

```nz fragment error="constant `300` does not fit `byte`"
byte b = (byte)300;
```

- Casts are checked even outside `checked`; `(Name)(x)` is a call of `Name`, not a cast (write the cast of a parenthesised value to a built-in type, or use a variable); a too large `double` cast to `float` stops instead of giving an infinity.

## 2.16 Checked arithmetic

Integer arithmetic is **checked by default**: an overflow stops the program with `nz: integer overflow`, and a narrowing back into a small type stops with ``nz: value does not fit in `byte` ``. `unchecked(expr)` and `unchecked { ... }` make it wrap instead; `checked(...)` and `checked { ... }` restore the default. Division by zero always stops.

```nz fragment output="-2147483648 4 2147483646\n"
int big = 2147483647;
int wrapped = unchecked(big + 1);
byte b = 0;
unchecked { b--; }
int ok = checked(big - 1);
Console.WriteLine($"{wrapped} {unchecked((byte)260)} {ok}");
```

```nz fragment error="constant expression overflows `int`"
int x = 2147483647 + 1;
```

- Arithmetic is checked by default. Today an overflow stops the program.

# 3. Variables and constants

## 3.1 Local variables and `var`

`T name = value;` declares a local with a type; `var name = value;` takes the type from the value. Locals are mutable.

```nz fragment output="5 1.5 hi\n"
int a = 5;
var b = 1.5;
var c = "hi";
Console.WriteLine($"{a} {b} {c}");
```

```nz fragment error="cannot infer the type of a variable from `null`"
var n = null;
```

- Assigning a class value to another variable moves it (3.6).

## 3.2 Definite assignment

Every non-optional local must be initialized where it is declared.

```nz fragment error="must be initialized"
int x;
```

- Write `int x = ...;`, or use a conditional, a `switch` expression or a local function.

## 3.3 Scope

A local is visible from its declaration to the end of its block. A name cannot be declared again while a local of that name is visible, in the same block or an inner one.

```nz fragment error="already declared"
int x = 1;
{
    int x = 2;
}
```


## 3.4 `const`

`const T Name = value;` declares a compile-time constant, local or at the top level. The value is a constant expression.

```nz output="20 nz 20000\n"
package main;

import std.io.Console;

const int Max = 10 * 2;
const string Name = "nz";

void Main(string[] args) {
    const long Big = Max * 1000L;
    Console.WriteLine($"{Max} {Name} {Big}");
}
```

```nz fragment error="cannot be a constant type"
const double D = 1.5;
```

- Real and `char` constants are not supported.

## 3.6 Ownership of variables

A class object has one owner. Assigning it to another variable, or passing it by value, **moves** it; the old variable cannot be used until it gets a new value. Numbers, `bool`, `char`, `string` and structs of such values are copied. `in` and `ref` lend a value for one call instead (4.4).

```nz error="use of moved value `a`"
package main;

import std.io.Console;

class Box { public int V = 1; }

void Main(string[] args) {
    Box a = new Box();
    Box b = a;
    Console.WriteLine($"{a.V}");
}
```

- Assigning a class value moves it, and both variables are not usable afterwards. See [Ownership](Ownership.md).

# 4. Expressions

## 4.1 Simple names, literals, parentheses and `this`

A name refers to a local, a parameter, a member or a type. `( e )` groups. `this` is the current object inside a method.

```nz output="3\n"
package main;

import std.io.Console;

class Point {
    public int X = 1;
    public int Y = 2;
    public int Sum() { return this.X + Y; }
}

void Main(string[] args) {
    var p = new Point();
    Console.WriteLine($"{p.Sum()}");
}
```

- `base` is not available.

## 4.2 Member access

`x.y` reads a field, property or method of `x`, a static member of a type (`int.MaxValue`), or a member of an enum.

```nz fragment output="3 ABC 2147483647\n"
string s = "abc";
Console.WriteLine($"{s.Length} {s.ToUpper()} {int.MaxValue}");
```

- `->` is not supported.

## 4.3 Invocation

`f(a, b)` calls a function or method. Arguments are evaluated from left to right, also when they are named.

```nz output="7\n"
package main;

import std.io.Console;

int Add(int a, int b) {
    return a + b;
}

void Main(string[] args) {
    Console.WriteLine($"{Add(3, 4)}");
}
```

- Passing a class object by value moves it into the call.

## 4.4 `ref`, `in` and `out` arguments

A `ref` parameter lends the argument for writing and an `in` parameter lends it for reading, for the time of the call. The caller writes the argument **without** a keyword.

```nz output="2 3\n"
package main;

import std.io.Console;

void Bump(ref int x) {
    x++;
}

int Next(in int x) {
    return x + 1;
}

void Main(string[] args) {
    int n = 1;
    Bump(n);
    Console.WriteLine($"{n} {Next(n)}");
}
```

```nz error="expected an expression, found Ref"
package main;

void Bump(ref int x) {
    x++;
}

void Main(string[] args) {
    int n = 1;
    Bump(ref n);
}
```

- `ref` is not written at the call site; NZ refuses it. A `ref` argument cannot share storage with another argument of the same call.

## 4.5 Named and default arguments

A parameter can have a default value; a call may leave it out. An argument can name its parameter. Positional arguments come first.

```nz output="20 6\n"
package main;

import std.io.Console;

int Scale(int a, int b = 10) {
    return a * b;
}

void Main(string[] args) {
    Console.WriteLine($"{Scale(2)} {Scale(b: 3, a: 2)}");
}
```


## 4.6 `params`

`params T[] xs` or `params Span<T> xs` as the last parameter takes any number of arguments.

```nz output="3 2\n"
package main;

import std.io.Console;

int Count(params int[] xs) {
    return xs.Length;
}

int Total(params Span<int> xs) {
    return xs.Length;
}

void Main(string[] args) {
    Console.WriteLine($"{Count(1, 2, 3)} {Total(4, 5)}");
}
```

- With `params T[]` the array moves into the callee.

## 4.7 Element access and indexes from the end

`a[i]` reads or writes an element of an array, a list, a string or a span. `a[^1]` is the last element.

```nz fragment output="3 10 1 l\n"
int[] a = [3, 1];
List<int> l = [1, 2, 10];
int i = 0;
Console.WriteLine($"{a[0]} {l[^1]} {a[^1]} {"hello"[2]}");
```

- A string index is a byte position (2.6); `a[i]` of a class element is a borrowed alias, not a copy of a reference.

## 4.8 Ranges

`a[x..y]` takes a part. On a `string` it gives a new string; on a span parameter, a span over the same storage.

```nz fragment output="el he llo\n"
string s = "hello";
Console.WriteLine($"{s[1..3]} {s[..2]} {s[^3..]}");
```


## 4.9 Object creation and object initializers

`new T(args)` creates an object. An object initializer sets members after the constructor: `new T { X = 1 }`.

```nz output="1 2\n"
package main;

import std.io.Console;

class Point {
    public int X = 0;
    public int Y = 0;
}

void Main(string[] args) {
    var p = new Point { X = 1, Y = 2 };
    Console.WriteLine($"{p.X} {p.Y}");
}
```

```nz fragment error="collection initializers"
List<int> l = new List<int> { 1, 2 };
```

- The new object has one owner (3.6).

## 4.10 Array creation

`new T[n]` creates `n` default elements. `new T[] { ... }`, `new[] { ... }` and `T[] a = { ... };` list the elements.

```nz fragment output="3 2 7\n"
int[] a = new int[3];
var b = new[] { 1, 2 };
int[] c = { 7 };
Console.WriteLine($"{a.Length} {b[1]} {c[0]}");
```

```nz fragment output="3 True\n"
var names = new string[3];
Console.WriteLine($"{names.Length} {names[0] == ""}");
```

- The element type needs a default value, so `new Node[3]` for a class `Node` is refused (use `new Node?[3]`); the elements of a `string[]` are `""`.

## 4.11 Collection expressions and spread

`[a, b, c]` creates an array, a `List<T>` or a span argument, from its target type. `..xs` spreads the elements of another collection.

```nz fragment output="3 4 3\n"
int[] a = [1, 2];
int[] b = [..a, 3];
List<int> l = [0, ..b];
Console.WriteLine($"{b.Length} {l.Count} {l[3]}");
```

```nz fragment error="there is no target type for the collection expression"
var xs = [1, 2];
```

- Spreading owned elements is refused (they would get a second owner).

## 4.12 Null-conditional `?.` and `?[]`

`x?.y` and `x?[i]` give `null` when `x` is `null`, and otherwise access the member or element. The result is optional.

```nz fragment output="True True 3\n"
List<int>? l = null;
int? first = l?[0];
string? s = null;
int? n = s?.Length;
string t = "abc";
Console.WriteLine($"{first == null} {n == null} {t?.Length}");
```


## 4.13 Null-coalescing `??` and `??=`

`a ?? b` is `a` when it is not `null`, else `b`. `x ??= v` assigns `v` only when `x` is `null`; as a statement it narrows `x` to non-null.

```nz fragment output="7 5\n"
int? a = null;
int b = a ?? 7;
int? c = null;
c ??= 5;
int d = c;
Console.WriteLine($"{b} {d}");
```

- The left side of `??=` must be optional.

## 4.15 Arithmetic operators

`+ - * / %`, unary `+` and `-`. `+` also joins strings. Integer division truncates toward zero.

```nz fragment output="2 -2 1 ab1 2.5\n"
Console.WriteLine($"{5 / 2} {-5 / 2} {7 % 3} {"a" + "b" + 1} {10.0 / 4}");
```

- Integer overflow and narrowing are checked (2.16).

## 4.16 Shift and bitwise operators

`<< >> >>>` shift; `& | ^ ~` work bit by bit on integers, and `& | ^` also on `bool` (both sides evaluated). `>>` keeps the sign of a signed type; `>>>` always shifts in zeros. The count is masked to the width of the type.

```nz fragment output="20 15 4294967055 48 241 15 True 1099511627776\n"
int x = 10;
x = x << 2 >> 1;
uint u = 0xF0;
long l = 1L << 40;
Console.WriteLine($"{x} {u >>> 4} {~u} {u & 0x30} {u | 1} {u ^ 0xFF} {true ^ false} {l}");
```


## 4.17 Comparison and equality

`< > <= >=` compare numbers and `char`; `==` and `!=` compare numbers, `bool`, `char`, strings (by bytes), enums and optionals.

```nz fragment output="True True False True\n"
Console.WriteLine($"{1 < 2} {"a" == "a"} {'a' == 'b'} {1 == 1.0}");
```

- String comparison is ordinal (it never depends on a culture).

## 4.18 Logical operators

`&&` and `||` evaluate the right side only when needed; `!` negates.

```nz fragment output="False True True\n"
bool a = true;
bool b = false;
Console.WriteLine($"{a && b} {a || b} {!b}");
```


## 4.19 Increment and decrement

`x++`, `x--`, `++x`, `--x` on numbers, `char` and enums.

```nz fragment output="4 2 1\n"
int i = 1;
int a = i++ + ++i;
int b = i-- - --i;
Console.WriteLine($"{a} {b} {i}");
```

- Checked, also on small types.

## 4.20 Assignment and compound assignment

`=` assigns and gives the assigned value. `+= -= *= /= %= &= |= ^= <<= >>= >>>= ??=` combine an operator with assignment; the target and its index are evaluated once. All are right-associative.

```nz fragment output="3 1 2 10 1\n"
int x = 1;
int y = 2;
int z = x = y;
x += y -= 1;
int[] a = [9];
int i = 0;
a[i++] += 1;
Console.WriteLine($"{x} {y} {z} {a[0]} {i}");
```

- Assigning a class value moves it; the old value of the target is dropped.

## 4.21 Conditional operator `?:`

`c ? a : b` evaluates one of its arms.

```nz fragment output="yes\n"
int x = 3;
Console.WriteLine(x > 2 ? "yes" : "no");
```


## 4.22 `is`

`x is T`, `x is T v` and `x is pattern` test a value against a type or a pattern (section 6) and give a `bool`. A variable declared by the pattern is in scope where the test is true.

```nz fragment output="True True 5\n"
int? n = 5;
int x = 7;
Console.WriteLine($"{x is int} {x is > 1 and < 10} {(n is int v ? v : 0)}");
```


## 4.24 Cast expressions

`(T)x` converts explicitly; the rules are in 2.15. A cast binds like a prefix operator.

```nz fragment output="66 3\n"
double d = 3.7;
Console.WriteLine($"{(int)'A' + 1} {(int)d}");
```

- See 2.15.

## 4.25 `switch` expressions

`x switch { pattern => value, ... }` gives the value of the first arm whose pattern matches. Arms are separated by commas, and `_` matches anything. An arm that earlier arms already cover is an error; a `switch` that does not cover every value is a warning, and an unmatched value stops the program with `SwitchExpressionException`.

```nz fragment output="small\n"
int x = 3;
string s = x switch {
    < 0 => "negative",
    0 => "zero",
    > 0 and < 10 => "small",
    _ => "big",
};
Console.WriteLine(s);
```

```nz fragment warning="does not cover every value"
int x = 7;
string s = x switch { 1 => "a", 2 => "b" };
```


## 4.27 `nameof`

`nameof(x)` is the last name of `x` as a `string` constant.

```nz fragment output="count Length\n"
int count = 1;
string s = "a";
Console.WriteLine($"{nameof(count)} {nameof(s.Length)}");
```


## 4.29 `sizeof`

`sizeof(T)` is the size in bytes of a built-in numeric type, `bool`, `char`, an enum, or (in `unsafe`) a pointer or a struct.

```nz fragment output="4 8 1 4\n"
Console.WriteLine($"{sizeof(int)} {sizeof(double)} {sizeof(bool)} {sizeof(char)}");
```

- `sizeof(char)` is 4 (2.5).

## 4.30 `default`

See 2.14.

- See 2.14.

## 4.31 `checked` and `unchecked` expressions

`checked(e)` and `unchecked(e)` choose the overflow behaviour of the integer operations written inside `e` (2.16).

```nz fragment output="-2147483648 44\n"
int big = int.MaxValue;
Console.WriteLine($"{unchecked(big + 1)} {unchecked((byte)300)}");
```

- The default outside is checked; `/`, `%`, real-to-integer casts and integer-to-`char` casts stay checked inside `unchecked`.

## 4.36 `stackalloc`

`stackalloc T[n]` allocates `n` elements on the stack. Today it initializes a pointer local inside `unsafe`.

```nz output="1\n"
package main;

import std.io.Console;

void Main(string[] args) {
    unsafe {
        int* q = stackalloc int[4];
        q[0] = 1;
        Console.WriteLine($"{q[0]}");
    }
}
```

```nz fragment error="stackalloc can only initialize a pointer local"
Span<int> s = stackalloc int[4];
```

- `stackalloc` initializes only a pointer local, inside `unsafe`; the `Span<int>` form is refused.

## 4.37 Error propagation `?` (NZ only)

`e?` after an expression of type `Result<T, E>` gives the `Ok` value, or returns the `Err` from the function at once. On a `T?` in a function that returns an optional, it returns `null`. It is NZ's own operator.

```nz output="7 -1\n"
package main;

import std.io.Console;

Result<int, string> Parse(string s) {
    if (s == "7") {
        return new Ok<int>(7);
    }
    return new Err<string>("bad");
}

Result<int, string> Twice(string s) {
    int v = Parse(s)?;
    return new Ok<int>(v);
}

int Show(Result<int, string> r) => r switch {
    Ok<int> ok => ok.Value,
    Err<string> => -1,
};

void Main(string[] args) {
    Console.WriteLine($"{Show(Twice("7"))} {Show(Twice("x"))}");
}
```

- When the next token can start an expression, `?` is the conditional operator: write `(f()?) - 1`. More in [Error Handling](Error-Handling.md).

# 5. Statements

## 5.1 Blocks and the empty statement

`{ ... }` groups statements and opens a scope.

```nz fragment output="1\n"
{
    int x = 1;
    Console.WriteLine($"{x}");
}
```

```nz fragment error="expected an expression, found Semi"
;
```

- An opening `{` stays on the same line as its statement (a style rule of NZ).

## 5.2 Declaration and expression statements

A declaration (3.1, 3.4), a local function (5.18) or an expression with an effect (an assignment, a call, `++`, `--`, `new`, `e?`) followed by `;`.

```nz fragment output="2\n"
int x = 1;
x++;
Console.WriteLine($"{x}");
```

- `F()?;` is allowed as a statement (the value is discarded).

## 5.3 `if` and `else`

```nz fragment output="two\n"
int x = 2;
if (x == 1) {
    Console.WriteLine("one");
} else if (x == 2) {
    Console.WriteLine("two");
} else {
    Console.WriteLine("other");
}
```

- A test such as `x != null` narrows an optional inside the branch.

## 5.4 `switch` statement

Sections start with one or more `case` labels or `default`, and a section may not fall through to the next one. Labels take every pattern of section 6 and `when` guards.

```nz fragment output="two or three\n"
int x = 2;
switch (x) {
    case 1:
        Console.WriteLine("one");
        break;
    case 2 or 3:
        Console.WriteLine("two or three");
        break;
    case int n when n > 5:
        Console.WriteLine("big");
        break;
    default:
        Console.WriteLine("other");
        break;
}
```

```nz fragment error="control cannot fall through"
int x = 1;
switch (x) {
    case 1:
        Console.WriteLine("one");
    case 2:
        break;
}
```


## 5.5 `while`

```nz fragment output="3\n"
int j = 0;
while (j < 3) {
    j++;
}
Console.WriteLine($"{j}");
```


## 5.6 `do ... while`

```nz fragment output="2\n"
int k = 0;
do {
    k++;
} while (k < 2);
Console.WriteLine($"{k}");
```


## 5.7 `for`

`for (init; condition; step)`; each part may be empty.

```nz fragment output="10\n"
int total = 0;
for (int i = 0; i < 5; i++) {
    total += i;
}
for (;;) {
    break;
}
Console.WriteLine($"{total}");
```


## 5.8 `foreach`

`foreach (T x in c)` walks an array, a list, a dictionary, a set, a string or a span. The collection cannot change during the loop.

```nz fragment output="3\n"
List<int> xs = [1, 2];
int total = 0;
foreach (var x in xs) {
    total += x;
}
Console.WriteLine($"{total}");
```

- The loop variable of a class element is a borrowed alias; the collection is frozen while it is walked.

## 5.9 `break` and `continue`

`break` leaves the innermost loop or `switch`; `continue` starts the next iteration. With a label they target an outer loop: `break outer;`, `continue outer;`.

```nz fragment output="20\n"
int total = 0;
outer: for (int a = 0; a < 3; a++) {
    for (int b = 0; b < 3; b++) {
        if (b == 1) continue outer;
        if (a == 2) break outer;
        total += 10;
    }
}
Console.WriteLine($"{total}");
```

- Labeled `break` and `continue` are NZ's own way to leave an outer loop.

## 5.11 `return`

`return;` leaves a `void` function; `return e;` gives its result. Pending `defer` statements and drops run first.

```nz output="-1 4\n"
package main;

import std.io.Console;

int Get(int? v) {
    if (v == null) return -1;
    return v;
}

void Main(string[] args) {
    Console.WriteLine($"{Get(null)} {Get(4)}");
}
```

- An early `return` narrows an optional after the test.

## 5.12 `using`

`using (var x = e) { ... }` owns `x` for the block and drops it at the end. Calling `Dispose` of an `IDisposable` at that point comes with interfaces.

```nz output="in 1\nend\n"
package main;

import std.io.Console;

class R { public int V = 1; }

void Main(string[] args) {
    using (var r = new R()) {
        Console.WriteLine($"in {r.V}");
    }
    Console.WriteLine("end");
}
```

- The block owns the value, so `using (x) { }` with an existing local moves `x` into the block.

## 5.14 `fixed`

Inside `unsafe`, `fixed (T* p = array) { ... }` gives a pointer to the elements for the block.

```nz output="8\n"
package main;

import std.io.Console;

void Main(string[] args) {
    unsafe {
        int[] arr = [9, 8];
        fixed (int* f = arr) {
            Console.WriteLine($"{f[1]}");
        }
    }
}
```


## 5.15 `defer` (NZ only)

`defer statement;` or `defer { ... }` runs when the enclosing block ends, in reverse order of the `defer` statements, on every way out of the block.

```nz fragment output="body\nsecond\nfirst\n"
{
    defer Console.WriteLine("first");
    defer Console.WriteLine("second");
    Console.WriteLine("body");
}
```


## 5.16 `unsafe`

`unsafe { ... }` turns on pointers, `&`, `fixed`, `stackalloc`, mutable statics and calls into C for the block.

```nz output="1\n"
package main;

import std.io.Console;

static int counter = 0;

void Main(string[] args) {
    unsafe {
        counter++;
        Console.WriteLine($"{counter}");
    }
}
```

- A mutable static is only usable inside `unsafe`; types, null safety, overflow checks and the ownership of managed values stay checked inside `unsafe`.

## 5.17 `checked` and `unchecked` blocks

`checked { ... }` and `unchecked { ... }` set the overflow behaviour of the statements inside (2.16).

```nz fragment output="255\n"
byte b = 0;
unchecked {
    b--;
}
Console.WriteLine($"{b}");
```

- The default outside is checked.

## 5.18 Local functions

A function declared directly inside a block is visible in that block, before or after its declaration. It reads the variables around it as a read-only borrow for each call.

```nz fragment output="16 6\n"
int factor = 3;
int square(int v) {
    return v * v;
}
int times(int v) => v * factor;
Console.WriteLine($"{square(4)} {times(2)}");
```

```nz fragment warning="visible only in its block"
int Square(int v) {
    return v * v;
}
Console.WriteLine($"{Square(2)}");
```

- A local function cannot change the variables it captures; an uppercase name gives a warning (lowercase means "not public").

## 5.19 `try`, `catch` and `finally`

`try { ... } finally { ... }` works: the `finally` block runs on every way out of the `try` block (its end, `return`, `break`, `continue`, `?`); see [Error Handling](Error-Handling.md). `catch (E e) when (...)` parses and is refused as "not supported yet"; exception classes are not available. Library members and run-time checks (index out of range, overflow, a failed `Assert`) do not throw exceptions yet. The no-throw way stays available: `Try`-prefixed members and `Result` with `?`.

```nz fragment output="a\nb\n"
try {
    Console.WriteLine("a");
} finally {
    Console.WriteLine("b");
}
```

- `throw` and `catch` are not available yet; `try`/`finally` works.

## 5.22 Deconstruction statement

`var (a, b) = e;` splits a tuple or a positional record into new variables; `_` discards an element.

```nz fragment output="1 a\n"
var (n, s) = (1, "a");
Console.WriteLine($"{n} {s}");
```

- See 2.11.

# 6. Patterns

Patterns appear after `is`, in `switch` expression arms and in `case` labels.

## 6.1 Type and declaration patterns

`T` tests that a value is a `T` (for an optional: not `null`; for a union: holds the case `T`). `T name` also binds the value.

```nz output="circle 2\nsquare\n"
package main;

import std.io.Console;

union Shape(Circle, Square);
record struct Circle(double R);
record struct Square(double S);

string Describe(Shape s) => s switch {
    Circle c => $"circle {c.R}",
    Square => "square",
};

void Main(string[] args) {
    Console.WriteLine(Describe(new Circle(2.0)));
    Console.WriteLine(Describe(new Square(3.0)));
}
```

- A binding of an owned value borrows it; it does not copy a reference. Tests against base classes are not available yet.

## 6.2 Constant patterns

A literal, `null`, a named constant or an enum member. On a `T?`, a constant tests the value.

```nz fragment output="null\n"
string? s = null;
switch (s) {
    case null:
        Console.WriteLine("null");
        break;
    case "a":
        Console.WriteLine("a");
        break;
    default:
        Console.WriteLine("other");
        break;
}
```


## 6.3 Relational patterns

`< v`, `<= v`, `> v`, `>= v` on integers, `char`, `float`, `double` and enums.

```nz fragment output="True\n"
char ch = 'x';
Console.WriteLine($"{ch is >= 'a' and <= 'z'}");
```


## 6.4 Logical patterns: `not`, `and`, `or`

`not` binds tighter than `and`, which binds tighter than `or`; parentheses group.

```nz fragment output="a True\n"
int x = 3;
string s = x switch { (> 0 and < 5) or 10 => "a", _ => "b" };
int? n = 5;
Console.WriteLine($"{s} {n is not null}");
```

```nz fragment error="a variable cannot be declared under `not`"
int? v = 1;
if (v is not int n) {
    return;
}
```

- A variable cannot be declared under `not` (`x is not T v`); test the positive form instead.

## 6.5 Property patterns

`{ Member: pattern, ... }`, optionally after a type and followed by a name, tests fields of classes, structs and tuples.

```nz output="y-axis 3\n"
package main;

import std.io.Console;

class Point {
    public int X = 0;
    public int Y = 0;
}

void Main(string[] args) {
    var p = new Point { X = 0, Y = 3 };
    string a = p switch {
        { X: 0, Y: > 1 } => "y-axis",
        _ => "other",
    };
    if (p is Point { X: 0 } q) {
        Console.WriteLine($"{a} {q.Y}");
    }
}
```


## 6.6 Positional patterns

`(p1, p2, ...)` matches a tuple or a positional record by position.

```nz output="one 2 match\n"
package main;

import std.io.Console;

record struct Pair(int A, int B);

void Main(string[] args) {
    var pr = new Pair(1, 2);
    string b = pr switch { (1, var y) => $"one {y}", _ => "no" };
    var t = (1, "a");
    string c = t switch { (1, "a") => "match", _ => "no" };
    Console.WriteLine($"{b} {c}");
}
```

- Works on tuples and positional record structs; a user `Deconstruct` method is not used.

## 6.7 `var` pattern

`var x` matches any value and binds it.

```nz fragment output="one 2\n"
var t = (1, 2);
string s = t switch { (1, var y) => $"one {y}", _ => "no" };
Console.WriteLine(s);
```


## 6.8 Discard pattern

`_` matches anything without binding.

```nz fragment output="other\n"
int x = 9;
Console.WriteLine(x switch { 1 => "one", _ => "other" });
```


## 6.10 `when` guards

`pattern when condition` adds a condition to a `switch` arm or a `case` label.

```nz fragment output="big\n"
int x = 12;
string s = x switch {
    int n when n > 10 => "big",
    _ => "small",
};
Console.WriteLine(s);
```


## 6.11 Exhaustiveness

The compiler computes which values a `switch` expression covers. A missing value is a warning that names an example; an arm that earlier arms cover is an error. On an enum, a `switch` without `_` warns, since an enum can hold values that have no name. A `switch` over a union that names every case needs no `_`.

```nz fragment error="already handled by an earlier arm"
int x = 3;
string s = x switch { > 0 => "positive", 1 => "one", _ => "other" };
```

```nz warning="`Blue` not handled"
package main;

enum Color { Red, Green, Blue }

void Main(string[] args) {
    Color c = Color.Green;
    string s = c switch { Color.Red => "r", Color.Green => "g" };
}
```

- A non-exhaustive `switch` gives a warning, and an arm already covered is an error. String and floating-point constants never complete a `switch`, so such a `switch` needs `_`.

# 7. Index

| Construct | Section |
|---|---|
| Source files | 1.1 |
| Comments, documentation comments | 1.2 |
| Identifiers | 1.3 |
| Keywords | 1.4 |
| Integer literals | 1.5 |
| Real literals | 1.6 |
| Character literals | 1.7 |
| String literals | 1.8 |
| Raw string literals | 1.10 |
| Interpolated strings | 1.11 |
| UTF-8 literals `u8` | 1.12 |
| `true`, `false`, `null` | 1.13 |
| Operators and punctuators | 1.14 |
| Precedence and associativity | 1.15 |
| Integer types | 2.1 |
| `nint`, `nuint` | 2.2 |
| `float`, `double` | 2.3 |
| `bool` | 2.4 |
| `char` | 2.5 |
| `string` | 2.6 |
| Optional types `T?` | 2.7 |
| Arrays | 2.8 |
| Spans | 2.9 |
| `Index` and `Range` | 2.10 |
| Tuples | 2.11 |
| Pointers | 2.12 |
| Default values | 2.14 |
| Conversions and casts | 2.15 |
| Checked arithmetic | 2.16 |
| Local variables and `var` | 3.1 |
| Definite assignment | 3.2 |
| Scope | 3.3 |
| `const` | 3.4 |
| Moves | 3.6 |
| Simple names, parentheses, `this` | 4.1 |
| Member access | 4.2 |
| Invocation | 4.3 |
| `ref`, `in`, `out` arguments | 4.4 |
| Named and default arguments | 4.5 |
| `params` | 4.6 |
| Element access, `^` | 4.7 |
| Ranges `..` | 4.8 |
| `new`, object initializers | 4.9 |
| Array creation | 4.10 |
| Collection expressions, spread | 4.11 |
| `?.`, `?[]` | 4.12 |
| `??`, `??=` | 4.13 |
| Arithmetic operators | 4.15 |
| Shift and bitwise operators | 4.16 |
| Comparison and equality | 4.17 |
| Logical operators | 4.18 |
| Increment and decrement | 4.19 |
| Assignment, compound assignment | 4.20 |
| Conditional `?:` | 4.21 |
| `is` | 4.22 |
| Cast expressions | 4.24 |
| `switch` expressions | 4.25 |
| `nameof` | 4.27 |
| `sizeof` | 4.29 |
| `default` | 4.30 |
| `checked`, `unchecked` expressions | 4.31 |
| `stackalloc` | 4.36 |
| Error propagation `?` | 4.37 |
| Blocks, empty statement | 5.1 |
| Declaration and expression statements | 5.2 |
| `if`, `else` | 5.3 |
| `switch` statement | 5.4 |
| `while` | 5.5 |
| `do ... while` | 5.6 |
| `for` | 5.7 |
| `foreach` | 5.8 |
| `break`, `continue`, labels | 5.9 |
| `return` | 5.11 |
| `using` | 5.12 |
| `fixed` | 5.14 |
| `defer` | 5.15 |
| `unsafe` | 5.16 |
| `checked`, `unchecked` blocks | 5.17 |
| Local functions | 5.18 |
| Deconstruction statement | 5.22 |
| Type and declaration patterns | 6.1 |
| Constant patterns | 6.2 |
| Relational patterns | 6.3 |
| `not`, `and`, `or` | 6.4 |
| Property patterns | 6.5 |
| Positional patterns | 6.6 |
| `var` pattern | 6.7 |
| Discard `_` | 6.8 |
| `when` guards | 6.10 |
| Exhaustiveness | 6.11 |

# Not available yet

These constructs are not available in NZ:

- Verbatim strings
- `object` and `dynamic`
- `ref` locals
- Null-forgiving `x!`
- `as`
- `with` expressions
- `typeof`
- Lambdas
- Anonymous methods
- `await`
- `throw` expressions
- `lock`
- `throw`
- `yield`
- List patterns

The compiler refuses them:

**Verbatim strings**

```nz fragment error="unexpected character"
string s = @"C:\temp";
```

**`object` and `dynamic`**

```nz fragment error="unknown type `object`"
object o = 1;
```

**`ref` locals**

```nz fragment error="expected an expression, found Ref"
int x = 5;
ref int r = ref x;
```

**Null-forgiving `x!`**

```nz fragment error="expected Semi, found Bang"
int? a = 5;
int b = a!;
```

**`as`**

```nz error="expected Semi"
package main;

class A { public int X = 0; }

void Main(string[] args) {
    A? a = new A();
    A? b = a as A;
}
```

**`with` expressions**

```nz error="expected Semi"
package main;

record struct P(int X, int Y);

void Main(string[] args) {
    P p = new P(1, 2);
    var q = p with { Y = 5 };
}
```

**`typeof`**

```nz fragment error="unknown name `typeof`"
var t = typeof(int);
```

**Lambdas**

```nz fragment error="expected Semi, found Arrow"
Func<int, int> f = x => x * 2;
```

**`await`**

```nz fragment error="`await` is reserved for a future feature"
int x = 5;
await x;
```

**`throw` expressions**

```nz fragment error="not supported yet: `throw`"
int x = 1;
int y = x > 0 ? x : throw new Exception("bad");
```

**`lock`**

```nz fragment error="`lock` is reserved for a future feature"
lock (args) { }
```

**`throw`**

```nz fragment error="not supported yet: `throw`"
throw new Exception("x");
```

**`yield`**

```nz error="`yield` is reserved for a future feature"
package main;

import std.collections.List;

List<int> Numbers() {
    yield return 1;
}

void Main(string[] args) { }
```

**List patterns**

```nz fragment error="expected a pattern"
int[] a = [1, 2, 3];
bool b = a is [1, .., 3];
```

# Not supported

NZ does not support these constructs:

- Query expressions (LINQ)
- Anonymous types
- Labeled statements and `goto`

The compiler refuses them:

**Labeled statements and `goto`**

```nz fragment error="a label must be followed by a loop"
done: Console.WriteLine("x");
```
