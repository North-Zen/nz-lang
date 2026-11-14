# Syntax Reference

> **Status: Experimental.** NZ is young and its syntax will change. NZ follows C# 14 syntax, except where a difference is listed on this page. Syntax that does not fit the language (for example because of ownership) will be changed, so read this page as a description of today, not a promise.

This page is the reference for the syntax of NZ code (C# style): the lexical structure, the built-in types, variables, expressions, statements and patterns. Declarations (types and their members, generics, interfaces, delegates, attributes, packages and the rest) are in [Syntax Reference: Declarations](Syntax-Declarations.md). The [language guide](../README.md) explains the same features as a tour, with more examples.

Every construct has a short entry with its syntax, a short explanation, a small example, its **status** and a **Differs from C#** line. Every example in an `nz` block is compiled by the real compiler when the tests run, and the examples with an output are also run, so an "Available" entry cannot go stale.

| Status | Meaning |
|---|---|
| Available | works in today's compiler |
| Partial | works, with the gaps the entry lists |
| Planned | decided, not built yet |
| Later | wanted, but deferred until other work is done |
| Not planned | decided against |

When an entry says "as C#", the reference is C# 14. A program on this page is "NZ code (C# style)": it looks like C#, but it is compiled by `nzc`, not by a C# compiler.

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

- **Status:** Available.
- **Differs from C#:** `package` and `import` replace `namespace` and `using` (see [Declarations](Syntax-Declarations.md)). Top-level statements without a `Main` are not supported.

## 1.2 Comments

`// text` runs to the end of the line. `/* text */` can span lines and does not nest: the first `*/` ends it. A run of `///` lines before a declaration is a documentation comment, written in markdown.

```nz fragment output="3\n"
// a line comment
int x = 3; /* a block comment */
/* a block comment
   over several lines */
Console.WriteLine($"{x}");
```

- **Status:** Available.
- **Differs from C#:** documentation comments are markdown, not XML. A `///` run that belongs to no declaration gives a warning.

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

- **Status:** Partial: only ASCII letters; the `@` prefix that turns a keyword into a name (`@class`) is not supported.
- **Differs from C#:** C# allows Unicode letters and `@` names. In NZ the first letter of a function, method, field or property name also decides its visibility (see [Declarations](Syntax-Declarations.md)).

## 1.4 Keywords

The compiler's keyword table, generated from the compiler:

<!-- GEN:keywords -->
NZ has 59 keywords: `import` `package` `void` `public` `private` `class` `struct` `enum` `interface` `var` `new` `this` `if` `else` `while` `for` `foreach` `in` `break` `continue` `return` `switch` `case` `default` `is` `ref` `using` `defer` `unsafe` `true` `false` `null` `abstract` `base` `const` `do` `nameof` `operator` `override` `params` `protected` `readonly` `sealed` `sizeof` `static` `type` `union` `virtual` `where` `async` `await` `move` `lock` `yield` `extern` `volatile` `fixed` `stackalloc` `derive`.
<!-- /GEN -->

Some of them are reserved for features that are planned (`async`, `await`, `lock`, `yield`, `abstract`, `override`, `virtual`, `sealed`, `protected`, `base`) or kept for later (`move`, `volatile`, `derive`): using them as a name gives "`async` is reserved for a future feature".

`checked` and `unchecked` are keywords as in C#: nothing can be named with them.

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

- **Status:** Partial: the C# keywords `try`, `catch`, `finally`, `throw`, `goto`, `typeof`, `as`, `delegate`, `event` and the built-in type names (`int`, `string`, ...) are not refused as names yet.
- **Differs from C#:** `package`, `import`, `defer`, `type`, `union` and `move` are NZ keywords; C# does not have them. `namespace`, `object`, `decimal`, `dynamic`, `goto` and `typeof` are not NZ keywords today.

## 1.5 Integer literals

Decimal `42`, hexadecimal `0xFF`, binary `0b1010`, with `_` between digits (`1_000_000`). The suffixes `U`, `L` and `UL` (any case, also `LU`) choose an unsigned or long type.

Without a suffix, a literal has the first type that holds its value among `int`, `uint`, `long`, `ulong`, as in C#. So `3000000000` is a `uint`. A literal that does not fit `ulong` is an error.

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

- **Status:** Available.
- **Differs from C#:** Same as C#. There is no `M` suffix (no `decimal`).

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

- **Status:** Partial: the `D` and `M` suffixes and a literal that starts with the dot (`.5`) are not supported; write `1.5` and `0.5`.
- **Differs from C#:** no `decimal` literals.

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

- **Status:** Available.
- **Differs from C#:** a code point is always written with braces, `\u{41}`; the C# forms with exactly four or eight digits and no braces, `\x`, `\a`, `\b`, `\f` and `\v` are not supported. A `char` holds any Unicode scalar value, so `'\u{1F600}'` is one `char` (see 2.5).

## 1.8 String literals

`"text"` is a `string`, with the same escapes as characters.

```nz fragment output="tab\there \"q\" \\\n"
string s = "tab\there \"q\" \\";
Console.WriteLine(s);
```

- **Status:** Available.
- **Differs from C#:** a string is UTF-8, not UTF-16 (see 2.6).

## 1.9 Verbatim strings

C#'s `@"..."` (no escapes, `""` for a quote) is not supported; use a raw string (1.10).

```nz fragment error="unexpected character"
string s = @"C:\temp";
```

- **Status:** Later.
- **Differs from C#:** not available.

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

- **Status:** Partial: the opening and closing quotes are always three, so a raw string cannot contain `"""`; interpolated raw strings (`$"""..."""`) are not supported yet.
- **Differs from C#:** C# allows four or more quotes to delimit a raw string.

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

- **Status:** Partial: alignment (`{x,5}`), the formats other than `F`, `D` and `X`, `$@"..."` and `$"""..."""` are not supported yet.
- **Differs from C#:** formatting never depends on the culture (it is always the invariant culture).

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

- **Status:** Available.
- **Differs from C#:** the type is `byte*`, not `ReadOnlySpan<byte>`, and it needs `unsafe`.

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

- **Status:** Available.
- **Differs from C#:** `null` is valid only for `T?`; a plain `string` or class type cannot be `null`.

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

- **Status:** Available.
- **Differs from C#:** C#'s `->` (member access through a pointer) is not supported; `?` after an expression can be NZ's propagation operator.

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

`is` binds at the level of the relational operators. As in C#, the operand of a `switch` expression is a range expression, so a prefix operator or a cast binds tighter than `switch`, and a binary operator binds looser. Operands are evaluated from left to right.

```nz fragment output="14 20 2 True\n"
int a = 2 + 3 * 4;
int b = (2 + 3) * 4;
int c = 10 - 4 - 4;
bool d = 1 < 2 == true;
Console.WriteLine($"{a} {b} {c} {d}");
```

- **Status:** Available.
- **Differs from C#:** Same as C# for the operators NZ has.

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

- **Status:** Available.
- **Differs from C#:** arithmetic is checked by default (2.16).

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

- **Status:** Partial: `sizeof(nint)` and `sizeof(nuint)` are not supported yet.
- **Differs from C#:** a constant above the `int`/`uint` range is refused even on a 64-bit target.

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

- **Status:** Partial: `decimal` is not supported; `float.MaxValue`, `double.MaxValue` and similar constants are not there yet.
- **Differs from C#:** a `double` cast to `float` that is too large stops the program instead of giving an infinity (2.15).

## 2.4 `bool`

`bool` holds `true` or `false`. Its default is `false`. It does not convert to or from a number.

```nz fragment error="cannot cast `bool` to `int`"
bool t = true;
int n = (int)t;
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

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

- **Status:** Available.
- **Differs from C#:** a C# `char` is a 16-bit UTF-16 code unit; an NZ `char` is a whole code point, so `sizeof(char)` is 4 and `char` does not convert to `ushort`. `c += 1` is an error (as in C#); `c++` works.

## 2.6 `string`

A `string` is an immutable sequence of UTF-8 bytes. `Length` counts bytes, and `s[i]` is the `char` that starts at byte `i`. `foreach (char c in s)` gives the characters. `==` compares bytes, and comparisons and case changes are ordinal (they never depend on a culture). A string is copied freely, never moved.

```nz fragment output="6 é True -31\n97\n233\n"
string s = "héllo";
Console.WriteLine($"{s.Length} {s[1]} {s == "héllo"} {"B".CompareTo("a")}");
foreach (char c in "aé") {
    Console.WriteLine($"{(int)c}");
}
```

- **Status:** Available. The default value of a `string` is `""` (2.14).
- **Differs from C#:** UTF-8 instead of UTF-16, so `Length` and positions count bytes; comparisons are ordinal, so `"B".CompareTo("a")` is negative; a `string` is never `null` (use `string?`).

## 2.7 Optional types `T?`

`T?` holds a `T` or `null`, for every type `T`. A `T?` must be checked before its value is used: `is`, `== null`, `!= null` and an early `return` narrow it. `??` gives a fallback. For value types, `.HasValue` and `.Value` work as in C#.

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

- **Status:** Available.
- **Differs from C#:** `T?` is the only way to hold `null`, for classes and strings too, and the compiler refuses a use that is not checked (C# only warns). The details are in [Null Safety](Null-Safety.md).

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

- **Status:** Partial: arrays of arrays (`int[][]`), multi-dimensional arrays (`int[,]`), an optional array type (`int[]?`) and slicing an array (`a[1..3]`) are not supported yet. Fixed-size inline arrays are postponed.
- **Differs from C#:** an array of a class type owns its elements; see [Ownership](Ownership.md).

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

- **Status:** Partial: span locals, fields, return values and `ReadOnlySpan<T>` are not supported yet; span locals and `Span<T> s = stackalloc int[n];` in safe code are planned.
- **Differs from C#:** a span lives for one call; an `in Span<T>` is read-only.

## 2.10 `Index` and `Range`

`^n` is an index counted from the end, and `a..b` is a range (both ends optional, the end excluded). They have the types `Index` and `Range`.

```nz fragment output="4 1..^1\n"
int[] a = [1, 2, 3, 4, 5];
Index i = ^2;
Range r = 1..^1;
Console.WriteLine($"{a[i]} {r}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

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

- **Status:** Partial: names in a tuple expression (`(a: 1, b: 2)`), deconstruction into existing variables (`(x, y) = (y, x)`), nested and typed deconstruction, deconstruction in `foreach`, and `==` on tuples are not supported yet.
- **Differs from C#:** a tuple that holds an owned value is moved as a whole.

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

- **Status:** Partial: `p->m` and function pointers (`delegate*`) are not supported yet. The details are in the guide, section 1.13.
- **Differs from C#:** the ownership rules still apply to managed values inside `unsafe`.

## 2.13 `object` and `dynamic`

NZ has no common base type and no `dynamic`. An owned `object` for class values (with a checked cast back) is planned after class inheritance; `dynamic` is deferred.

```nz fragment error="unknown type `object`"
object o = 1;
```

- **Status:** `object`: Planned. `dynamic`: Later.
- **Differs from C#:** there is no boxing; a value type never converts to `object`.

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

- **Status:** Partial: classes, structs, enums and type parameters have no default yet. The default of `string` is `""`.
- **Differs from C#:** C#'s default of a reference type is `null`, and `default(string)` is `null`; in NZ a non-optional type never holds `null`, so the default of `string` is `""`, as the default of `int` is `0`. An empty `T?` is never turned into a `T` by itself: write `s ?? ""`.

## 2.15 Conversions

**Implicit conversions** exist only where every value of the source fits the target (C#'s table): `sbyte` to `short`, `int`, `long`, `nint`, `float`, `double`; `byte` to every wider integer type and the reals; `int` to `long`, `nint`, `float`, `double`; `uint` to `long`, `ulong`, `nuint`, `float`, `double`; `long` and `ulong` to `float` and `double`; `float` to `double`; and `char` as in 2.5. An integer constant converts to any integer type that holds it (`byte b = 7;`).

When two operands meet, both are converted to one type first: `double` if either is `double`, else `float`, else `ulong`, `long`, `uint` or `int` (C#'s rule). `ulong` with a signed type has no common type.

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

- **Status:** Partial: casts between class types (downcasts), to generic types and user-defined conversion operators are not supported yet; downcasts that throw `InvalidCastException` are planned.
- **Differs from C#:** casts are checked even outside `checked`; `(Name)(x)` is a call of `Name`, not a cast (write the cast of a parenthesised value to a built-in type, or use a variable); a too large `double` cast to `float` stops instead of giving an infinity.

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

- **Status:** Available.
- **Differs from C#:** C# code is unchecked by default (only constants are checked); NZ checks by default. Today an overflow stops the program; it will throw `OverflowException` once exceptions exist.

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

- **Status:** Partial: several declarators in one statement (`int a = 1, b = 2;`) are not supported yet.
- **Differs from C#:** assigning a class value to another variable moves it (3.6).

## 3.2 Definite assignment

Every non-optional local must be initialized where it is declared.

```nz fragment error="must be initialized"
int x;
```

- **Status:** Available.
- **Differs from C#:** stricter: C# allows `int x;` followed by an assignment on every path. In NZ write `int x = ...;`, or use a conditional, a `switch` expression or a local function.

## 3.3 Scope

A local is visible from its declaration to the end of its block. A name cannot be declared again while a local of that name is visible, in the same block or an inner one.

```nz fragment error="already declared"
int x = 1;
{
    int x = 2;
}
```

- **Status:** Available.
- **Differs from C#:** Same as C# (CS0136).

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

- **Status:** Partial: a constant holds an integer type, `bool` or `string`; `float`, `double` and `char` constants are not supported (use `static readonly`).
- **Differs from C#:** C# also allows real and `char` constants.

## 3.5 `ref` locals

`ref var x = ref a[i];` (an alias of a variable or element) and `ref readonly` locals are planned, together with `ref` returns.

```nz fragment error="expected an expression, found Ref"
int x = 5;
ref int r = ref x;
```

- **Status:** Planned.
- **Differs from C#:** not available yet. `var t = list[i];` on a class element is already a borrowed, read-only alias (see [Ownership](Ownership.md)).

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

- **Status:** Available.
- **Differs from C#:** C# copies the reference and keeps both usable; NZ moves. See [Ownership](Ownership.md).

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

- **Status:** Available.
- **Differs from C#:** `base` is not available until class inheritance arrives.

## 4.2 Member access

`x.y` reads a field, property or method of `x`, a static member of a type (`int.MaxValue`), or a member of an enum.

```nz fragment output="3 ABC 2147483647\n"
string s = "abc";
Console.WriteLine($"{s.Length} {s.ToUpper()} {int.MaxValue}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#. `->` is not supported.

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

- **Status:** Available.
- **Differs from C#:** passing a class object by value moves it into the call.

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

- **Status:** Partial: `out` parameters are not supported yet (planned, with `out`-style library members next to the optional forms such as `int? int.TryParse(s)`); `ref` and `in` are not written at the call site.
- **Differs from C#:** C# requires `ref` (and allows `in`) at the call site; NZ refuses it. A `ref` argument cannot share storage with another argument of the same call.

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

- **Status:** Partial: a named argument in its own position followed by positional ones (`F(a: 1, 2)`), and named arguments on library members and on `new`, are not supported yet.
- **Differs from C#:** Same as C# for what is supported.

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

- **Status:** Available.
- **Differs from C#:** Same as C#. With `params T[]` the array moves into the callee.

## 4.7 Element access and indexes from the end

`a[i]` reads or writes an element of an array, a list, a string or a span. `a[^1]` is the last element.

```nz fragment output="3 10 1 l\n"
int[] a = [3, 1];
List<int> l = [1, 2, 10];
int i = 0;
Console.WriteLine($"{a[0]} {l[^1]} {a[^1]} {"hello"[2]}");
```

- **Status:** Available.
- **Differs from C#:** a string index is a byte position (2.6); `a[i]` of a class element is a borrowed alias, not a copy of a reference.

## 4.8 Ranges

`a[x..y]` takes a part. On a `string` it gives a new string; on a span parameter, a span over the same storage.

```nz fragment output="el he llo\n"
string s = "hello";
Console.WriteLine($"{s[1..3]} {s[..2]} {s[^3..]}");
```

- **Status:** Partial: slicing an array (C# gives a new array) is planned; a `List<T>` cannot be sliced (as in C#).
- **Differs from C#:** Same as C# for strings.

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

- **Status:** Partial: target-typed `new()` and collection initializers (`new List<int> { 1, 2 }`) are not supported yet; use a collection expression (4.11).
- **Differs from C#:** the new object has one owner (3.6).

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

- **Status:** Available. The elements of `new string[n]` are `""`, those of `new string?[n]` are `null`.
- **Differs from C#:** the element type needs a default value, so `new Node[3]` for a class `Node` is refused (use `new Node?[3]`); the elements of a `string[]` are `""` where C# has `null`.

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

- **Status:** Available.
- **Differs from C#:** spreading owned elements is refused (they would get a second owner).

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

- **Status:** Partial: assigning through `?.` or `?[]` (`a?[0] = v`, C# 14) is not supported yet.
- **Differs from C#:** Same as C# for reading.

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

- **Status:** Available.
- **Differs from C#:** Same as C#; the left side of `??=` must be optional.

## 4.14 Null-forgiving `x!`

`x!` will be a checked unwrap that throws `NullReferenceException` when `x` is `null`.

```nz fragment error="expected Semi, found Bang"
int? a = 5;
int b = a!;
```

- **Status:** Planned.
- **Differs from C#:** in C# `!` only silences a warning; in NZ it will be checked when the program runs.

## 4.15 Arithmetic operators

`+ - * / %`, unary `+` and `-`. `+` also joins strings. Integer division truncates toward zero.

```nz fragment output="2 -2 1 ab1 2.5\n"
Console.WriteLine($"{5 / 2} {-5 / 2} {7 % 3} {"a" + "b" + 1} {10.0 / 4}");
```

- **Status:** Available.
- **Differs from C#:** integer overflow and narrowing are checked (2.16).

## 4.16 Shift and bitwise operators

`<< >> >>>` shift; `& | ^ ~` work bit by bit on integers, and `& | ^` also on `bool` (both sides evaluated). `>>` keeps the sign of a signed type; `>>>` always shifts in zeros. The count is masked to the width of the type.

```nz fragment output="20 15 4294967055 48 241 15 True 1099511627776\n"
int x = 10;
x = x << 2 >> 1;
uint u = 0xF0;
long l = 1L << 40;
Console.WriteLine($"{x} {u >>> 4} {~u} {u & 0x30} {u | 1} {u ^ 0xFF} {true ^ false} {l}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.17 Comparison and equality

`< > <= >=` compare numbers and `char`; `==` and `!=` compare numbers, `bool`, `char`, strings (by bytes), enums and optionals.

```nz fragment output="True True False True\n"
Console.WriteLine($"{1 < 2} {"a" == "a"} {'a' == 'b'} {1 == 1.0}");
```

- **Status:** Partial: `==` on tuples and on records is not supported yet.
- **Differs from C#:** string comparison is ordinal (it never depends on a culture).

## 4.18 Logical operators

`&&` and `||` evaluate the right side only when needed; `!` negates.

```nz fragment output="False True True\n"
bool a = true;
bool b = false;
Console.WriteLine($"{a && b} {a || b} {!b}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.19 Increment and decrement

`x++`, `x--`, `++x`, `--x` on numbers, `char` and enums.

```nz fragment output="4 2 1\n"
int i = 1;
int a = i++ + ++i;
int b = i-- - --i;
Console.WriteLine($"{a} {b} {i}");
```

- **Status:** Available.
- **Differs from C#:** checked, also on small types.

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

- **Status:** Available.
- **Differs from C#:** assigning a class value moves it; the old value of the target is dropped.

## 4.21 Conditional operator `?:`

`c ? a : b` evaluates one of its arms.

```nz fragment output="yes\n"
int x = 3;
Console.WriteLine(x > 2 ? "yes" : "no");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.22 `is`

`x is T`, `x is T v` and `x is pattern` test a value against a type or a pattern (section 6) and give a `bool`. A variable declared by the pattern is in scope where the test is true.

```nz fragment output="True True 5\n"
int? n = 5;
int x = 7;
Console.WriteLine($"{x is int} {x is > 1 and < 10} {(n is int v ? v : 0)}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.23 `as`

`x as T` will be the safe cast between class and interface types: `null` when `x` is not a `T`.

```nz error="expected Semi"
package main;

class A { public int X = 0; }

void Main(string[] args) {
    A? a = new A();
    A? b = a as A;
}
```

- **Status:** Planned (with class inheritance).
- **Differs from C#:** not available yet.

## 4.24 Cast expressions

`(T)x` converts explicitly; the rules are in 2.15. A cast binds like a prefix operator.

```nz fragment output="66 3\n"
double d = 3.7;
Console.WriteLine($"{(int)'A' + 1} {(int)d}");
```

- **Status:** Partial: see 2.15.
- **Differs from C#:** see 2.15.

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

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.26 `with` expressions

`r with { X = 1 }` (a copy of a record with some members changed) is planned. On a record that owns objects, `with` will consume the source.

```nz error="expected Semi"
package main;

record struct P(int X, int Y);

void Main(string[] args) {
    P p = new P(1, 2);
    var q = p with { Y = 5 };
}
```

- **Status:** Planned.
- **Differs from C#:** a `with` on a record that owns objects moves the source.

## 4.27 `nameof`

`nameof(x)` is the last name of `x` as a `string` constant.

```nz fragment output="count Length\n"
int count = 1;
string s = "a";
Console.WriteLine($"{nameof(count)} {nameof(s.Length)}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 4.28 `typeof`

`typeof(T)` will give a `Type` object (with `Name`), as part of reflection.

```nz fragment error="unknown name `typeof`"
var t = typeof(int);
```

- **Status:** Planned.
- **Differs from C#:** not available yet; the reflection subset will be small.

## 4.29 `sizeof`

`sizeof(T)` is the size in bytes of a built-in numeric type, `bool`, `char`, an enum, or (in `unsafe`) a pointer or a struct.

```nz fragment output="4 8 1 4\n"
Console.WriteLine($"{sizeof(int)} {sizeof(double)} {sizeof(bool)} {sizeof(char)}");
```

- **Status:** Partial: `sizeof(nint)` and `sizeof(nuint)` are not supported yet.
- **Differs from C#:** `sizeof(char)` is 4 (2.5).

## 4.30 `default`

See 2.14.

- **Status:** Partial.
- **Differs from C#:** see 2.14.

## 4.31 `checked` and `unchecked` expressions

`checked(e)` and `unchecked(e)` choose the overflow behaviour of the integer operations written inside `e` (2.16).

```nz fragment output="-2147483648 44\n"
int big = int.MaxValue;
Console.WriteLine($"{unchecked(big + 1)} {unchecked((byte)300)}");
```

- **Status:** Available.
- **Differs from C#:** the default outside is checked; `/`, `%`, real-to-integer casts and integer-to-`char` casts stay checked inside `unchecked`.

## 4.32 Lambdas

`x => x * 2`, `(a, b) => a + b`, `() => { ... }`, with `Func<...>` and `Action<...>` types, are planned. A lambda used during one call will be able to change the variables it captures; a lambda that is stored or returned will capture by copy or move, read-only.

```nz fragment error="expected Semi, found Arrow"
Func<int, int> f = x => x * 2;
```

- **Status:** Planned.
- **Differs from C#:** a stored lambda cannot change what it captured, and an owned object it captures is moved into it.

## 4.33 Anonymous methods

`delegate (int a) { return a; }` and `delegate { ... }` are planned, with the same meaning as the equivalent lambda.

- **Status:** Planned.
- **Differs from C#:** as lambdas (4.32).

## 4.34 `await`

`await e` inside an `async` method is planned, with `Task` and `Task<T>`. `async` and `await` are reserved words today.

```nz fragment error="`await` is reserved for a future feature"
int x = 5;
await x;
```

- **Status:** Planned.
- **Differs from C#:** async code will run on one main loop, in a fixed order; only `Task.Run` will use other threads. `async void` will be allowed only for event handlers.

## 4.35 `throw` expressions

`x ?? throw new E(...)` and `c ? a : throw ...` are planned with exceptions.

```nz fragment error="expected Semi, found New"
int x = 1;
int y = x > 0 ? x : throw new Exception("bad");
```

- **Status:** Planned.
- **Differs from C#:** `throw` moves the exception object; `throw null` is refused.

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

- **Status:** Partial: `Span<T> s = stackalloc T[n];` in safe code is planned.
- **Differs from C#:** C# allows the span form outside `unsafe` today.

## 4.37 Error propagation `?` (NZ only)

`e?` after an expression of type `Result<T, E>` gives the `Ok` value, or returns the `Err` from the function at once. On a `T?` in a function that returns an optional, it returns `null`. It is NZ's own operator; C# has no equivalent.

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

- **Status:** Available.
- **Differs from C#:** not in C#. When the next token can start an expression, `?` is the conditional operator: write `(f()?) - 1`. More in [Error Handling](Error-Handling.md).

## 4.38 Query expressions (LINQ)

`from x in xs where ... select ...` is not part of NZ.

- **Status:** Not planned.
- **Differs from C#:** not available; write a loop.

## 4.39 Anonymous types

`new { A = 1 }` is not provided; tuples cover the use.

- **Status:** Not planned.
- **Differs from C#:** not available.

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

- **Status:** Partial: the empty statement `;` is not supported yet.
- **Differs from C#:** an opening `{` stays on the same line as its statement (a style rule of NZ).

## 5.2 Declaration and expression statements

A declaration (3.1, 3.4), a local function (5.18) or an expression with an effect (an assignment, a call, `++`, `--`, `new`, `e?`) followed by `;`.

```nz fragment output="2\n"
int x = 1;
x++;
Console.WriteLine($"{x}");
```

- **Status:** Available.
- **Differs from C#:** `F()?;` is allowed as a statement (the value is discarded).

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

- **Status:** Available.
- **Differs from C#:** Same as C#. A test such as `x != null` narrows an optional inside the branch.

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

- **Status:** Partial: `goto case`, `goto default` and a pattern variable in a section with several labels are not supported yet.
- **Differs from C#:** Same as C# for what is supported.

## 5.5 `while`

```nz fragment output="3\n"
int j = 0;
while (j < 3) {
    j++;
}
Console.WriteLine($"{j}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 5.6 `do ... while`

```nz fragment output="2\n"
int k = 0;
do {
    k++;
} while (k < 2);
Console.WriteLine($"{k}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

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

- **Status:** Partial: several declarators or several step expressions (`for (int i = 0, j = 5; ...; i++, j--)`) are not supported yet.
- **Differs from C#:** Same as C# for what is supported.

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

- **Status:** Partial: deconstruction in the loop variable (`foreach (var (a, b) in pairs)`), `await foreach` and walking a `Stack<T>` or `Queue<T>` are not supported yet; `foreach` over user types that implement `IEnumerable<T>` comes with iterators.
- **Differs from C#:** the loop variable of a class element is a borrowed alias; the collection is frozen while it is walked.

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

- **Status:** Available.
- **Differs from C#:** labeled `break` and `continue` are NZ's; C# uses `goto` for that.

## 5.10 Labeled statements and `goto`

A label (`name:`) can only stand before a loop, for labeled `break` and `continue` (5.9). `goto label;` is not part of NZ; `goto case` and `goto default` in a `switch` may come later.

```nz fragment error="a label must be followed by a loop"
done: Console.WriteLine("x");
```

- **Status:** `goto` and general labels: Not planned. `goto case` and `goto default`: Later.
- **Differs from C#:** no `goto`.

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

- **Status:** Available.
- **Differs from C#:** Same as C#; an early `return` narrows an optional after the test.

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

- **Status:** Partial: the `using var x = e;` declaration and the `IDisposable` rules (C# 14: an error unless the type is `IDisposable`; `Dispose` runs once when the value is dropped) are planned; `await using` is planned with async.
- **Differs from C#:** the block owns the value, so `using (x) { }` with an existing local moves `x` into the block.

## 5.13 `lock`

`lock (m) { ... }` on a `Mutex<T>` that owns its data is planned; inside the block the data can be changed. `lock (this)` on an arbitrary object will be refused.

```nz fragment error="`lock` is reserved for a future feature"
lock (args) { }
```

- **Status:** Planned.
- **Differs from C#:** a lock owns the data it protects (C# locks any object).

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

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 5.15 `defer` (NZ only)

`defer statement;` or `defer { ... }` runs when the enclosing block ends, in reverse order of the `defer` statements, on every way out of the block.

```nz fragment output="body\nsecond\nfirst\n"
{
    defer Console.WriteLine("first");
    defer Console.WriteLine("second");
    Console.WriteLine("body");
}
```

- **Status:** Partial: `return`, `break` and `continue` that would leave a deferred statement are not supported yet; a `defer` cannot declare a variable.
- **Differs from C#:** not in C# (it comes from Go and Zig); `try`/`finally` is C#'s nearest form.

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

- **Status:** Available.
- **Differs from C#:** a mutable static is only usable inside `unsafe`; types, null safety, overflow checks and the ownership of managed values stay checked inside `unsafe`.

## 5.17 `checked` and `unchecked` blocks

`checked { ... }` and `unchecked { ... }` set the overflow behaviour of the statements inside (2.16).

```nz fragment output="255\n"
byte b = 0;
unchecked {
    b--;
}
Console.WriteLine($"{b}");
```

- **Status:** Available.
- **Differs from C#:** the default outside is checked.

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

- **Status:** Partial: generic and `static` local functions and attributes on local functions are not supported yet.
- **Differs from C#:** a local function cannot change the variables it captures; an uppercase name gives a warning (lowercase means "not public").

## 5.19 `try`, `catch` and `finally`

`try { ... } catch (E e) when (...) { ... } finally { ... }` is planned with C# syntax. Library members will throw .NET's exception types (`FileNotFoundException`, `FormatException`, ...), and run-time checks (index out of range, overflow, a failed `Assert`) will throw too. The no-throw way stays available: `Try`-prefixed members and `Result` with `?`.

```nz fragment error="expected Semi, found LBrace"
try {
    Console.WriteLine("a");
} finally {
    Console.WriteLine("b");
}
```

- **Status:** Planned.
- **Differs from C#:** an exception object is owned: `throw` moves it and `catch` owns it; an exception nothing catches stops the program with `nz: <Type>: <message>` without running `finally` blocks. Under `--no-runtime`, only `try`/`finally` will be allowed.

## 5.20 `throw`

`throw e;` and `throw;` (rethrow inside `catch`) are planned with exceptions (5.19). Today `Abort("message")` stops the program and `Assert(condition)` checks a condition.

```nz fragment error="expected Semi, found New"
throw new Exception("x");
```

- **Status:** Planned.
- **Differs from C#:** see 5.19.

## 5.21 `yield`

`yield return e;` and `yield break;` in iterator methods that return `IEnumerable<T>` are planned. Async iterators come later.

```nz error="`yield` is reserved for a future feature"
package main;

import std.collections.List;

List<int> Numbers() {
    yield return 1;
}

void Main(string[] args) { }
```

- **Status:** Planned.
- **Differs from C#:** a value from a `yield return` iterator is moved to the loop variable.

## 5.22 Deconstruction statement

`var (a, b) = e;` splits a tuple or a positional record into new variables; `_` discards an element.

```nz fragment output="1 a\n"
var (n, s) = (1, "a");
Console.WriteLine($"{n} {s}");
```

- **Status:** Partial: see 2.11.
- **Differs from C#:** see 2.11.

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

- **Status:** Available.
- **Differs from C#:** a binding of an owned value borrows it; it does not copy a reference. Tests against base classes come with class inheritance.

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

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 6.3 Relational patterns

`< v`, `<= v`, `> v`, `>= v` on integers, `char`, `float`, `double` and enums.

```nz fragment output="True\n"
char ch = 'x';
Console.WriteLine($"{ch is >= 'a' and <= 'z'}");
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

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

- **Status:** Available.
- **Differs from C#:** a variable cannot be declared under `not` (`x is not T v`); test the positive form instead.

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

- **Status:** Partial: extended property patterns (`{ A.X: 0 }`) and property patterns on built-in types and handles are not supported yet.
- **Differs from C#:** Same as C# for what is supported.

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

- **Status:** Available.
- **Differs from C#:** works on tuples and positional record structs; a user `Deconstruct` method is not used.

## 6.7 `var` pattern

`var x` matches any value and binds it.

```nz fragment output="one 2\n"
var t = (1, 2);
string s = t switch { (1, var y) => $"one {y}", _ => "no" };
Console.WriteLine(s);
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 6.8 Discard pattern

`_` matches anything without binding.

```nz fragment output="other\n"
int x = 9;
Console.WriteLine(x switch { 1 => "one", _ => "other" });
```

- **Status:** Available.
- **Differs from C#:** Same as C#.

## 6.9 List patterns

`[1, .., 3]` (C# 11) is not supported yet; test `Length` and the elements instead.

```nz fragment error="expected a pattern"
int[] a = [1, 2, 3];
bool b = a is [1, .., 3];
```

- **Status:** Later.
- **Differs from C#:** not available yet.

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

- **Status:** Available.
- **Differs from C#:** Same as C#.

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

- **Status:** Available.
- **Differs from C#:** Same as C# (a warning, as CS8509, and an error for an arm already covered, as CS8510). String and floating-point constants never complete a `switch`, so such a `switch` needs `_`.

# 7. Index

| Construct | Section | Status |
|---|---|---|
| Source files | 1.1 | Available |
| Comments, documentation comments | 1.2 | Available |
| Identifiers | 1.3 | Partial |
| Keywords | 1.4 | Partial |
| Integer literals | 1.5 | Available |
| Real literals | 1.6 | Partial |
| Character literals | 1.7 | Available |
| String literals | 1.8 | Available |
| Verbatim strings `@"..."` | 1.9 | Later |
| Raw string literals | 1.10 | Partial |
| Interpolated strings | 1.11 | Partial |
| UTF-8 literals `u8` | 1.12 | Available |
| `true`, `false`, `null` | 1.13 | Available |
| Operators and punctuators | 1.14 | Available |
| Precedence and associativity | 1.15 | Available |
| Integer types | 2.1 | Available |
| `nint`, `nuint` | 2.2 | Partial |
| `float`, `double` | 2.3 | Partial |
| `decimal` | 2.3 | Later |
| `bool` | 2.4 | Available |
| `char` | 2.5 | Available |
| `string` | 2.6 | Partial |
| Optional types `T?` | 2.7 | Available |
| Arrays | 2.8 | Partial |
| Spans | 2.9 | Partial |
| `Index` and `Range` | 2.10 | Available |
| Tuples | 2.11 | Partial |
| Pointers | 2.12 | Partial |
| `object` | 2.13 | Planned |
| `dynamic` | 2.13 | Later |
| Default values | 2.14 | Partial |
| Conversions and casts | 2.15 | Partial |
| Checked arithmetic | 2.16 | Available |
| Local variables and `var` | 3.1 | Partial |
| Definite assignment | 3.2 | Available |
| Scope | 3.3 | Available |
| `const` | 3.4 | Partial |
| `ref` locals | 3.5 | Planned |
| Moves | 3.6 | Available |
| Simple names, parentheses, `this` | 4.1 | Available |
| Member access | 4.2 | Available |
| Invocation | 4.3 | Available |
| `ref`, `in`, `out` arguments | 4.4 | Partial |
| Named and default arguments | 4.5 | Partial |
| `params` | 4.6 | Available |
| Element access, `^` | 4.7 | Available |
| Ranges `..` | 4.8 | Partial |
| `new`, object initializers | 4.9 | Partial |
| Array creation | 4.10 | Partial |
| Collection expressions, spread | 4.11 | Available |
| `?.`, `?[]` | 4.12 | Partial |
| `??`, `??=` | 4.13 | Available |
| Null-forgiving `x!` | 4.14 | Planned |
| Arithmetic operators | 4.15 | Available |
| Shift and bitwise operators | 4.16 | Available |
| Comparison and equality | 4.17 | Partial |
| Logical operators | 4.18 | Available |
| Increment and decrement | 4.19 | Available |
| Assignment, compound assignment | 4.20 | Available |
| Conditional `?:` | 4.21 | Available |
| `is` | 4.22 | Available |
| `as` | 4.23 | Planned |
| Cast expressions | 4.24 | Partial |
| `switch` expressions | 4.25 | Available |
| `with` | 4.26 | Planned |
| `nameof` | 4.27 | Available |
| `typeof` | 4.28 | Planned |
| `sizeof` | 4.29 | Partial |
| `default` | 4.30 | Partial |
| `checked`, `unchecked` expressions | 4.31 | Available |
| Lambdas | 4.32 | Planned |
| Anonymous methods | 4.33 | Planned |
| `await` | 4.34 | Planned |
| `throw` expressions | 4.35 | Planned |
| `stackalloc` | 4.36 | Partial |
| Error propagation `?` | 4.37 | Available |
| Query expressions (LINQ) | 4.38 | Not planned |
| Anonymous types | 4.39 | Not planned |
| Blocks, empty statement | 5.1 | Partial |
| Declaration and expression statements | 5.2 | Available |
| `if`, `else` | 5.3 | Available |
| `switch` statement | 5.4 | Partial |
| `while` | 5.5 | Available |
| `do ... while` | 5.6 | Available |
| `for` | 5.7 | Partial |
| `foreach` | 5.8 | Partial |
| `break`, `continue`, labels | 5.9 | Available |
| `goto` | 5.10 | Not planned |
| `goto case`, `goto default` | 5.10 | Later |
| `return` | 5.11 | Available |
| `using` | 5.12 | Partial |
| `lock` | 5.13 | Planned |
| `fixed` | 5.14 | Available |
| `defer` | 5.15 | Partial |
| `unsafe` | 5.16 | Available |
| `checked`, `unchecked` blocks | 5.17 | Available |
| Local functions | 5.18 | Partial |
| `try`, `catch`, `finally` | 5.19 | Planned |
| `throw` | 5.20 | Planned |
| `yield` | 5.21 | Planned |
| Deconstruction statement | 5.22 | Partial |
| Type and declaration patterns | 6.1 | Available |
| Constant patterns | 6.2 | Available |
| Relational patterns | 6.3 | Available |
| `not`, `and`, `or` | 6.4 | Available |
| Property patterns | 6.5 | Partial |
| Positional patterns | 6.6 | Available |
| `var` pattern | 6.7 | Available |
| Discard `_` | 6.8 | Available |
| List patterns | 6.9 | Later |
| `when` guards | 6.10 | Available |
| Exhaustiveness | 6.11 | Available |
