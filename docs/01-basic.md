# 1. Basic

> NZ is still in development and is not stable. Anything in this documentation can change.

## 1.1 Data Types

NZ has primitive types for numbers, characters and booleans, plus `string`.

### 1.1.1 Numbers

- **Integer**: ten types, listed below
- **Float**: `float`, `double`

| Type | Size | Range | C type |
|---|---|---|---|
| `sbyte` | 8 bits | -128 to 127 | `int8_t` |
| `byte` | 8 bits | 0 to 255 | `uint8_t` |
| `short` | 16 bits | -32768 to 32767 | `int16_t` |
| `ushort` | 16 bits | 0 to 65535 | `uint16_t` |
| `int` | 32 bits | -2147483648 to 2147483647 | `int32_t` |
| `uint` | 32 bits | 0 to 4294967295 | `uint32_t` |
| `long` | 64 bits | -9223372036854775808 to 9223372036854775807 | `int64_t` |
| `ulong` | 64 bits | 0 to 18446744073709551615 | `uint64_t` |
| `nint` | pointer size | the range of `intptr_t` | `intptr_t` |
| `nuint` | pointer size | the range of `uintptr_t` | `uintptr_t` |

A value of any of them is copied, never moved. `nint` and `nuint` have the width of a pointer, so their range depends on the target; a constant converted or cast to them is checked against the `int` or `uint` range, so that a program means the same on every target. `sizeof(nint)` is not supported yet (see "Not yet" in [Appendix B](appendix-b-current-behavior.md#appendix-b-current-behavior-notes)).

```nz fragment output="-128 255 -32768 65535 4294967295 -9223372036854775808 18446744073709551615 5 7\n"
sbyte a = -128;
byte b = 255;
short c = -32768;
ushort d = 65535;
uint e = 4294967295;
long f = -9223372036854775808;
ulong g = 18446744073709551615;
nint h = 5;
nuint i = 7;
Console.WriteLine($"{a} {b} {c} {d} {e} {f} {g} {h} {i}");
```

An integer literal has the first of the types `int`, `uint`, `long`, `ulong` that holds its value. A real literal (one with a fraction, an exponent or the `F` suffix) has type `double`, and `float` when it ends in `F`.

**Arithmetic is checked.** `+ - *` and unary `-` on every integer type stop the program with ``nz: integer overflow`` when the result does not fit the promoted type, `/` and `%` by zero stop with ``nz: division by zero``, the smallest value divided by (or taken modulo) -1 stops with ``nz: integer overflow``, and a conversion back into a narrower type (`b += 10`, `b++`, a cast) stops with ``nz: value does not fit in `byte` `` naming the target. Shifts never fail: the count is masked to the width of the promoted type, and bits shifted out are lost (`3000000000 << 1` is 1705032704). A compound shift on a smaller type is a compound assignment like the others: `b <<= 1` on a `byte` shifts the promoted `int`, then narrows the result back to `byte`, checked, so `byte b = 128; b <<= 1;` stops with ``nz: value does not fit in `byte` `` (inside `unchecked` it keeps the low bits).

**Constant overflow is a compile-time error.** A constant expression whose value does not fit its type is refused before the program runs: `2147483647 + 1`, `100000 * 100000`, a constant `Big + 1` and `4294967295 + 1` (a `uint`) are errors, while `2147483647 + 1L` is a `long` and `unchecked(2147483647 + 1)` wraps. Dividing a constant by a constant zero is an error too.

```nz fragment error="constant expression overflows `int`"
int a = 2147483647 + 1;
```

```nz fragment output="-2147483648 2147483648\n"
int a = unchecked(2147483647 + 1);
long b = 2147483647 + 1L;
Console.WriteLine($"{a} {b}");
```

**Integer literals.** Besides decimal there is hexadecimal (`0xFF`, also `0X`) and binary (`0b1010`, also `0B`). A leading zero changes nothing: `007` is the decimal number 7. A `_` may stand between two digits of the literal, as many as you like: `1_000_000`, `0xFFFF_FFFF`, `0b1111_0000`. It may not come first, last, right after `0x` or `0b`, or before a suffix.

The type of an integer literal is the first type of a list that holds the value, and a suffix chooses the list. Without a suffix the list is `int`, `uint`, `long`, `ulong`; with `U` it is `uint`, `ulong`; with `L` it is `long`, `ulong`; `UL` (also written `LU`) is always `ulong`. The letters may be upper or lower case (`1L`, `1l`, `1UL`, `1ul`, `1uL`, `1lu`). The rule is the same in every base: `0x7FFFFFFF` is an `int`, `2147483648` and `0xFFFFFFFF` are `uint` values (4294967295, not -1), `4294967296` is a `long` and `9223372036854775808` a `ulong`. A value above the largest `ulong` is an error. `-2147483648` is the smallest `int` and `-9223372036854775808` the smallest `long`: the minus belongs to the literal there. Arithmetic on such a literal runs in its type and is checked like any other: `3000000000 + 1` is the `uint` 3000000001, `4294967295 + 1` is a compile-time error (a constant that overflows, see below), `uint big = 4294967295; big + 1` aborts with an overflow when it runs, and `4294967295L + 1` is `long` arithmetic. Shifts are not checked: `3000000000 << 1` is the `uint` 1705032704.

```nz fragment output="255 10 1000000 4294967295 10\n"
int hex = 0xFF;
int bin = 0b1010;
int million = 1_000_000;
long mask = 0xFFFFFFFF;
long ten = 10L;
Console.WriteLine($"{hex} {bin} {million} {mask} {ten}");
```

A `U` literal is a `uint`, or a `ulong` when it is larger; a `UL` literal is a `ulong`:

```nz fragment output="4294967295 18446744073709551615\n"
var u = 4294967295U;
ulong big = 18446744073709551615UL;
Console.WriteLine($"{u} {big}");
```

Letters or digits glued to a number that are not a suffix are an error on the literal, not the start of a name (`12abc`, `0x1G`); `_1` is still a name.

```nz fragment error="invalid suffix"
int a = 12abc;
```

```nz fragment error="cannot assign `ulong` to `long`"
long z = 9223372036854775808;
```

```nz fragment error="cannot assign `uint` to `int`"
int x = 0xFFFFFFFF;
```

```nz fragment output="-2147483648 -9223372036854775808 3000000000 3000000001\n"
int min = -2147483648;
long lmin = -9223372036854775808;
var u = 3000000000;
Console.WriteLine($"{min} {lmin} {u} {3000000000L + 1}");
```

```nz fragment output="3000000001 1705032704\n"
var u = 3000000000;
var next = u + 1;
Console.WriteLine($"{next} {u << 1}");
```

```nz fragment error="must stand between two digits"
int a = 1_;
```

**Real literals.** A real has digits, optionally a `.` followed by digits, optionally an exponent (`e` or `E`, an optional sign, digits) and optionally the suffix `F` or `f`. `1.5`, `2.5e-3` and `1e5` are `double`; with `F` they are `float`, also after whole digits (`1F` is the `float` 1). A `_` may stand between digits of each part (`1_000.5`, `1e1_0`). There is no `D` or `M` suffix and no hexadecimal real. A value too large for its type is an error; a value too small rounds to zero without one.

```nz fragment output="1.5 0.0025 100000 1 3\n"
float f = 1.5F;
double d = 2.5e-3;
double big = 1e5;
float whole = 1F;
Console.WriteLine($"{f} {d} {big} {whole} {f * 2F}");
```

Because a real literal without `F` has type `double`, this is rejected, and the message says how to write a `float` literal:

```nz fragment error="write `1.5F`"
float f = 1.5;
```

The `F` form is accepted:

```nz fragment output="1.5\n"
float f = 1.5F;
Console.WriteLine($"{f}");
```

```nz fragment error="real literal out of range for `float`"
float f = 3.5e38F;
```

### 1.1.2 Characters

```nz fragment
char letter = 'A';
bool active = true;
string name = "NZ";
```

**Escapes.** One table serves characters, strings and the text of interpolated strings: `\n` `\t` `\r` `\0` `\\` `\"` `\'`, and `\u{...}` with 1 to 6 hexadecimal digits (either case, leading zeros count) naming a Unicode scalar value. A `\u` without braces, a surrogate such as `\u{D800}` and a value above `10FFFF` are errors, and so is any other escape, in an interpolated string too (`$"\q"`).

```nz fragment output="HI\nAB C\na\tb \\ \"q\"\n"
Console.WriteLine("\u{48}\u{49}");
Console.WriteLine($"{'\u{41}'}{"\u{42}"} \u{43}");
Console.WriteLine("a\tb \\ \"q\"");
```

```nz fragment error="unknown escape sequence"
string a = "\q";
```

```nz fragment error="unknown escape sequence"
string b = $"a\q{1}";
```

```nz fragment error="is not a Unicode scalar value"
string d = "\u{D800}";
```

#### Strings

A `string` is an immutable sequence of UTF-8 bytes (`Length` is the number of bytes). Two limits apply: a string the language operators make (`+`, `+=`, an interpolation) can be up to 2,147,483,647 bytes, and a longer one stops the program with `nz: string too long`; a string a library member makes (`Substring`, `Replace`, `PadLeft`, `string.Join`, `StringBuilder.ToString` and the rest) can be up to 1,073,741,791 bytes, and a longer one throws `OutOfMemoryException`. A string is copied freely: assigning it, passing it, storing it in a field, a list or a tuple never moves it, and two copies compare equal by their bytes. A computed string is freed when its last holder goes; a literal is never freed (see [Ownership: Strings](+/Ownership.md#strings)).

**Raw strings.** A string that starts with `"""` is raw: it ends at the next `"""`, has no escapes, and may hold quotes and backslashes (so it cannot contain `"""` or end with a `"`). Written on one line, its content is kept exactly. Written over several lines, the line break right after the opening `"""` is dropped; when the closing `"""` stands alone on its line, that last line break and the spaces and tabs before the `"""` are dropped too, and the same indentation is removed from the start of every line (a blank line becomes empty; a line indented less than the closing line is an error). Line breaks `\r\n` count as `\n`, so a file gives the same string whether it was saved with LF or CRLF. If the closing `"""` follows the last character of the text, nothing is removed.

```nz fragment output="C:\\temp\\new\nfirst\n  second\n11 14\n[a\"b]\n"
string path = """C:\temp\new""";
string text = """
    first
      second
    """;
Console.WriteLine(path);
Console.WriteLine(text);
Console.WriteLine($"{path.Length} {text.Length}");
Console.WriteLine($"[{"""a"b"""}]");
```

Text on the same line as the opening `"""` of a multi-line raw string is kept as written and is not checked against the indentation. (This rule is provisional and may become stricter before the first release.)

```nz fragment output="[ab\nc]\n[  a\n  b]\n"
string open = """ab
  c
  """;
string inline = """
  a
  b""";
Console.WriteLine($"[{open}]");
Console.WriteLine($"[{inline}]");
```

```nz fragment error="indented less than its closing"
string text = """
    first
  second
    """;
```

```nz fragment error="unterminated raw string literal"
string s = """never closed;
```

An interpolated raw string (`$"""..."""`) is not supported yet.

```nz fragment error="interpolated raw strings are not supported yet"
string s = $"""x""";
```

**String interpolation** uses `$"..."` with expressions inside `{ }`:

```nz fragment output="Hello, NZ v1\n"
string name = "NZ";
int version = 1;
Console.WriteLine($"Hello, {name} v{version}");
```

Write `{{` and `}}` for a literal brace. Inside a hole, strings, raw strings, characters and nested interpolated strings are read whole, so a `:`, `{` or `}` inside them never ends the hole:

```nz fragment output="big [a:b] [:] [x}y] []\n{n} = 5\n"
int n = 5;
int? none = null;
Console.WriteLine($"{(n > 3 ? "big" : "small")} [{"a:b"}] [{':'}] [{"""x}y"""}] [{none:D2}]");
Console.WriteLine($"{{n}} = {n}");
```

**Formats.** A hole may end with `:` and a format: a letter, then an optional precision from 0 to 99, with no spaces.

| Format | Takes | Prints | Default precision |
|---|---|---|---|
| `F` or `f` | any integer type, `float`, `double` | fixed point with that many digits after the point | 2 |
| `D` or `d` | any integer type | at least that many digits, zero-padded after the sign | 0 |
| `X` | any integer type | hexadecimal in upper case, at least that many digits | 0 |
| `x` | any integer type | the same in lower case | 0 |

`F` rounds the exact binary value of the number to the nearest digit and, on an exact tie, to the even one: `{0.125:F2}` is `0.12`, `{2.5:F0}` is `2`, and `{2.675:F2}` is `2.67` because the stored value is just below the tie. A negative value that rounds to zero keeps its sign (`-0.00`). A `float` is widened to `double` first, so `{0.1F:F10}` is `0.1000000015`. NaN and infinities print as without a format and ignore the precision. `F` on an integer is exact (`{5:F2}` is `5.00`). `X` shows an integer in the width of its type: `-1` is two `F`s as an `sbyte`, eight as an `int` and sixteen as a `long`. On a `T?` of these types a null prints nothing, as it does without a format.

```nz fragment output="3.14 3 005 FF 00ff FFFFFFFF -005\n0.12 2.67 2 -0.00 5.0\n"
double x = 3.14159;
int n = 5;
int m = 255;
Console.WriteLine($"{x:F2} {x:F0} {n:D3} {m:X} {m:x4} {-1:X} {-5:D3}");
Console.WriteLine($"{0.125:F2} {2.675:F2} {2.5:F0} {-0.001:F2} {n:F1}");
```

All ten integer types take `F`, `D` and `X`; an unsigned value is never shown as negative, and `nint` and `nuint` use the width of a pointer (`-1` as a `nint` is sixteen `F`s on a 64-bit target). An enum or a `char` takes no format: cast to an integer first.

```nz fragment output="FF FF 07 1234567 0001234567 255.00 7.0 18446744073709551615\n"
uint u = 255;
sbyte s = -1;
long l = 1234567;
byte b = 7;
Console.WriteLine($"{u:X} {s:X} {b:X2} {l:D} {l:D10} {u:F2} {b:F1} {(ulong)18446744073709551615:D}");
```

A format must fit the type, and it is the only place a `:` is read as a format:

```nz fragment error="format `D` needs an integer, found `double`"
string s = $"{1.5:D2}";
```

```nz fragment error="unknown format `Q3`"
int n = 5;
string s = $"{n:Q3}";
```

```nz fragment error="a format precision is 0 to 99"
int n = 5;
string s = $"{n:F100}";
```

Because `:` starts a format, a conditional `? :` directly in a hole must be put in parentheses:

```nz fragment error="must be in parentheses"
int n = 5;
string s = $"{n > 3 ? "a" : "b"}";
```

The `+` operator on a string accepts any value on its right-hand side:

```nz fragment
string text = "total = " + 3;
```

### 1.1.3 Boolean

A `bool` prints as `True` or `False`, in `Console.WriteLine` and in an interpolation alike.

### 1.1.4 Pointer

Taking addresses, `fixed`, `stackalloc` and manual memory are in [15.6 Unsafe memory](15-memory-management.md#156-unsafe-memory).

A pointer type is written: `int*`, `void*`, `Point**`. The type it points at must be **unmanaged**: an integer type, `float`, `double`, `bool`, `char`, an enum, a pointer, or a struct whose fields are all unmanaged. A pointer to a `string`, a class, an array, a `List<T>`, a type parameter, an optional or a tuple does not exist, and `int?` and tuples are not unmanaged either.

The operators are `&x`, `*p`, `p->F`, `p[i]`, `p + n`, `p - n`, `p - q` (a `long`), `p++`, `p--`, and the comparisons. `null` converts to every pointer type, and `T*` converts to `void*` without a cast. Casts are `(T*)p`, `(void*)p`, `(nint)p`, `(nuint)p`, and `(T*)n` from `nint`, `nuint`, `long` or `ulong`. Inside `unsafe`, `sizeof(T)` works for every unmanaged type.

Pointer arithmetic is not checked, `p[i]` has no bounds check, and nothing checks a dereference of `null`. A pointer that outlives what it points at, a write past the end of a buffer, a misaligned read through a cast, a double `Free`: the compiler does not see these, and the program that writes `unsafe` vouches for them. Tests find them with the sanitizers (`NZ_CC` with `-fsanitize=address,undefined`).

A pointer never leaves `unsafe`. A local, parameter, result or field of pointer type needs an unsafe context; a struct or class with a pointer field is an `unsafe struct` or `unsafe class`; a function with a pointer in its signature is called only from `unsafe`. A pointer cannot be a type argument, an array element, a tuple element or optional. The one way out is an explicit `(nint)` or `(nuint)` cast, which is a plain number in safe code.

```nz error="can only be used inside `unsafe`"
package main;

void Main(string[] args) {
    int x = 1;
    int* p = &x;
}
```

### 1.1.5 Empty data type (`void`)

This section is not written yet.

Outline:

- `void` as a return type
- `void*`

### 1.1.6 Aliases

`type Name = Type;` gives a type another name. An alias is transparent: it is expanded wherever a type is written, and messages show the type it stands for. An alias can have type parameters, and an alias of an enum or class works as `Alias.Member`.

```nz output="7 3\n"
package main;

import std.io.Console;
import std.collections.List;

type Id = int;
type Ids = List<Id>;
type Pair<T> = (T, T);

void Main(string[] args) {
    Ids xs = new List<int>();
    xs.Add(7);
    Pair<int> p = (1, 2);
    Console.WriteLine($"{xs[0]} {p.Item1 + p.Item2}");
}
```

An alias that refers to itself is an error, and so is an alias that would make a shape the type syntax cannot write (an optional of an optional, an optional array, an array of arrays):

```nz error="alias `A` refers to itself"
package main;

type A = B;
type B = A;

void Main(string[] args) {
}
```

### 1.1.7 Type checking

This section is not written yet.

Outline:

- `is` type tests
- `typeof` and `GetType()`

## 1.2 Conversion, type checks and casting

### Implicit conversions

Implicit conversions only *widen*, never narrow, and only where every value of the source fits the target. A source converts to these types:

| From | Implicitly to |
|---|---|
| `sbyte` | `short` `int` `long` `nint` `float` `double` |
| `byte` | `short` `ushort` `int` `uint` `long` `ulong` `nint` `nuint` `float` `double` |
| `short` | `int` `long` `nint` `float` `double` |
| `ushort` | `int` `uint` `long` `ulong` `nint` `nuint` `float` `double` |
| `int` | `long` `nint` `float` `double` |
| `uint` | `long` `ulong` `nuint` `float` `double` |
| `long`, `ulong` | `float` `double` |
| `nint` | `long` `float` `double` |
| `nuint` | `ulong` `float` `double` |
| `float` | `double` |

The same value may differ after a conversion to a real: a `long`, `ulong`, `nint` or `nuint` that does not fit the 24 or 53 bits of a `float` or `double` is rounded. Nothing else converts: `short` to `ushort`, `int` to `uint` and `ulong` to `long` are errors, in both directions.

```nz fragment output="200 200 200 200 200\n"
byte b = 200;
ushort us = b;
uint u = us;
ulong ul = u;
float f = ul;
Console.WriteLine($"{b} {us} {u} {ul} {f}");
```

```nz fragment error="cannot assign `int` to `uint`"
int i = -1;
uint u = i;
```

**Constants take their type from context.** An integer constant (a literal, a named constant, or an expression of constants) converts to any integer type that holds its value, wherever a value is assigned: a variable, a field, an argument, a return value, an element. `byte b = 200;` and `byte c = 100 + 100;` work; `byte b = 300;` and `sbyte s = -129;` are errors, and so is a variable: `int i = 5; byte b = i;`. A `long` constant converts only to `ulong`.

```nz fragment output="200 200 -128 65535 7\n"
byte a = 200;
byte b = 100 + 100;
sbyte c = -128;
ushort d = 65535;
byte? e = 7;
Console.WriteLine($"{a} {b} {c} {d} {e}");
```

```nz fragment error="cannot assign `int` to `byte`"
byte b = 300;
```

**Binary numeric promotion.** An arithmetic, bitwise or comparison operator promotes both operands to one type: if one is `double` the result is `double`, else if one is `float` it is `float`; else `ulong` if one is `ulong`; else `long` if one is `long`; else `uint` when both are unsigned among `byte`, `ushort`, `uint`; `uint` with a signed `sbyte`, `short` or `int` gives `long`; everything else is `int`. So `byte + byte` is an `int` (assigning it to a `byte` is an error; cast it), and `byte * 2.5` is a `double`. `nint` and `nuint` follow the same pattern at pointer width: `nint + int` is `nint`, `nuint + uint` is `nuint`, `nint + uint` is `long`. A `ulong` cannot be mixed with a signed type, and neither can `nuint`: there is no integer type that holds both, and the error says so. An integer constant that fits the other operand joins it (`ul + 5` is `ulong`, `u + 5` is `uint`).

```nz fragment output="2 3 8 7 10 11 12 6 1.5 15\n"
byte b = 1; sbyte sb = 2; short s = 3; ushort us = 4; uint u = 5; long l = 6; ulong ul = 7;
var r1 = b + b;
var r2 = sb + b;
var r3 = us + us;
var r4 = u + sb;
var r5 = u + u;
var r6 = l + u;
var r7 = ul + u;
var r8 = u + 1;
var r9 = b * 1.5F;
var r10 = l * 2.5;
Console.WriteLine($"{r1} {r2} {r3} {r4} {r5} {r6} {r7} {r8} {r9} {r10}");
int x1 = r1;
long x2 = r4;
uint x3 = r8;
```

```nz fragment error="no integer type holds both; cast one side"
long l = 1;
ulong ul = 2;
var r = l + ul;
```

```nz fragment error="cannot assign `int` to `byte`"
byte b = 1;
byte c = b + b;
```

A shift promotes only its left operand, and its count must widen to `int`: `ulong << int` is a `ulong`, `uint << ulong` is an error. Unary `-` and `~` promote their operand: `-b` and `~b` on a `byte` are `int`, `-u` on a `uint` is a `long`, and `-` on a `ulong` or `nuint` is an error.

A `char` widens to `int`, `uint`, `long`, `ulong`, `nint`, `nuint`, `float` and `double` (not to `ushort`: a `char` reaches `\u{10FFFF}`), and an operator reads it as an `int`: `'A' + 1` is `66`, `c < 'B'` compares code points, `'A' == 65` holds, and `c++` moves to the next Unicode scalar value (it aborts past `\u{10FFFF}` or into the surrogates). A `char` is not an integer: `char c = 65;` and `c = c + 1` are errors (`c + 1` is an `int`); write `(char)65`. A compound assignment with a `char` value converts back: `c += 'b'`, `c -= (char)2`, `c |= ' '`, `c <<= 1` and the others run in `int` and store a `char`. The value must be a `char` (a shift takes an integer count), so `c += 1` is an error: write `c += (char)1` or `c++`. The result is checked like `c++`: past `\u{10FFFF}` or into the surrogates it aborts, inside `unchecked` too.

```nz fragment output="66 130 True True 65 65 B C\n"
char c = 'A';
var i = c + 1;
var j = c + c;
bool lt = c < 'B';
bool eq = c == 65;
long l = c;
double d = c;
c++;
Console.WriteLine($"{i} {j} {lt} {eq} {l} {d} {c} {(char)(c + 1)}");
```

```nz fragment error="cannot assign `int` to `char`"
char c = 65;
```

```nz fragment
int a = 1;
long b = a;
double c = b;
long big = 8000000000;
```

Narrowing is not allowed:

```nz fragment error="cannot assign"
double d = 2.5;
int i = d;
```

### Casts

**Casts.** `(T)x` converts a value explicitly. A cast is checked: a value that does not fit stops the program with ``nz: value does not fit in `T` ``, unless it is written inside `unchecked`. A cast of a constant that does not fit is a compile-time error instead. The pairs that are allowed:

| From | To |
|---|---|
| any integer type | any integer type, `float`, `double` |
| `float`, `double` | any integer type (truncated toward zero), `char`, `float`, `double` |
| `char` | any integer type, `float`, `double` |
| any integer type | `char` (a Unicode scalar value) |
| an enum | any integer type, another enum, `char`, `float`, `double` |
| any integer type, `char`, `float`, `double` | an enum |
| any type | the same type (an identity cast) |

A real turns into an integer by truncation toward zero, and NaN, an infinity or a value outside the range stops the program (or, for a constant, is a compile-time error: `(int)1e10`). A `double` above the largest `float` that is cast to `float` also stops the program. `(char)65.0` and `(double)'A'` work. A cast to `nint` or `nuint` checks a constant against the `int` or `uint` range. A cast cannot produce an optional (`(byte?)i` is an error): cast the value, then assign it. A cast between `bool`, `string` or an unrelated type is an error, except the identity cast.

```nz fragment output="300 200 2 -2 A 65 65 A 2.9 4000000000\n"
int i = 300;
double d = 2.9;
double nd = -2.9;
char c = 'A';
byte b = 65;
long l = 4000000000;
Console.WriteLine($"{(long)i} {(byte)(i - 100)} {(int)d} {(int)nd} {(char)b} {(int)c} {(double)c} {(char)65.0} {(float)d} {(uint)l}");
```

```nz fragment error="constant `300` does not fit `byte`"
byte b = (byte)300;
```

```nz fragment error="constant `10000000000` does not fit `int`"
int x = (int)1e10;
```

```nz fragment error="cannot cast `bool` to `int`"
bool t = true;
int n = (int)t;
```

An identity cast moves an owned object like an assignment does, so after `K b = (K)a;` the name `a` is moved:

```nz error="use of moved value `a`"
package main;

class K { public int V = 3; }

void Main(string[] args) {
    K a = new K();
    K b = (K)a;
    int v = a.V;
}
```

One rule of reading casts: `(Name)(x)` is a call of `Name`, not a cast, so write a cast of a parenthesised value to a class or enum with a variable or `checked(...)`/`unchecked(...)`. A cast binds tighter than `switch`: `(int)b switch { ... }` casts `b` first (the order of precedence is in section [1.5](01-basic-operators-control-flow.md#15-operators)). A cast to a generic type (`(List<int>)x`) is not supported.

```nz fragment output="8 16 15 -4 15 5 False False 1099511627776\n"
int a = 12; a &= 10;
int b = 1; b <<= 4;
int c = -16; c >>>= 28;
int d = -16; d >>= 2;
int e = 6; e |= 9;
int f = 6; f ^= 3;
bool t = true; t &= false;
bool u = false; u |= true; u ^= true;
long l = 1; l <<= 40;
Console.WriteLine($"{a} {b} {c} {d} {e} {f} {t} {u} {l}");
```

```nz fragment error="cannot apply compound assignment to `int` and `long`"
int x = 1;
x <<= 1L;
```

## 1.3 Data items (variables, `var`, constants)

### Variables

Every non-optional variable **must be given an initial value**, so it can never hold `null`.

```nz fragment
var count = 0;
count += 1;
string name = "NZ";
int? maybe;
```

Without an initial value, a non-optional variable is rejected:

```nz fragment error="must be initialized"
int n;
```

A local name may not hide another local that is still visible:

```nz fragment error="already declared"
var a = 1;
{
    var a = 2;
}
```

The scope of a local is its whole block, not only what follows its declaration. A use before the declaration is an error, also when an outer name (a top-level constant, a static, a field, a function) is spelled the same, and also from a nested block or a local function written earlier in the block. A nested block may not declare a name that its block declares later; two sibling blocks may each declare it.

```nz error="cannot use local `Limit` before it is declared"
package main;

import std.io.Console;

const int Limit = 10;

void Main(string[] args) {
    int y = Limit;
    const int Limit = 20;
    Console.WriteLine($"{y} {Limit}");
}
```

```nz fragment error="already declared"
{
    var b = 1;
}
var b = 2;
```

### Constants

A constant is a value the compiler knows. It has no storage: every read is replaced by the value. A constant is declared with `const`, a type and a value, as a local, at the top level, or as a member of a class or struct (see [Static members](05-object-oriented-programming.md#516-static)):

```nz output="3 12 km\n"
package main;

import std.io.Console;

const int Size = 3;
const string Unit = "km";

class Grid {
    public const int Cells = Size * Size + Size;
}

void Main(string[] args) {
    const int Local = Grid.Cells - 9;
    Console.WriteLine($"{Local} {Grid.Cells} {Unit}");
}
```

- **What may be constant:** the eight integer types `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long` and `ulong`, `bool`, and `string`. The value is built from literals, operators (no string concatenation), other constants (`Max`, `Grid.Cells`), `sizeof`, `nameof` and `default`. `nint` and `nuint` (their size depends on the target), `char`, `double`, `float` and enums cannot be constants; use `static readonly`:

```nz error="cannot be a constant type"
package main;

const double Pi = 3.14;

void Main(string[] args) {
}
```

- **Order.** Top-level and member constants may read each other in any order, across types and files; they are evaluated in dependency order. A constant that depends on itself, directly or through others, is one error: ``constant `A` depends on itself: `A` -> `B` -> `A` ``.
- **Overflow** is checked: `const int Big = 2147483647 + 1;` is an error, and so is `2147483647 + 1` (or `1 / 0`) on constants anywhere outside `unchecked`.
- **Scope.** A local constant is in scope in its whole block, as any local (see [Variables](#variables)): using it before its declaration is an error, even where a top-level or member constant of the same name exists, and so is naming it in its own value (`const int K = K + 1;`). A local function written after the constant reads it, wherever it is called.
- **Writing** a constant is an error, as is passing it by `ref`.
- **Visibility** follows the first-letter rule of members and top-level names (see [Visibility](20-module-system-and-visibility.md#203-access-modifiers)); a lowercase member constant is seen only inside its type.
- **In `switch` patterns** a named constant works like a literal: `x switch { Max => ..., _ => ... }`; on a value of a smaller integer type an `int` constant that fits works too (`const int Five = 5;` on a `byte`), as a literal does, and one that does not fit is an error. On an enum value a name pattern is a variant (`Red` or `Color.Red`). A pattern that an earlier arm already handles is an error, and so is any arm after `_`.

### Default values (`default`)

`default(T)` is the zero of `T`; a bare `default` takes the type of where it stands (a declared variable, `return`, `=`, an argument, a field initializer).

| Type | `default` |
|---|---|
| `int`, `long`, `float`, `double` | `0` |
| `bool` | `false` |
| `char` | the character with code 0 |
| `T?` | `null` |
| `string` | the empty string `""` |
| a class or struct `C` | an error: write `new C()` or use `C?` |
| an enum | an error: name one of its variants or use `E?` |
| a type parameter `T` | an error: `default(T)` needs a constraint, which is not supported yet |

In a constant, `nameof` can name only constants, and a bare `default` takes the constant's type. A bare `default` anywhere else (`var x = default;`, `1 + default`) needs a type: write `default(int)`.

```nz fragment output="0 0 False True\n"
int i = default;
double d = default(double);
bool b = default;
string? s = default;
Console.WriteLine($"{i} {d} {b} {s == null}");
```

Previous: [0. Introduction](00-introduction.md#0-introduction) · [Contents](README.md) · Next: [1.4 Data Structures](01-basic-data-structures.md#14-data-structures)
