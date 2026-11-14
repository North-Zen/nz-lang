# NZ Programming Language

![](../assets/nz-logo-filled.svg)

> This document is **alive**: every `nz` code block is checked by the real compiler during `cargo test`, and the sections marked `GEN` are rewritten from the compiler's own code with `nzc docs`. The details are in [Compiler](+/Compiler.md).

# 0. Introduction

**NZ** is a *systems* programming language that is memory safe and has **no garbage collector by default**. Its syntax is deliberately very close to **C#**, while its model takes the ideas of **ownership** and **borrowing** from Rust: a strong architecture, but with a syntax that is easier to read.

NZ is designed for many kinds of programs: kernels (`no_runtime`), web/app servers, and games.

> ⚠️ Status: NZ programs now **compile to C and run** through `clang` (`nzc run`), after passing the lexer, parser and semantic analysis. **Ownership and borrow checking are active**: owned objects are freed deterministically, and strings are freed with their last holder (see [Ownership](+/Ownership.md) and [Compiler](+/Compiler.md)).

## Installation, Running Hello World

**Requirements:**

- **Rust toolchain**: [https://rustup.rs](https://rustup.rs)
- **Git**
- **clang**: the C compiler that `nzc` uses to build programs (set `NZ_CC` to use another one).
- **Code editor**: any; the `.nz` extension has no dedicated editor support yet.

**Running NZ code with `nzc run`:**

1. Get the code and enter its folder.
2. Run `cargo run -p nzc -- run <file.nz|folder>`. The compiler checks the program (every error is printed at once with its line and column), translates it to C, builds it with `clang`, and runs it.
3. `nzc check <file.nz|folder>` only checks and prints `ok`; `nzc build <file.nz|folder>` leaves an executable (for a folder, in its `bin` folder). A folder is a package (see 1.6).

```bash
# in the compiler repository (its source is not public yet)
cargo run -p nzc -- run examples/hello.nz
```

The program in `examples/hello.nz` prints its greeting and exits with code 0:

```text
Halo, NZ
```

The same program, with an English greeting:

```nz output="Hello, NZ\n"
package main;

// main.nz
import std.io.Console;

void Main(string[] args) {
    Console.WriteLine("Hello, NZ");
}
```

Every file starts with a `package` line. In a program made of a folder, the package is named after the folder that holds the file (`package main;` for a folder called `main`); see 1.6. `import` lines come next, then the declarations.

The `examples/` folder holds more programs that run the same way, for example `../examples/fibonacci.nz` and `../examples/shapes.nz`. `../examples/stack.nz` (a stack and a queue) and `../examples/bst.nz` (a binary search tree) build linked structures and free every node they own.

## Toolchain

`nzc` is the NZ compiler. The table below is written automatically from the command list inside its code.

<!-- GEN:cli -->
| Command | Purpose |
|---|---|
| `nzc parse <file.nz>` | Parse a `.nz` file and print its AST, or the syntax errors. |
| `nzc check <file.nz\|folder> [--target triple] [--no-runtime] [-Werror]` | Check a file, or a folder as a package with the packages it imports: types, names, `null` and the other rules; print every error at once. `--no-runtime` (also for `build` and `run`) checks a program that links the C library only: its entry point is `void Main()` or `int Main()` (the exit status), and strings, `Console`, classes, arrays, `List<T>`, `Abort` and `Assert` are refused. |
| `nzc build <file.nz\|folder> [-o out] [--target triple] [--release] [-L dir] [--no-runtime] [--emit-c-part out.h] [-Werror]` | Check the program, translate it to C, and compile it to an executable with `clang`: `<folder>/bin/<folder>` for a package, the file's name in the current folder for a file, or `-o`. A debug build by default: no optimisation, debug info (`-g`; a `.pdb` beside the executable on Windows) and `#line` directives, so a debugger or a sanitizer report shows `.nz` files and lines. `--emit-c-part out.h` writes, instead of an executable, the C of the `[UnmanagedCallersOnly]` functions and what they need, all `static inline`, with no `main` and no `#include "nz.h"`: a header to include after the runtime header. `--release` builds optimised C (`-O2`) without debug info; the program behaves the same. `--target triple` (also for `check`) builds for another machine: it passes `--target=<triple>` to clang and uses that target's executable suffix, link libraries and defines; the triples are `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `x86_64-pc-windows-gnu`, `x86_64-pc-windows-msvc`, `x86_64-apple-darwin` and `aarch64-apple-darwin`. Without it nothing is passed and the build is for this machine. A cross link needs the target's sysroot, which `nzc` does not provide (point `NZ_CC` at a configured cross compiler). `run` and `test` start the program here, so they refuse a target that is not this machine. |
| `nzc run <file.nz\|folder> [--target triple] [--release] [-L dir] [--no-runtime] [-Werror] [-- args...]` | Build the program in a temporary location (a debug build, or optimised with `--release`), run it, and exit with its exit code. Everything after `--` is passed to the program as the `string[] args` of `Main`, as `dotnet run -- args` does: `nzc run tool.nz -- input.txt "two words" ""` gives three arguments, the first `args[0]` (the program's own path is not in `args`), and a word that looks like an option (`--release`) is the program's too. Options of `nzc` go before the `--`. `-L dir` (repeatable, also for `build` and `test`) adds a folder where the linker looks for the libraries `[DllImport]` names. |
| `nzc test <file.nz\|folder> [--target triple] [--release] [-L dir] [-Werror]` | Build the package's `[Test]` functions (a debug build, or optimised with `--release`) and run each in its own process; report each result and leaks. |
| `nzc docs <dir>` | Refresh the `GEN` sections of the markdown files in a folder. |
<!-- /GEN -->

`nzc test` runs the functions marked `[Test]`; see 2.5.

**Debug and release builds.** As with `dotnet build` and `dotnet build -c Release`, `build`, `run` and `test` make a debug build unless `--release` is given. A debug build compiles the C without optimisation and with debug info (`-g`), and the C carries `#line` directives that name the `.nz` file and line of every statement, so gdb, lldb, the Visual Studio debugger and sanitizer reports show NZ source: `break main.nz:12` stops on line 12 of `main.nz`. On Windows the debug info is a `.pdb` file written beside the executable. A release build compiles the C with `-O2` (and `-fwrapv -fno-strict-aliasing`, so the optimiser keeps every behaviour of the debug build) and without debug info: it runs faster and prints exactly what the debug build prints. `--release` is accepted by `build`, `run` and `test`.

**Warnings.** A warning is a problem that does not stop the build. It prints as `Warning:` with the source line. After all the diagnostics of a run comes one summary line that names the path as you typed it, such as `main.nz: 1 warning(s)` or `main.nz: 1 error(s), 2 warning(s)`. `check` still ends with `ok` and exit code 0 when there are only warnings. The warnings are: `private` on a name that starts with an uppercase letter, which is public anyway; a `package` line that differs from its folder; an import of the package of the same file (see 1.6); a `///` documentation comment that is not attached to a declaration (see **Comments** below); and an attribute name the compiler does not know (see 2.5). The first one:

```nz warning="starts with an uppercase letter"
package main;

private int Add() {
    return 1;
}

void Main(string[] args) { }
```

The exit code of `nzc` is 0 on success, 1 when the program has errors (or when `-Werror` turned a warning into one), and 2 when the command line itself is wrong (a missing or unknown command or option). When a file has a syntax error, or the loader cannot find a package or a file, `check` prints the errors and the summary line, while `build` and `run` print the errors only, with no summary line, and stop before checking. When the loader cannot read anything at all (a path that does not exist, a folder with no `.nz` files) the message is `nzc: <message>` with exit code 1 and no summary. `-Werror` is accepted by `check`, `build`, `run` and `test`, not by `parse` or `docs`: it prints the same warnings, ends with `1 warning(s) treated as errors`, and exits with 1.

## Characteristics, Rules & Naming

> ⚠️ Note: NZ is **case-sensitive**, and the first letter of a name decides the visibility of functions and members, and types are private unless they are written `public` (see 2.4).

**File naming**

- **Do**:
  - `main.nz`
  - `counter.nz`
- **Avoid**:
  - `Main.NZ` (the extension is always `.nz`, lowercase, written in full)

**Characteristics of NZ:**

- **File extension**: `.nz`.
- **Statically typed**: all types are checked at compile time.
- **Type inference**: `var` infers the type from the initial value.
- **No GC by default**: memory is managed through ownership and borrowing; `Arc<T>` is the explicit choice for shared data (see [Memory Model](+/Memory-Model.md)).
- **Null safety**: `null` is only allowed for optional types `T?`, and a `T?` must be checked before it is used.
- **C# syntax**, with three deliberate differences:
  1. `package a.b;` at the top of every file and `import a.b.C;` replace `namespace`. A folder is a package, and `import` brings in one name, an alias, a group or a whole package (see 1.6).
  2. The opening brace `{` stays on the same line.
  3. The first letter of a name decides the visibility of functions, methods, fields and properties (`Add` is public, `add` is private); classes, structs, interfaces and enums are private unless they are written `public`.

**Comments**:

```nz fragment
// a single-line comment
int x = 5; // explains the value of x

/*
a comment
over several lines
*/
```

A run of `///` lines directly before a declaration is its **documentation comment**, written in markdown. It belongs to the top-level item, the member (field, property, method) or the enum variant that follows, written before the modifiers (`/// x` then `public class C`). `////` and more slashes make a plain comment, and so do `//` and `/* */`. A plain comment between the run and the declaration does not detach it; a blank line does. The stored text is each line without `///` and one following space.

A run that does not belong to any declaration gives a warning: a blank line after it, a place inside a body or before a statement, the position before the `package` line or an `import`, the end of a file, or after code on the same line (`int F() { } /// x`). Under `-Werror` the warning fails the build.

```nz warning="not attached"
package main;
/// lost

int F() {
    return 1;
}

void Main(string[] args) {
}
```

```nz output="2\n"
package main;
import std.io.Console;
/// Adds one.
int Inc(int x) { return x + 1; }
void Main(string[] args) { Console.WriteLine($"{Inc(1)}"); }
```

Parameters and statements have no documentation comments. A `///` inside a string, a raw string or an interpolation hole is not a documentation comment (inside a hole `//` starts an ordinary comment of the hole's own lexing).

**Keywords**

<!-- GEN:keywords -->
NZ has 59 keywords: `import` `package` `void` `public` `private` `class` `struct` `enum` `interface` `var` `new` `this` `if` `else` `while` `for` `foreach` `in` `break` `continue` `return` `switch` `case` `default` `is` `ref` `using` `defer` `unsafe` `true` `false` `null` `abstract` `base` `const` `do` `nameof` `operator` `override` `params` `protected` `readonly` `sealed` `sizeof` `static` `type` `union` `virtual` `where` `async` `await` `move` `lock` `yield` `extern` `volatile` `fixed` `stackalloc` `derive`.
<!-- /GEN -->

# 1. Basic

## 1.1 Data Types

NZ uses the same primitive types as C# for numbers, characters and booleans, plus `string`.

### 1.1.1 Numbers

- **Integer**: ten types, as in C#, listed below
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

A value of any of them is copied, never moved. `nint` and `nuint` have the width of a pointer, so their range depends on the target; a constant converted or cast to them is checked against the `int` or `uint` range, so that a program means the same on every target. `sizeof(nint)` is not supported yet (see "Not yet" in section 7).

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

An integer literal has the first of the types `int`, `uint`, `long`, `ulong` that holds its value, as in C#. A real literal (one with a fraction, an exponent or the `F` suffix) has type `double`, and `float` when it ends in `F`.

Implicit conversions only *widen*, never narrow, and only where every value of the source fits the target, as in C#. A source converts to these types:

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

The same value may differ after a conversion to a real: a `long`, `ulong`, `nint` or `nuint` that does not fit the 24 or 53 bits of a `float` or `double` is rounded, as in C#. Nothing else converts: `short` to `ushort`, `int` to `uint` and `ulong` to `long` are errors, in both directions.

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

**Constants take their type from context.** An integer constant (a literal, a named constant, or an expression of constants) converts to any integer type that holds its value, wherever a value is assigned: a variable, a field, an argument, a return value, an element. `byte b = 200;` and `byte c = 100 + 100;` work; `byte b = 300;` and `sbyte s = -129;` are errors, and so is a variable: `int i = 5; byte b = i;`. A `long` constant converts only to `ulong`, as in C#.

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

**Binary numeric promotion.** An arithmetic, bitwise or comparison operator promotes both operands to one type, with C#'s rules: if one is `double` the result is `double`, else if one is `float` it is `float`; else `ulong` if one is `ulong`; else `long` if one is `long`; else `uint` when both are unsigned among `byte`, `ushort`, `uint`; `uint` with a signed `sbyte`, `short` or `int` gives `long`; everything else is `int`. So `byte + byte` is an `int` (assigning it to a `byte` is an error; cast it), and `byte * 2.5` is a `double`. `nint` and `nuint` follow the same pattern at pointer width: `nint + int` is `nint`, `nuint + uint` is `nuint`, `nint + uint` is `long`. A `ulong` cannot be mixed with a signed type, and neither can `nuint`: there is no integer type that holds both, and the error says so. An integer constant that fits the other operand joins it (`ul + 5` is `ulong`, `u + 5` is `uint`).

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

**Arithmetic is checked.** `+ - *` and unary `-` on every integer type stop the program with ``nz: integer overflow`` when the result does not fit the promoted type, `/` and `%` by zero stop with ``nz: division by zero``, the smallest value divided by (or taken modulo) -1 stops with ``nz: integer overflow``, and a conversion back into a narrower type (`b += 10`, `b++`, a cast) stops with ``nz: value does not fit in `byte` `` naming the target. Shifts never fail: the count is masked to the width of the promoted type, and bits shifted out are lost, as in C# (`3000000000 << 1` is 1705032704). A compound shift on a smaller type is a compound assignment like the others: `b <<= 1` on a `byte` shifts the promoted `int`, then narrows the result back to `byte`, checked, so `byte b = 128; b <<= 1;` stops with ``nz: value does not fit in `byte` `` (inside `unchecked` it keeps the low bits).

**Constant overflow is a compile-time error.** A constant expression whose value does not fit its type is refused before the program runs, as C# does (error CS0220): `2147483647 + 1`, `100000 * 100000`, a constant `Big + 1` and `4294967295 + 1` (a `uint`) are errors, while `2147483647 + 1L` is a `long` and `unchecked(2147483647 + 1)` wraps. Dividing a constant by a constant zero is an error too.

```nz fragment error="constant expression overflows `int`"
int a = 2147483647 + 1;
```

```nz fragment output="-2147483648 2147483648\n"
int a = unchecked(2147483647 + 1);
long b = 2147483647 + 1L;
Console.WriteLine($"{a} {b}");
```

A `char` widens to `int`, `uint`, `long`, `ulong`, `nint`, `nuint`, `float` and `double` (not to `ushort`: a `char` reaches `\u{10FFFF}`), and an operator reads it as an `int`, as in C#: `'A' + 1` is `66`, `c < 'B'` compares code points, `'A' == 65` holds, and `c++` moves to the next Unicode scalar value (it aborts past `\u{10FFFF}` or into the surrogates). A `char` is not an integer: `char c = 65;` and `c = c + 1` are errors (`c + 1` is an `int`); write `(char)65`. As in C#, a compound assignment with a `char` value converts back: `c += 'b'`, `c -= (char)2`, `c |= ' '`, `c <<= 1` and the others run in `int` and store a `char`. The value must be a `char` (a shift takes an integer count), so `c += 1` is an error, as in C#: write `c += (char)1` or `c++`. The result is checked like `c++`: past `\u{10FFFF}` or into the surrogates it aborts, inside `unchecked` too.

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

**Integer literals.** Besides decimal there is hexadecimal (`0xFF`, also `0X`) and binary (`0b1010`, also `0B`). A leading zero changes nothing: `007` is the decimal number 7. A `_` may stand between two digits of the literal, as many as you like: `1_000_000`, `0xFFFF_FFFF`, `0b1111_0000`. It may not come first, last, right after `0x` or `0b`, or before a suffix.

The type of an integer literal follows C#: it is the first type of a list that holds the value, and a suffix chooses the list. Without a suffix the list is `int`, `uint`, `long`, `ulong`; with `U` it is `uint`, `ulong`; with `L` it is `long`, `ulong`; `UL` (also written `LU`) is always `ulong`. The letters may be upper or lower case (`1L`, `1l`, `1UL`, `1ul`, `1uL`, `1lu`). The rule is the same in every base: `0x7FFFFFFF` is an `int`, `2147483648` and `0xFFFFFFFF` are `uint` values (4294967295, not -1), `4294967296` is a `long` and `9223372036854775808` a `ulong`. A value above the largest `ulong` is an error. As in C#, `-2147483648` is the smallest `int` and `-9223372036854775808` the smallest `long`: the minus belongs to the literal there. Arithmetic on such a literal runs in its type and is checked like any other: `3000000000 + 1` is the `uint` 3000000001, `4294967295 + 1` is a compile-time error (a constant that overflows, see below), `uint big = 4294967295; big + 1` aborts with an overflow when it runs, and `4294967295L + 1` is `long` arithmetic. Shifts are not checked: `3000000000 << 1` is the `uint` 1705032704, as in C#.

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

### 1.1.2 Characters, Booleans & Strings

```nz fragment
char letter = 'A';
bool active = true;
string name = "NZ";
```

A `string` is an immutable sequence of UTF-8 bytes (`Length` is the number of bytes). Two limits apply: a string the language operators make (`+`, `+=`, an interpolation) can be up to 2,147,483,647 bytes, and a longer one stops the program with `nz: string too long`; a string a library member makes (`Substring`, `Replace`, `PadLeft`, `string.Join`, `StringBuilder.ToString` and the rest) can be up to 1,073,741,791 bytes (.NET's `String.MaxLength`, counted in bytes), and a longer one throws `OutOfMemoryException`, as .NET does. As in C#, a string is copied freely: assigning it, passing it, storing it in a field, a list or a tuple never moves it, and two copies compare equal by their bytes. A computed string is freed when its last holder goes; a literal is never freed (see [Ownership: Strings](+/Ownership.md#strings)).

A `bool` prints as `True` or `False`, in `Console.WriteLine` and in an interpolation alike, as C#'s `bool.ToString()` does.

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

The `+` operator on a string accepts any value on its right-hand side, like C#:

```nz fragment
string text = "total = " + 3;
```

## 1.2 Variables & `var`

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

A local name may not hide another local that is still visible (the C# rule):

```nz fragment error="already declared"
var a = 1;
{
    var a = 2;
}
```

As in C#, the scope of a local is its whole block, not only what follows its declaration. A use before the declaration is an error, also when an outer name (a top-level constant, a static, a field, a function) is spelled the same, and also from a nested block or a local function written earlier in the block. A nested block may not declare a name that its block declares later; two sibling blocks may each declare it.

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

## 1.3 Operators

```nz fragment
int a = 7;
int b = 2;
var sum = a + b;
bool greater = a > b && b != 0;
int bits = a & b;
var answer = greater ? "yes" : "no";
a++;
```

Besides the usual C# operators, NZ supports `>>>` (a logical shift right), unary `+`, `??` (coalesce), `??=`, `?.` (optional chaining) and `is`.

`>>` keeps the sign and `>>>` does not. Both take an `int` or a `long` on the left and an `int` count on the right; the count is masked to the width of the type, so a shift never fails at run time (`1 >>> 33` is `1 >>> 1`). Unary `+` leaves a number as it is.

```nz fragment output="15 15 0 -4 2147483644\n-3 -3 -1\n"
Console.WriteLine($"{-16 >>> 28} {-1L >>> 60} {1 >>> 33} {-8 >> 1} {-8 >>> 1}");
int x = -3;
Console.WriteLine($"{+x} {+ +x} {1 - +2}");
```

**Compound assignment.** `+= -= *= /= %=` work as before. `&= |= ^=` take an integer target and a value that widens to it, or two `bool`s; `<<= >>= >>>=` take an integer target and an `int` count. `x op= y` means `x = x op y`, but the target and its index are evaluated once. On `bool`, both sides are always evaluated (no short circuit), like `&`, `|` and `^`. The value must convert to the target's type implicitly (a constant that fits counts, so `b += 1` works on a `byte`); the operation runs in the promoted type and the result is converted back, checked: `byte b = 250; b += 10;` stops the program with ``nz: value does not fit in `byte` ``. `++` and `--` work the same way on every number type.

**`unchecked` and `checked`.** As in C#, `unchecked(expr)` and `unchecked { ... }` make the integer `+ - *`, negation, compound assignments, `++`/`--` and integer casts written inside wrap around or keep the low bits instead of stopping the program; `checked(...)` and `checked { ... }` restore the default. Four things stay checked inside `unchecked`: `/` and `%` (a zero divisor, and the smallest value divided by -1), a cast from a real to an integer, and a cast from an integer to `char` (a `char` must be a Unicode scalar value). A function called inside keeps its own checking. Inside `unchecked` a constant folds the same way, so `unchecked((byte)300)` is 44. Both words are keywords, as in C#: nothing can be named `checked` or `unchecked`.

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

A `char` is not an integer, so `c += 1` is an error, as in C#: write `c += (char)1` or `c++`.

**Casts.** `(T)x` converts a value explicitly, with the same syntax as C#. A cast is checked: a value that does not fit stops the program with ``nz: value does not fit in `T` ``, unless it is written inside `unchecked`. A cast of a constant that does not fit is a compile-time error instead. The pairs that are allowed:

| From | To |
|---|---|
| any integer type | any integer type, `float`, `double` |
| `float`, `double` | any integer type (truncated toward zero), `char`, `float`, `double` |
| `char` | any integer type, `float`, `double` |
| any integer type | `char` (a Unicode scalar value) |
| an enum | any integer type, another enum, `char`, `float`, `double` |
| any integer type, `char`, `float`, `double` | an enum |
| any type | the same type (an identity cast) |

A real turns into an integer by truncation toward zero, and NaN, an infinity or a value outside the range stops the program (or, for a constant, is a compile-time error: `(int)1e10`). A `double` above the largest `float` that is cast to `float` also stops the program, where C# gives an infinity. `(char)65.0` and `(double)'A'` work as in C#. A cast to `nint` or `nuint` checks a constant against the `int` or `uint` range. A cast cannot produce an optional (`(byte?)i` is an error): cast the value, then assign it. A cast between `bool`, `string` or an unrelated type is an error, except the identity cast.

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

Casts are read as C# reads them, with one difference. `(Name)(x)` is a call of `Name`, not a cast, so write a cast of a parenthesised value to a class or enum with a variable or `checked(...)`/`unchecked(...)`. A cast binds tighter than `switch`, as in C#: `(int)b switch { ... }` casts `b` first (the order of precedence is in section 1.3). A cast to a generic type (`(List<int>)x`) is not supported.

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

`switch` follows C#'s grammar, where the operand of a `switch` expression is a range expression: a prefix operator, a cast, a range or a postfix operator binds tighter than `switch`, and a binary operator looser. So `-n switch { ... }` is `(-n) switch { ... }`, `(int)d switch { ... }` is `((int)d) switch { ... }`, `a..b switch { ... }` is `(a..b) switch { ... }`, and `1 + n switch { ... }` is `1 + (n switch { ... })`:

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

## 1.4 Control Flow

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

On a union, a `switch` arm names a case type (`Cat c => ...`), and `switch` also has C#'s statement form with `case` and `default` sections; both are in section 2.10.

`break` is only valid inside a loop or a `switch` statement, and `continue` inside a loop:

```nz fragment error="only valid inside a loop"
break;
```

## 1.5 Functions

Functions are declared as in C#: **the return type goes in front of the name**, and `void` stands for "returns nothing" (it is only valid as a return type). The entry point of a program is `void Main(string[] args)`.

```nz output="Hello, NZ\n3\n"
package main;

import std.io.Console;

int Add(int a, int b) {
    return a + b;
}

void Greet(string name) {
    Console.WriteLine($"Hello, {name}");
}

void Main(string[] args) {
    Greet("NZ");
    Console.WriteLine($"{Add(1, 2)}");
}
```

A non-void function must return a value on **every path**:

```nz error="not all code paths return"
package main;

int Positive(bool b) {
    if (b) {
        return 1;
    }
}

void Main(string[] args) { }
```

**Generic functions** infer their type from the arguments:

```nz output="1\nfirst\n"
package main;

import std.io.Console;

T First<T>(T a, T b) {
    return a;
}

void Main(string[] args) {
    int x = First(1, 2);
    string s = First("first", "second");
    Console.WriteLine($"{x}");
    Console.WriteLine(s);
}
```

**`ref` parameters** are used when a function writes to the caller's variable. The argument must be a variable of exactly the same type:

```nz output="x = 2, y = 1\n"
package main;

import std.io.Console;

void Swap(ref int a, ref int b) {
    var tmp = a;
    a = b;
    b = tmp;
}

void Main(string[] args) {
    int x = 1;
    int y = 2;
    Swap(x, y);
    Console.WriteLine($"x = {x}, y = {y}");
}
```

### Local functions

A function can be declared inside a block. It is written like any other function, the return type first, and it is visible only in that block. It can be called before or after its declaration, and it can read the variables around it.

```nz output="26\n120\n"
package main;

import std.io.Console;

int Total(int n) {
    int bonus = 10;

    int sum = add(n);

    int add(int x) {
        return x + bonus;
    }

    return sum + add(1);
}

int Fact(int n) {
    int fact(int k) {
        if (k <= 1) {
            return 1;
        }
        return k * fact(k - 1);
    }
    return fact(n);
}

void Main(string[] args) {
    Console.WriteLine($"{Total(5)}");
    Console.WriteLine($"{Fact(5)}");
}
```

`add` is called before its declaration, reads `bonus`, and `fact` calls itself. The rules:

- **Where.** A local function is a statement directly inside a braced block: a function body, the body of an `if` or a loop, a plain block, a `defer` block. It cannot be the unbraced body of an `if`, `else`, `while`, `for`, `foreach` or `using`, a `for` initializer, or follow `defer`:

```nz fragment error="a local function must be declared directly inside a block"
if (args.Length == 0) int g() { return 1; }
```

- **Visible only in its block.** The name is known from the start of the block to its end, and nowhere else:

```nz fragment error="unknown name"
{
    int g() { return 8; }
    Console.WriteLine($"{g()}");
}
int h = g();
```

- **What it can read.** The variables declared before it, in its own block or an enclosing one: locals, parameters, loop variables and `is` bindings. A variable or constant of the block declared later cannot be used inside the function (``cannot use local `x` before it is declared``, the block-scope rule of [Variables](#12-variables--var)). In a method it can also read the fields and properties of the object and call methods that do not change it. A local function that calls another local function reads what that one reads, too.
- **Calls before the declaration.** A function that reads no variable can be called anywhere in its block. One that does can be called only where every variable it reads is already declared; the same holds for a call from inside another local function:

```nz fragment error="which is not declared at this point"
int r = early();
int x = 1;
int early() { return x; }
```

- **Read-only.** A local function only reads what it captures. It cannot assign a captured variable (also with `++` or `+=`), pass it by `ref`, move it, write a field or element of it, or call a method on it that changes it. The same holds for `this` in a method:

```nz error="captured variables are read-only"
package main;

void Main(string[] args) {
    int count = 0;
    void bump() {
        count = count + 1;
    }
    bump();
}
```

  To change a variable, give the function a `ref` parameter and pass the variable at the call (as with any function, no keyword at the call):

```nz output="2 1\n7\n"
package main;

import std.io.Console;

void Swap(ref int a, ref int b) {
    int t = a;
    a = b;
    b = t;
}

void Main(string[] args) {
    int x = 1;
    int y = 2;
    void swapThem(ref int p, ref int q) {
        Swap(p, q);
    }
    swapThem(x, y);
    Console.WriteLine($"{x} {y}");
    void addTo(ref int total, int amount) {
        total = total + amount;
    }
    addTo(x, 5);
    Console.WriteLine($"{x}");
}
```

  Assigning a variable **outside** the local function is fine; each call reads the value the variable has at that call:

```nz fragment output="5\n"
int x = 1;
int f() { return x; }
x = 5;
Console.WriteLine($"{f()}");
```

- **Optionals are not narrowed inside.** The body is checked once, so a captured `T?` is not known to be non-null in it, even when every call is inside an `if (x != null)`. Test it inside the function. A call does not undo the caller's own checks, except through a `ref` parameter ([Null Safety](+/Null-Safety.md)):

```nz fragment error="possibly null"
string? s = null;
if (s != null) {
    int f() { return s.Length; }
    int n = f();
}
```

- **In a method.** The function may read fields and call methods that leave the object as it is; writing a field, or calling a method that changes the object, is the same error as for a captured variable:

```nz output="12\n"
package main;

import std.io.Console;

class Counter {
    int Step = 2;
    int Value = 10;

    int Next() {
        int plus() {
            return Value + Step;
        }
        return plus();
    }
}

void Main(string[] args) {
    Counter c = new Counter();
    Console.WriteLine($"{c.Next()}");
}
```

```nz error="a local function only reads `this`"
package main;

class Counter {
    int Value = 10;

    void Grow() {
        void bump() {
            Value = Value + 1;
        }
        bump();
    }
}

void Main(string[] args) { }
```

- **Names.** A local function is written with a lower-case first letter: it is visible only in its block, so the rule "an upper-case name is public" does not apply, and it takes no `public` or `private`. An upper-case name works but gives a warning (an error with `-Werror`):

```nz warning="visible only in its block"
package main;

void Main(string[] args) {
    int Helper(int x) {
        return x;
    }
    int r = Helper(1);
}
```

  Inside its block a local function hides a top-level function, a method or a built-in of the same name. It cannot reuse the name of a visible variable or of another local function, in either order, and its parameters cannot reuse the name of a variable of the enclosing function:

```nz output="-3\n"
package main;

import std.io.Console;

int Abs(int v) { return 100; }

void Main(string[] args) {
    int abs(int v) { return v; }
    Console.WriteLine($"{abs(-3)}");
}
```

```nz fragment error="already declared"
int n = 1;
int n() { return 2; }
```

- **Jumps.** `return` leaves the local function and has its own result type. `break` and `continue` cannot reach a loop of the enclosing function:

```nz fragment error="`break` is only valid inside a loop"
while (true) {
    int f() { break; return 1; }
    int r = f();
}
```

- **Not a value yet.** A local function can only be called; `var g = f;` is an error until function values arrive with lambdas:

```nz fragment error="must be called"
int f() { return 1; }
var g = f;
```

- **Not supported yet.** A generic local function (`int f<T>(T x) { ... }`) and a local function used as a value. The first reports "not supported yet"; using a local function as a value reports ``function `f` must be called``. The type parameters of the enclosing function or class can be used inside a local function.

## 1.6 Packages & Imports

A **package** is a folder. All the `.nz` files directly inside one folder are the files of one package; a subfolder is another package. A program is the package of the folder you give to `nzc`, plus every package it imports.

```bash
cargo run -p nzc -- run examples/packages/app
```

The program in `examples/packages/` has two packages:

```text
examples/packages/
  app/main.nz          package app;
  geo/point.nz         package geo;
  geo/distance.nz      package geo;
```

```text
// examples/packages/app/main.nz
package app;

import std.io.Console;
import geo.Point;
import geo.{Origin, Distance2};

void Main(string[] args) {
    Point a = Origin();
    Point b = new Point();
    b.X = 3;
    b.Y = 4;
    Console.WriteLine($"squared distance = {Distance2(a, b)}");
}
```

```text
// examples/packages/geo/point.nz
package geo;

public class Point {
    int X;
    int Y;
}

public Point Origin() {
    return new Point();
}
```

```text
// examples/packages/geo/distance.nz
package geo;

int square(int n) {
    return n * n;
}

public int Distance2(in Point a, in Point b) {
    return square(b.X - a.X) + square(b.Y - a.Y);
}
```

It prints `squared distance = 25`. The `.nz` files of a package are read in order of their file names.

### The `package` line

Every file starts with `package <name>;`. In folder mode the name of a package is its folder: `geo/` is `geo`. A file whose `package` line says something else gets a warning, and the folder name is used:

```text
Warning: package `application` differs from its folder `app`; the folder name is used
```

A folder name must be a plain identifier (ASCII letters, digits and `_`, not starting with a digit), and not a reserved word. A folder called `my-app`, `2app`, `my.app` or `class` cannot be a package, for example ``folder `my-app` is not a valid package name``, and for a reserved word ``folder `class` is not a valid package name: `class` is a reserved word``. The first segment `std` is reserved for the standard library, so no package of yours can be called `std` (``the package name `std` is reserved for the standard library``).

### One namespace per package

The files of a package share one namespace. A function or type declared in `point.nz` is visible in `distance.nz` without an import (`Point` in `Distance2` above), and two files cannot declare the same name: the second is `duplicate declaration`. Visibility follows 2.4, and at the top level **private means visible in the whole package**: `square` above can be called from `point.nz` too, but not from `app`. Only what is public (a function that starts with an uppercase letter, a type written `public`) can be imported by another package.

Imports belong to one file. A file that does not write `import std.io.Console;` cannot use `Console`, even when another file of its package does.

### Import forms

| Form | Brings in |
|---|---|
| `import geo.Point;` | `Point` |
| `import geo.Point as P;` | `Point`, under the name `P` |
| `import geo.{Point, Origin};` | each name of the group |
| `import geo.*;` | every public name of `geo` (a glob) |

An import names a package and a declaration (`import geo;` is an error). The standard library is a set of built-in packages (`std.io`, `std.collections`, see 6) and takes the same four forms. An alias works on one name only: `import geo.* as G;` is an error.

```nz output="3\n"
package main;

import std.io.Console;
import std.collections.List as Seq;

void Main(string[] args) {
    Seq<int> numbers = new Seq<int>();
    numbers.Add(1);
    numbers.Add(2);
    numbers.Add(3);
    Console.WriteLine($"{numbers.Count}");
}
```

A name used with its package (`geo.Point p;`) without importing it is an error that suggests the import: ``unknown type `geo.Point`; import it with `import geo.Point;` ``.

**Globs.** A name has a rank in each file. Declarations of the file's own package (in any file), explicit imports and aliases are the highest rank, and they win over a glob: `import geo.*;` never hides your own `Point`, and `import geo.Point;` beats a glob that also holds a `Point`. A glob comes next. When two globs bring in different declarations with one name, there is no error at the imports; the error appears where the name is used, and it names the packages:

```text
`Thing` is ambiguous: it is declared in `p` and `q`, both imported with `.*`; import it by name or with an alias
```

An alias or an import by name fixes it (`import p.Thing;`). Importing the same package with `.*` twice, or importing the same name twice, is an error.

An import of the package of the file itself has no effect, since its names are already visible; it is a warning (`this import names the package of this file and has no effect`), and a glob of the own package brings in nothing. An alias that differs from the name does work (`import app.Helper as H;`), as a second name.

```nz warning="has no effect"
package main;

import main.Helper;

class Helper { }

void Main(string[] args) { }
```

An import of a name that the file's own package also declares is an error at the import (``the import of `geo.Point` clashes with `Point` declared in package `app` ``).

### Import cycles

Packages cannot import each other, directly or through other packages. The compiler reports one error per group of packages that import each other in a circle, naming the shortest cycle:

```text
Error: import cycle: `p` imports `q`, which imports `p`; packages cannot import each other
```

Importing a package of the standard library takes no part in a cycle.

### Running a folder or a file

`nzc check`, `nzc build` and `nzc run` take a `.nz` file or a folder.

- **A folder** is the entry package, and it needs a `Main`: `void Main()`, `void Main(string[] args)`, `int Main()` or `int Main(string[] args)`, as in C# (an `int` result is the exit status; any other return type, such as a `Result`, is refused). A folder without `Main`, such as a library, fails with `no entry point`, for `check` as well. `nzc build app` writes the executable into the package's `bin` folder, named after the package folder: `app/bin/app` (`app\bin\app.exe` on Windows), wherever you build from, like `dotnet build`. The `bin` folder is created when missing. `-o` names another executable instead. An error in the program is reported before any problem with the output path; if a folder already has the executable's name, the build stops with ``cannot write the executable `app/bin/app`: a folder of that name exists; use -o``. A single file, `nzc build hello.nz`, still leaves `hello` in the current folder.
- **A file** is the entry package on its own. Its `package` line is its name, and no folder name check is made. No other file of its folder is read, unless the program imports the file's own package (`package app;` in `app/main.nz` with `import app.Helper;`): then every other `.nz` file of that folder is read as part of the package, so a broken file there fails the build.

Imported packages are found from a **source root**. By default it is the folder above the entry folder: `import geo.Point;` loads the folder `geo` next to `app`, and `import a.b.C;` loads `<root>/a/b/`. When the `package` line of the entry package is dotted and the folder path ends with that name (the folder `src/app/cli` with `package app.cli;`), the source root is the path before the name (`src`), the package is `app.cli`, and no warning is given. For a file, the same rule is applied to the folder that holds it. There is no option to set the root.

A package that cannot be found (``package `geo` not found: no folder `<root>/geo` ``, where the message shows the whole path) or a folder with no `.nz` files is an error at the import that asked for it.

## 1.7 Errors: Abort and Assert

The complete guide to errors, with every runtime stop, the `Try` members, the comparison with C# and what is planned, is [Error Handling](+/Error-Handling.md). This section is the short form.

`Abort` and `Assert` are built-in functions. They need no `import`.

- `Abort(string message)` stops the program.
- `Assert(bool condition, string message)` does nothing when `condition` is true, and stops the program when it is false.

```nz fragment
int n = args.Length;
Assert(n >= 0, "negative length");
if (n > 100) {
    Abort("too many arguments");
}
```

Stopping means: standard output is flushed, `nz: <message>` (for `Assert`, `nz: assertion failed: <message>`) and a newline go to standard error, and the exit code is 1. No cleanup runs: owned values are not dropped, `defer` statements do not run, and the `NZ_ALLOC_REPORT=1` live-object report is not printed. This is the same as the other runtime stops (index out of range, overflow, `.Value` of a null optional).

The message of `Assert` is evaluated only when the condition is false, so it can be costly to build. Since `Abort` never returns, a function that returns a value may end with `Abort(...);` and no `return`. Both are calls with no value, so `var x = Abort("m");` is an error. A method named `Abort` or `Assert` hides the built-in for calls by that bare name inside its class. A local variable with that name makes the call an error (`` `Abort` is a variable, not a function ``), and a top-level function with that name is an error (`` `Abort` is a built-in function and cannot be declared ``).

**Current limits.** For the check that every path returns a value, `Abort` ends the path. For the other three flow analyses (definite assignment, null narrowing, and move analysis) it does not: the compiler treats the code after an `Abort` as reachable. Valid guard code can be rejected because of this. Here the path that moves `n` stops at `Abort`, yet the last line is an error:

```nz error="may have been moved"
package main;

import std.io.Console;

class Node {
    string Name = "n";
}

string Eat(Node n) {
    return n.Name;
}

void Main(string[] args) {
    Node n = new Node();
    if (args.Length > 0) {
        string s = Eat(n);
        Abort(s);
    }
    Console.WriteLine(n.Name);
}
```

The workaround is `return`: with `return;` in place of `Abort(s);` the program is accepted. In the same way, a move inside an `Assert` message counts after the `Assert`, even though the message runs only when the assertion fails: after `Assert(c, Eat(n));`, using `n` is an error. `Abort` does not yet end a path for these three analyses.

### `Result`

A function that can fail returns `Result<T, E>`: either `Ok<T>` with the value or `Err<E>` with the error. They are declared in NZ code (C# style) in the built-in package `std.result` and need no `import`:

```nz-api
public union Result<T, E>(Ok<T>, Err<E>);
public record struct Ok<T>(T Value);
public record struct Err<E>(E Error);
```

`new Ok<int>(1)` converts to `Result<int, string>` as any case converts to its union, and a `switch` reads it:

```nz output="value 1\nerror: cannot read `two`\n2\n"
package main;

import std.io.Console;
import std.collections.List;

Result<int, string> Parse(string s) {
    if (s == "one") {
        return new Ok<int>(1);
    }
    return new Err<string>($"cannot read `{s}`");
}

void Main(string[] args) {
    var inputs = new List<string>();
    inputs.Add("one");
    inputs.Add("two");
    foreach (var s in inputs) {
        switch (Parse(s)) {
            case Ok<int> ok:
                Console.WriteLine($"value {ok.Value}");
                break;
            case Err<string> err:
                Console.WriteLine($"error: {err.Error}");
                break;
        }
    }
    string text = Parse("one") switch {
        Ok<int> ok => $"{ok.Value + 1}",
        Err<string> err => err.Error,
    };
    Console.WriteLine(text);
}
```

A `switch` that forgets a case gets the warning of section 2.10, and a type that is not a case is an error:

```nz error="is not a case of"
package main;

class Missing { }

int Get(Result<int, string> r) => r switch {
    Ok<int> ok => ok.Value,
    Missing m => 0,
    _ => -1,
};

void Main(string[] args) {
    int n = Get(new Ok<int>(3));
}
```

`Ok` and `Err` are positional `record struct`s (C# 10): `record struct Point(int X, int y);` declares a struct whose public fields are its parameters, whatever their case. `new Point(1, 2)` gives them their values in order, `new Point()` gives each its default (as C#'s parameterless struct constructor does), and `var (a, b) = p;` takes them apart in order:

```nz output="1 2 0\n"
package main;

import std.io.Console;

record struct Point(int X, int y);

void Main(string[] args) {
    Point p = new Point(1, 2);
    var (a, b) = p;
    Point z = new Point();
    Console.WriteLine($"{a} {p.y} {z.X}");
}
```

`new Point()` needs a default for every parameter: a non-optional class, `string` or struct has none in NZ, because a non-optional type never holds `null` (section 3), where C# would leave it `null`. So `new Ok<C>()` with a class `C`, and `new P()` for `record struct P(string Name)`, are errors that ask for the values (`new Ok<C>(Value)`); declare the parameter optional (`string? Name`) to allow `new P()`. `record class P(int X);` (or `record P(int X);`) is a positional record class: the same public fields and `new P(1)`, but it is not taken apart with `var (a, b) = p;`. `with`, record equality and `ToString` are not supported yet. A program's own type named `Result` hides the built-in one, and `?` (below) works only on the built-in one.

### Passing errors on with `?`

The postfix operator `?` passes an error on. On a `Result<T, E>` it gives the `Ok` value, or makes the enclosing function return the `Err` at once; on a `T?` it gives the value, or makes the function return `null`. Real C# has no such operator (it throws exceptions instead); NZ takes it from Rust.

```nz output="ok 2\nerr bad\n2 -1\n"
package main;

import std.io.Console;

Result<int, string> Parse(string s) {
    if (s == "1") {
        return new Ok<int>(1);
    }
    return new Err<string>("bad");
}

Result<int, string> Twice(string s) {
    int n = Parse(s)?;
    return new Ok<int>(n * 2);
}

int? Find(int k) {
    if (k > 0) {
        return k;
    }
    return null;
}

int? Next(int k) {
    int v = Find(k)?;
    return v + 1;
}

string Show(Result<int, string> r) => r switch {
    Ok<int> ok => $"ok {ok.Value}",
    Err<string> e => $"err {e.Error}",
};

void Main(string[] args) {
    Console.WriteLine(Show(Twice("1")));
    Console.WriteLine(Show(Twice("x")));
    Console.WriteLine($"{Next(1) ?? -1} {Next(0) ?? -1}");
}
```

The standard library offers `Try` twins of its file members that return a `Result<T, IOError>` instead of stopping the program, so `?` passes a failed read on (see [Standard Library: members that return a result](+/Standard-Library.md#members-that-return-a-result-try-twins)):

```nz output="no such file\n"
package main;

import std.io.Console;
import std.io.File;

Result<int, IOError> Count(string path) {
    string text = File.TryReadAllText(path)?;
    return new Ok<int>(text.Length);
}

void Main(string[] args) {
    switch (Count("no-such-file-nz-readme.txt")) {
        case Ok<int> ok:
            Console.WriteLine($"{ok.Value} characters");
            break;
        case Err<IOError> e:
            Console.WriteLine(e.Error.Kind == IOErrorKind.NotFound ? "no such file" : e.Error.Message);
            break;
    }
}
```

The rules:

- `?` may stand anywhere in a function body where an expression can: in an argument, a condition, an arm of `?:`, `??`, `&&`, `||` or `switch`, a `when` guard, an interpolation hole, a loop header. It leaves the function exactly as `return` would (see [Ownership: `?` leaves the function like `return`](+/Ownership.md#-leaves-the-function-like-return)). Inside a local function it leaves that local function.
- On a `Result<T, E>` the function must return a `Result` whose error type is exactly `E` (no conversion); the success types may differ. On a `T?` the function must return an optional type (any `U?`). A `Result` operand in a function returning an optional, or the other way round, is an error. `Main` returns `void` or `int`, so `?` cannot stand in `Main`, and a `Main` that returns a `Result` is refused (``entry point `Main` must be `void Main()`, ...``).
- `?` cannot stand inside `defer`, nor in a field, constant or static initializer.
- The operand is evaluated once, where it stands, and is consumed: a variable that holds an owned `Result` is moved, and a field or a list element that owns something cannot be the operand (copy a `Result` of copyable values freely).
- `x?` does not make `x` non-null afterwards.
- `?` may also be a statement on its own: `Check()?;` evaluates the call, leaves the function with the error exactly as `var x = Check()?;` would, and discards the success value (an owned value, such as a string, a list or a class instance, is released at the end of the statement). Only a `?` can be a statement this way: `Check();` keeps its `Result` as a discarded value and `x + 1;` or `x;` is still an error, as in C#.

**Reading `?` next to the conditional operator.** After an expression, a `?` followed by a token that can start an expression is the conditional operator `c ? a : b`, as in C#; otherwise it passes an error on. So `f()? - 1` is a conditional (and an error without its `:`); write `(f()?) - 1` to pass the error on and then subtract. The same holds for `+`, `-`, `*`, `&`, `!`, `~`, `(`, `[`, `^` and `..` after the `?`. `?.`, `?[` and `??` are operators of their own, so `f()?.x` is a null-conditional member access and `f()? ?? 0` passes the error on, then coalesces. In an interpolation hole with a format, put the operator in parentheses: `$"{(f()?):D2}"`.

```nz error="(f()?)"
package main;

Result<int, string> Get() {
    return new Ok<int>(3);
}

Result<int, string> Less() {
    int n = Get()? - 1;
    return new Ok<int>(n);
}

void Main(string[] args) {
}
```

The error types must be identical:

```nz error="must be identical"
package main;

Result<int, int> Code() {
    return new Err<int>(1);
}

Result<int, string> Run() {
    int n = Code()?;
    return new Ok<int>(n);
}

void Main(string[] args) {
}
```

## 1.8 Constants

A constant is a value the compiler knows. It has no storage: every read is replaced by the value. A constant is declared with `const`, a type and a value, as a local, at the top level, or as a member of a class or struct (see [Static members](#27-static-members)):

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
- **Overflow** is checked: `const int Big = 2147483647 + 1;` is an error, and so is `2147483647 + 1` (or `1 / 0`) on constants anywhere outside `unchecked`, as in C#.
- **Scope.** A local constant is in scope in its whole block, as any local (see [Variables](#12-variables--var)): using it before its declaration is an error, even where a top-level or member constant of the same name exists, and so is naming it in its own value (`const int K = K + 1;`). A local function written after the constant reads it, wherever it is called.
- **Writing** a constant is an error, as is passing it by `ref`.
- **Visibility** follows the first-letter rule of members and top-level names (see [Visibility](#24-visibility)); a lowercase member constant is seen only inside its type.
- **In `switch` patterns** a named constant works like a literal: `x switch { Max => ..., _ => ... }`; on a value of a smaller integer type an `int` constant that fits works too (`const int Five = 5;` on a `byte`), as a literal does, and one that does not fit is an error. On an enum value a name pattern is a variant (`Red` or `Color.Red`). A pattern that an earlier arm already handles is an error, and so is any arm after `_`.

## 1.9 Statics and `unsafe`

A `static` is one variable for the whole program, declared at the top level or as a [static field](#27-static-members). A mutable static can be read and written only inside `unsafe`: an `unsafe { ... }` block, or an `unsafe` function, method, class or struct. A `static readonly` is never written and can be read anywhere.

```nz output="hi 2\n"
package main;

import std.io.Console;

static int counter;
static readonly string Greeting = "hi";

unsafe void Bump() {
    counter++;
}

void Main(string[] args) {
    unsafe {
        Bump();
        Bump();
        Console.WriteLine($"{Greeting} {counter}");
    }
}
```

```nz error="can only be used inside `unsafe`"
package main;

static int counter = 0;

void Main(string[] args) {
    counter = 1;
}
```

- **Initializers are constant.** A static starts with a constant expression, `null` (optional types), `default`, or nothing when its type has a default value (numbers, `bool`, `char`, optionals). A value computed when the program starts (`static List<int> xs = new List<int>();`) is not supported yet. So a static of a `char`, `double` or `float` type cannot take a literal initializer: it is zero by default (numbers, `bool`, `char`), or written as an optional. A static of an enum type starts with a member of that enum (`static readonly Color c = Color.Red;`, `static Color? d = Color.Red;`) or is optional (`static Color? e = null;`; enums have no default). As in C#, a member keeps its enum's type there too: `static int n = Color.Red;` and `const int K = Color.Red;` are errors (``cannot implicitly convert `Color` to `int`; write `(int)Color.Red` ``), and `(int)Color.Red` works. The same holds for a tuple: `(int, int)` needs `(int, int)?`, as a tuple field needs an initializer.

```nz output="Green 1\n"
package main;

import std.io.Console;

enum Color { Red, Green }

static readonly Color Start = Color.Green;
static readonly int Index = (int)Color.Green;

void Main(string[] args) {
    Console.WriteLine($"{Start} {Index}");
}
```

```nz error="cannot implicitly convert `Color` to `int`; write `(int)Color.Green`"
package main;

enum Color { Red, Green }

static readonly int Index = Color.Green;

void Main(string[] args) {
}
```
- **Never narrowed.** Any call can change a static, so `if (s != null) { ... }` does not make `s` non-null inside; test it with a pattern (`if (s is int v)`) or copy it into a local first.
- **`unsafe`** is valid on functions, methods, classes and structs. It allows the use of mutable statics and the systems features of [1.13](#113-unsafe-code-and-c-interop) (pointers, `stackalloc`, `NativeMemory`, calls into C): the ownership and borrowing rules are not suspended inside it.
- **Calling an `unsafe` function needs `unsafe`.** A call of an `unsafe` function or method, or of any method of an `unsafe` class or struct, is written inside an `unsafe` block or inside a function, method or type marked `unsafe`: ``calling unsafe function `Bump` needs an unsafe context; wrap the call in `unsafe { }` ``. So is `new T()` when a field initializer of an `unsafe` type `T` is not a constant expression (it calls something, creates an object with `new`, or writes or reads a static). The rule is lexical: a function that is not `unsafe` stays safe, whoever calls it, and the need does not spread to its callers.
- **The `unsafe` block is the vouch point.** As in Rust, the programmer who writes `unsafe { }` vouches for what is inside it, wherever the block appears. The checker sees one function at a time, so a safe function that contains an `unsafe` block which assigns a static is as dangerous to a caller lending it that static (by `in`, `ref` or as a receiver) as an `unsafe` function is: the lent value is freed under the caller. Writing the block is taking on that responsibility.
- A static is a place: it can be passed by `ref`, and two `ref` arguments naming one static are an overlapping borrow.

### Owned statics

A static of an owned type (a class, a list, an array, or a struct or tuple that owns something) is always optional, since such a type has no constant and no default. The static owns its value for the whole program:

- assigning a new value drops the old one;
- moving out of it is an error (pass it with `in`, or assign it a new value);
- the program drops every owned static when `Main` returns, in reverse declaration order (static fields included), before the `NZ_ALLOC_REPORT=1` report. `nzc test` drops them after each test. An `Abort`, a failed `Assert` or another runtime stop drops nothing, as everywhere.

```nz output="0\n"
package main;

import std.io.Console;

class Node {
    int V;
}

static Node? head;

void Main(string[] args) {
    unsafe {
        Node n = new Node();
        n.V = 1;
        head = n;
        head = new Node();
        if (head is Node h) {
            Console.WriteLine($"{h.V}");
        }
    }
}
```

```nz error="cannot move out of static `head`"
package main;

class Node {
    int V;
}

static Node? head;

void Main(string[] args) {
    unsafe {
        Node? n = head;
    }
}
```

**Any call may replace a static.** A function the compiler does not see at this point could assign `head = null`, which drops the node. So while something an owned static holds is in use, a call that may run NZ code is checked. Such a call is any call of a function, method, local function or static method, and `new T()` when a field initializer of `T` calls one. Calls of the standard library (`Console`, `List`, `Diag`, `string`) and `Abort`/`Assert` are not.

- Inside a `foreach` over a collection an owned static holds, such a call is an error.
- A pattern variable that borrows from an owned static (`if (head is Node h)`) can no longer be used after such a call. Test the static again after the call.
- When a call already holds part of an owned static (an `in` argument, the receiver of `head?.M(...)`, a `ref` to a field inside it), a later argument with such a call is an error. A `ref` to the whole static is allowed: it passes the static's address, read when the callee runs.
- The same for an assignment into an object a static holds (`h.V = F();`) and for an index into a list a static holds (`l[F()]`).

These rules also refuse some safe code. The fix is the same each time: compute the value in a statement of its own first.

```nz error="before the value is stored"
package main;

class Node {
    int V;
}

static Node? head;

int Next() {
    return 2;
}

void Main(string[] args) {
    unsafe {
        head = new Node();
        if (head is Node h) {
            h.V = Next();
        }
    }
}
```

```nz output="2\n"
package main;

import std.io.Console;

class Node {
    int V;
}

static Node? head;

int Next() {
    return 2;
}

void Main(string[] args) {
    unsafe {
        head = new Node();
        int v = Next();
        if (head is Node h) {
            h.V = v;
            Console.WriteLine($"{h.V}");
        }
    }
}
```

**What `unsafe` vouches for.** The analysis checks one function at a time. A function that receives a lent static (`Count(head)` with `int Count(in Node? n)`) cannot free it from safe code: a write to the static, or a call of a function that may write one, needs `unsafe` (see above). Inside `unsafe` code the programmer vouches for these, as for Rust's `static mut`:

- an `unsafe` function or method that receives a static by `in` or `ref`, or as its receiver (`head?.M()`), and calls something that assigns that static;
- an `unsafe` method of an object an owned static holds that frees the static (`ee?.M()` where `M` sets `ee = null`);
- an `unsafe` callee given a field inside a static by `ref` (`Rf(h.Next)` where `Rf` frees the static).

The checks above still refuse every such call that is visible where the static is held.

## 1.10 `do ... while` and labeled loops

`do { ... } while (condition);` runs its body once before the first test. A loop (`while`, `do`, `for`, `foreach`) can carry a label, and `break label;` or `continue label;` then leaves or continues that loop from inside nested loops. Every `defer` and drop of the scopes left runs once.

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

## 1.11 `=>` bodies, `sizeof`, `nameof` and `default`

A function or method whose body is one `return` can be written `=> expression;`. `sizeof(T)` is the size in bytes of a built-in numeric type, `bool`, `char` or an enum, as an `int` constant (`bool` 1, `char` 4; an enum has the size of its underlying type, so 4 for a plain enum and 1 for `enum B : byte`). `nameof(a.b)` is the last name as a `string` constant; every part must exist and be visible, and naming something is not a use (a moved variable or a mutable static can be named). `default(T)` is the zero of `T`; a bare `default` takes the type of where it stands (a declared variable, `return`, `=`, an argument, a field initializer).

```nz output="16 0 8 4 X Green 0 0 True\n"
package main;

import std.io.Console;

enum Color { Red, Green }

class P {
    int X;
    public int Twice() => X * 2;
}

int Square(int n) => n * n;

void Main(string[] args) {
    P p = new P();
    int zero = default;
    long big = default(long);
    string? s = default;
    Console.WriteLine($"{Square(4)} {p.Twice()} {sizeof(long)} {sizeof(Color)} {nameof(p.X)} {nameof(Color.Green)} {zero} {big} {s == null}");
}
```

| Type | `default` |
|---|---|
| `int`, `long`, `float`, `double` | `0` |
| `bool` | `false` |
| `char` | the character with code 0 |
| `T?` | `null` |
| `string` | an error: write `""` |
| a class or struct `C` | an error: write `new C()` or use `C?` |
| an enum | an error: name one of its variants or use `E?` |
| a type parameter `T` | an error: `default(T)` needs a constraint, which is not supported yet |

In a constant, `nameof` can name only constants, and a bare `default` takes the constant's type. A bare `default` anywhere else (`var x = default;`, `1 + default`) needs a type: write `default(int)`.

## 1.12 Parameters and arguments

The examples in this section are NZ code (C# style): defaults, named arguments and extension methods follow C# except where noted.

**Default values.** A parameter can have a default value, as in C#; a call may leave it out. A default is a constant expression (see [1.8 Constants](#18-constants); `string` constants and the constants of the declaring type count), `null` for an optional parameter, `default`/`default(T)` where the type has a default, or a `float`, `double` or `char` literal (with a sign or casts: `-0.5`, `2F`, `'x'`, `(char)120`). Every parameter after one with a default needs a default too. A `ref`, `this` or `params` parameter has none; an `in` parameter may have one (the caller stores it in a temporary and passes its address). A default is stored with the function, so a call from another package uses the declaring package's constants.

**Named arguments.** An argument can name its parameter: `Sub(b: 1, a: 5)`. Positional arguments come first; a name may not repeat, nor name a parameter already given by position; every parameter without a default must receive a value.

```nz output="6 4 a, z a, c 1.5 x\n"
package main;

import std.io.Console;

const int Step = 1;

int Sub(int a, int b = Step) {
    return a - b;
}

string Join(string a, string sep = ", ", string b = "z") {
    return a + sep + b;
}

string Show(double x = 1.5, char c = 'x') {
    return $"{x} {c}";
}

void Main(string[] args) {
    Console.WriteLine($"{Sub(7)} {Sub(b: 1, a: 5)} {Join("a")} {Join("a", b: "c")} {Show()}");
}
```

Arguments are evaluated **in the order they are written**, whatever parameters they bind to (C#'s rule); moves and borrows follow the same order:

```nz output="ba 4\n"
package main;

import std.io.Console;

int Log(string s, int v) {
    Console.Write(s);
    return v;
}

int Sub(int a, int b) {
    return a - b;
}

void Main(string[] args) {
    Console.WriteLine($" {Sub(b: Log("b", 1), a: Log("a", 5))}");
}
```

```nz error="positional argument after a named argument"
package main;

int Sub(int a, int b) {
    return a - b;
}

void Main(string[] args) {
    int x = Sub(b: 1, 2);
}
```

C# 7.2 and later also accept a named argument in its own position followed by positional ones (`Sub(a: 1, 2)`); NZ refuses that too for now, for now: every positional argument comes before the first named one.

Named arguments on built-in members (`Console.WriteLine(value: x)`, `List.Add`), on `Abort` and `Assert`, and on `new` are not supported yet.

**Extension methods.** A top-level function whose first parameter is marked `this` can also be called as if it were a member of that parameter's type: `5.Double()` calls `Double(5)`. The receiver is evaluated first, then the arguments. On a copied type the receiver is a plain `this` parameter (a copy); on an owned type (a class, a list, an array, a struct that owns something, a type parameter) it must say how it borrows the receiver, `this in T` (read) or `this ref T` (write), so calling an extension never moves the receiver. A `this ref` receiver must be a variable, field or element, as any `ref` argument.

```nz output="20 hi!! 5 2 3\n"
package main;

import std.io.Console;
import std.collections.List;

int Double(this int x) {
    return x * 2;
}

string Exclaim(this string s, int times = 1) {
    string r = s;
    for (int i = 0; i < times; i++) {
        r = r + "!";
    }
    return r;
}

int Total(this in List<int> xs) {
    int t = 0;
    foreach (var x in xs) {
        t += x;
    }
    return t;
}

void Bump(this ref int x) {
    x++;
}

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(2);
    l.Add(3);
    int n = 1;
    n.Bump();
    n.Bump();
    Console.WriteLine($"{5.Double().Double()} {"hi".Exclaim(times: 2)} {l.Total()} {l.Count} {n}");
}
```

```nz error="must borrow its receiver"
package main;

import std.collections.List;

int Size(this List<int> xs) {
    return xs.Count;
}

void Main(string[] args) { }
```

How an extension is found: a member of the receiver's type always wins over an extension of the same name; otherwise the function must be visible in the file (declared in the package or imported) and its first parameter marked `this`. There is no overloading: one name names one function. The receiver's type must be exactly the extended type once type parameters are inferred: no numeric widening (`5.Twice()` does not reach `this long x`) and no `int` to `int?`; an optional variable narrowed by `!= null` still reaches an extension on its optional type. Through a `Shared<T>` or `Arc<T>` handle an extension on `T` is reached as a member of `T` is, lending the object for the call (`this ref` through an `Arc<T>` is refused: its object never changes). An extension through `?.` and an extension on a type name (`Color.F()`) are not supported yet. An extension can always be called as a plain function too (`Double(5)`).

`params` parameters wait for spans: ``not supported yet: `params` needs `Span<T>`, which arrives with spans``.

## 1.13 Unsafe code and C interop

Inside `unsafe`, NZ has C#'s systems features: pointers, `&`, `fixed`, `stackalloc`, manual memory, C layouts, calls into C and functions C can call. Everything else stays checked inside `unsafe`: types, definite assignment, null safety, integer overflow, array and list indexes, and the ownership of managed values (classes, strings, arrays, lists, owned structs). What `unsafe` turns on is listed in [Ownership](+/Ownership.md#pointers-and-fixed), with why managed values stay sound.

### Pointers

A pointer type is written as in C#: `int*`, `void*`, `Point**`. The type it points at must be **unmanaged**: an integer type, `float`, `double`, `bool`, `char`, an enum, a pointer, or a struct whose fields are all unmanaged. A pointer to a `string`, a class, an array, a `List<T>`, a type parameter, an optional or a tuple does not exist. (C# also counts `int?` and tuples of unmanaged types as unmanaged; NZ does not.)

The operators are C#'s: `&x`, `*p`, `p->F`, `p[i]`, `p + n`, `p - n`, `p - q` (a `long`), `p++`, `p--`, and the comparisons. `null` converts to every pointer type, and `T*` converts to `void*` without a cast. Casts are `(T*)p`, `(void*)p`, `(nint)p`, `(nuint)p`, and `(T*)n` from `nint`, `nuint`, `long` or `ulong`. Inside `unsafe`, `sizeof(T)` works for every unmanaged type.

Pointer arithmetic is not checked, `p[i]` has no bounds check, and nothing checks a dereference of `null`, as in C#. A pointer that outlives what it points at, a write past the end of a buffer, a misaligned read through a cast, a double `Free`: the compiler does not see these, and the program that writes `unsafe` vouches for them. Tests find them with the sanitizers (`NZ_CC` with `-fsanitize=address,undefined`).

A pointer never leaves `unsafe`. A local, parameter, result or field of pointer type needs an unsafe context; a struct or class with a pointer field is an `unsafe struct` or `unsafe class`; a function with a pointer in its signature is called only from `unsafe`. A pointer cannot be a type argument, an array element, a tuple element or optional. The one way out is an explicit `(nint)` or `(nuint)` cast, which is a plain number in safe code.

```nz error="can only be used inside `unsafe`"
package main;

void Main(string[] args) {
    int x = 1;
    int* p = &x;
}
```

### `&` and `fixed`

`&` takes the address of a local, a value parameter, a field of a struct local, or a mutable static. A field of a class object, an array element and a `ref` or `in` parameter live in storage an owner can free, so their address is taken with `fixed`, as C# requires for "moveable" variables:

```nz output="6 9\n"
package main;

import std.io.Console;

struct Point {
    public int X;
    public int Y;
}

void Main(string[] args) {
    int[] values = new int[3];
    int sum = 0;
    unsafe {
        fixed (int* p = values) {
            for (int i = 0; i < 3; i++) {
                p[i] = i + 1;
            }
        }
        Point pt = new Point();
        Point* q = &pt;
        q->Y = 9;
        int* s = &sum;
        *s = values[0] + values[1] + values[2];
        Console.WriteLine($"{sum} {pt.Y}");
    }
}
```

`fixed (int* p = arr)` points at the first element, `fixed (int* p = &obj.X)` at a field, and `fixed (byte* p = "abc"u8)` at a UTF-8 literal, as in C#. The variable is a pointer (C#'s CS0209), and an expression that is already a pointer is not fixed again (CS0213). For the block, the root object is **frozen**: moving it, assigning it, dropping it, passing it by `ref`, or calling a method that may replace what is fixed are errors, while reading and writing its fields stays allowed. The `fixed` pointer itself cannot be changed. Two limits: a second `Shared<T>` or `Arc<T>` handle to the same object is not frozen, and `fixed` on a `string` is not supported yet (C# gives a `char*` to UTF-16; NZ strings are UTF-8).

Unlike C#, `&` of a captured local is refused only inside the local function that captures it; the enclosing function may take it, which is sound because captures are addresses of stack storage that never moves. And `&` takes a mutable static directly, where C# wants `fixed`.

### `stackalloc`

`byte* buf = stackalloc byte[64];` and `byte* s = stackalloc byte[] { 104, 105, 0 };` allocate on the stack. As in C#, the buffer lives until the function returns, also when it is allocated in an inner block, and it is zeroed. The length must be a constant, and a `stackalloc` stands only as the initializer of a pointer local. Inside a loop each pass takes more stack, so the compiler warns, as C#'s analyzer rule CA2014 does:

```nz warning="`stackalloc` in a loop"
package main;

void Main(string[] args) {
    unsafe {
        for (int i = 0; i < 3; i++) {
            byte* p = stackalloc byte[16];
            p[0] = 1;
        }
    }
}
```

A buffer too large for the stack ends the program with a crash, with no `nz:` message, as a stack overflow ends a C# program.

### Manual memory: `NativeMemory`

`import std.interop.NativeMemory;` gives C#'s six functions, inside `unsafe`: `Alloc(nuint size)`, `AllocZeroed(nuint count, nuint size)`, `Realloc(void* p, nuint size)`, `Free(void* p)`, `Copy(void* source, void* destination, nuint size)` and `Clear(void* p, nuint size)`. Out of memory stops the program with `nz: out of memory` (C# throws). The runtime counts the blocks: `NZ_ALLOC_REPORT=1` prints `nz: live native blocks = N` before the owned-objects line, and `nzc test` fails a test that leaves a block, with both counts when it leaks both.

```nz output="104 105 30\n"
package main;

import std.io.Console;
import std.interop.NativeMemory;

void Main(string[] args) {
    unsafe {
        byte* text = stackalloc byte[] { 104, 105, 0 };
        long* numbers = (long*)NativeMemory.Alloc((nuint)(sizeof(long) * 3));
        for (int i = 0; i < 3; i++) {
            numbers[i] = (i + 1) * 5;
        }
        long total = numbers[0] + numbers[1] + numbers[2];
        NativeMemory.Free(numbers);
        Console.WriteLine($"{text[0]} {text[1]} {total}");
    }
}
```

### C layout: `[StructLayout]`

`[StructLayout(LayoutKind.Sequential)]` on a struct of unmanaged fields gives it C's layout: the fields in declaration order with C alignment. `Pack = n` (C#'s values: 0 for the natural layout, 1, 2, 4, 8, 16, 32, 64, 128) lowers the alignment. The generated C checks every offset with `_Static_assert(offsetof(...))`. A struct used in a C signature must carry the attribute. The attributes come from `std.interop` (`import std.interop.*;`), as C# needs `using System.Runtime.InteropServices;`.

```nz output="5 8\n"
package main;

import std.io.Console;
import std.interop.*;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Packed {
    public byte Tag;
    public int Value;
}

[StructLayout(LayoutKind.Sequential)]
struct Natural {
    public byte Tag;
    public int Value;
}

void Main(string[] args) {
    int packed = 0;
    int natural = 0;
    unsafe {
        packed = sizeof(Packed);
        natural = sizeof(Natural);
    }
    Console.WriteLine($"{packed} {natural}");
}
```

A field that `Pack` puts below its natural alignment is used by name only: its address would be a misaligned pointer, so `&`, `fixed`, a `ref` or `in` argument, a `this ref` receiver and a method call on it are not supported yet. `char` cannot be a field of a laid-out struct (NZ's `char` is 4 bytes, C#'s 2); use `byte`, `ushort` or `uint`. `LayoutKind.Explicit`, `[FieldOffset]`, `LayoutKind.Auto`, and the attribute on a class or a generic struct are not supported yet.

```nz error="not supported yet: the address of a packed field"
package main;

import std.interop.*;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Packed {
    public byte Tag;
    public int Value;
}

void Main(string[] args) {
    Packed p = new Packed();
    unsafe {
        int* v = &p.Value;
    }
}
```

### Calling C: `extern` and `[DllImport]`

A C function is declared at the top level with C#'s `[DllImport]` and `extern`, without `static` (top-level functions take none) and without a body. The first argument names the library: `"c"` and `"m"` add nothing, any other name links `-l<name>`; `nzc build`, `run` and `test` take `-L <dir>` (repeatable) for a library outside the system folders. `EntryPoint = "name"` gives the C name when it differs. Every call is written inside `unsafe`, even without a pointer in the signature: this is stricter than C#, because nothing checks the C side.

```nz output="5 4\n"
package main;

import std.io.Console;
import std.interop.*;

[DllImport("c")]
extern int abs(int value);

[DllImport("m", EntryPoint = "sqrt")]
extern double SquareRoot(double x);

void Main(string[] args) {
    int a = 0;
    double r = 0;
    unsafe {
        a = abs(-5);
        r = SquareRoot(16.0);
    }
    Console.WriteLine($"{a} {r}");
}
```

```nz error="calling unsafe function `abs` needs an unsafe context"
package main;

import std.interop.*;

[DllImport("c")]
extern int abs(int value);

void Main(string[] args) {
    int a = abs(-5);
}
```

A C signature takes the integer types, `float`, `double`, `bool`, pointers, enums (as their underlying type) and `[StructLayout]` structs by value or by pointer, and `void` as a result; `ref` and `in` are passed as pointers. Not `string`, classes, arrays, optionals, tuples, or `char` (use `byte` or `uint` for a C `char`), and no variable argument list. Two sizes differ from what C# marshals by default: NZ `bool` is C's one-byte `bool` (C#'s default is the four-byte Win32 `BOOL`; NZ matches `[MarshalAs(UnmanagedType.U1)]`), and NZ `long` is always 64 bits, while C's `long` is 32 bits on Windows (C# has `CLong` for it).

Names are checked because they are written into the generated C and the link line. A C name must be a C identifier, not a C keyword and not `main`; names the generated code or the runtime use are refused. A library must be a plain name (letters, digits, `_`, `.`, `+`, `-`): C# also takes a path, which is not supported yet. The C name is given to the linker as is, which works on Linux and Windows x64; a target that prefixes C symbols with `_` (macOS, 32-bit Windows) is not supported yet.

### Functions C can call: `[UnmanagedCallersOnly]`

`[UnmanagedCallersOnly(EntryPoint = "add_i32")] int AddI32(int a, int b) { ... }` at the top level is a C function of exactly that name. `EntryPoint` is required; the signature follows the rules above, without `ref` or `in` (C#'s CS8977: take a pointer); NZ code cannot call it, as in C#. An export has external linkage and would replace a C function of the same name for the whole program, so its name may not be one C keeps:
- a function of the C library (`exit`, `malloc`, `fmod`, `sqrt` and its `f` and `l` forms), or a name C11 keeps for it (`str`, `mem`, `wcs`, `is` and `to` followed by a lowercase letter, `atomic_`);
- a name that starts with `_`;
- a macro or type of the C headers (`EOF`, `stdout`, any all-capitals name, any name ending in `_t`);
- a POSIX or start-up name (`bcmp`, `write`, `pthread_...`);
- a name of the NZ runtime (`nz_...`, `NZ...`).

A `[DllImport]` may name any of them: importing them is what it is for.

```nz
package main;

import std.interop.*;

[UnmanagedCallersOnly(EntryPoint = "add_i32")]
int AddI32(int a, int b) {
    return a + b;
}

void Main(string[] args) {
}
```

`nzc build file.nz --emit-c-part out.h` writes, instead of an executable, the exports and what they need as a C header: every function `static inline` and marked unused, no `main`, no `#include "nz.h"`. It is included after the runtime header and compiles under `-Wall -Wextra -Werror`. A part is C for clang or GCC (also `clang-cl`), not for MSVC's `cl.exe`: it uses `__attribute__((unused))`, `__asm__` labels and `_Static_assert`. A part works after `nz_core.h` alone or after `nz.h`; an enum's name function, which returns a `string`, is in it only after `nz.h`. Its C names carry the package path (`T_2_4main1P` is the struct `P` of package `main`), so two parts of different packages can be included together, and two parts of the same package name clash: give each part its own package. A part built with `--no-runtime` may supply `nz_panic`: it is then an ordinary external function, and the C file that includes the part defines `NZ_USER_PANIC` before `nz_core.h`.

### Without the runtime: `--no-runtime`

`nzc check`, `nzc build` and `nzc run` take `--no-runtime`: the program links the C library only, and its C includes `nz_core.h` (checked arithmetic and its failure path, optionals of the built-in types, `NativeMemory`) instead of `nz.h`. The entry point is `void Main()` or `int Main()` (as in C#; the `int` is the exit status); `string[] args` needs the runtime. A `string` value or interpolation, `Console`, `Diag`, a class (also through its static members), an array, a `List<T>` or a handle, `Abort` and `Assert` are refused, each with ``needs the NZ runtime; not available with --no-runtime``. Numbers, `bool`, `char`, enums, structs of those, pointers, statics, local functions, `extern` calls, `stackalloc` and `NativeMemory` work, and integer arithmetic stays checked: an overflow prints `nz: integer overflow` and exits with 1.

Text for C is a UTF-8 literal, `"hi\n"u8` (or `U8`, also after a raw `"""..."""`). A raw literal that spans lines holds the source's line breaks, which the repository keeps as `\n` on every system (`.gitattributes`). Its type in NZ is `byte*`: a pointer to static, read-only bytes ending with a NUL, usable only inside `unsafe`, with or without the runtime. This **differs from C#**, where the type is `ReadOnlySpan<byte>`. The bytes are read-only, as a `ReadOnlySpan<byte>` is: writing through the pointer is undefined and usually crashes.

A program may supply the failure path itself: `[UnmanagedCallersOnly(EntryPoint = "nz_panic")] unsafe void Panic(byte* message)`, only with `--no-runtime`, once, and with exactly that signature. `message` points at read-only text, as a `u8` literal does. If the function returns, or fails itself (an overflow inside it), the program stops with exit status 1.

```text
package main;

import std.interop.*;

[DllImport("c")]
unsafe extern int puts(byte* s);

int Main() {
    unsafe {
        puts("hello from C"u8);
    }
    return 3;
}
```

`nzc run --no-runtime hello.nz` prints the line and exits with status 3.

### Not supported yet

Each of these is refused with a message that says so: function pointers (`delegate*`), `LayoutKind.Explicit` and `[FieldOffset]`, a variable `stackalloc` length, `volatile` fields, pointers as type arguments, array elements, tuple elements or optional values, a C function with a variable argument list (C's `...`), `[LibraryImport]` (C# generates its code; use `[DllImport]`), `fixed` on a `string`, `CallingConvention` and C#'s other `DllImport` fields (the calling convention is always C's), and a library given as a path.

# 2. Types

## 2.1 Class & Struct

A `class` is a heap object with a **single owner** (move-only); a `struct` is a value type that is copied. An object is freed when its owner goes out of scope.

```nz output="1\n"
package main;

import std.io.Console;

class Counter {
    int count;
    string label = "counter";

    int Next() {
        count += 1;
        return count;
    }
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.Next();
    Console.WriteLine($"{n}");
}
```

Assigning an object to another variable, or passing it to a parameter without a modifier, **moves** it: the new owner has it, and the old variable cannot be used again until it is given a new value. Passing an object somewhere and then using it again is rejected:

```nz error="moved"
package main;

import std.io.Console;

class Counter {
    int Count;
}

void Show(Counter c) {
    Console.WriteLine($"{c.Count}");
}

void Main(string[] args) {
    var c = new Counter();
    Show(c);
    Show(c);
}
```

To keep the object, lend it instead: `in` lends it for reading and `ref` lends it to be changed. The caller still owns it after the call. As in C#, `in` is a read-only reference for every type, value types included, not a copy, except `in string` and `in string?`, which hold a copy of their own (see [Ownership: Strings](+/Ownership.md#strings)) ([Ownership](+/Ownership.md#borrowing-in-and-ref)):

```nz output="0\n1\n"
package main;

import std.io.Console;

class Counter {
    int Count;
}

void Show(in Counter c) {
    Console.WriteLine($"{c.Count}");
}

void Bump(ref Counter c) {
    c.Count += 1;
}

void Main(string[] args) {
    var c = new Counter();
    Show(c);
    Bump(c);
    Show(c);
}
```

A field or property needs a value: **an initial value, an optional type, or an assignment in every constructor**. Numbers, `bool` and `char` are zero automatically; other types are not:

```nz error="needs a value"
package main;

class User {
    string Name;
}

void Main(string[] args) { }
```

```nz
package main;

class User {
    string Name = "no name";
    string? Note;
    int Age;
}

void Main(string[] args) { }
```

### Constructors

A **constructor** is a member named after its type, with no return type, as in C#. A type may declare several; `new` picks the one whose parameters take the arguments (named and default arguments work as in calls), and when several do, the one whose parameter types equal the argument types. A type that declares a constructor has only the ones it declares; a type that declares none has the implicit `new T()`. A constructor is as visible as its type unless it is written `private`.

```nz output="Ada 36\nnobody 0\n"
package main;

import std.io.Console;

class Person {
    string Name;
    int Age;

    Person(string name, int age) {
        Name = name;
        Age = age;
    }

    Person() : this("nobody", 0) { }
}

void Main(string[] args) {
    var a = new Person("Ada", 36);
    var b = new Person();
    Console.WriteLine($"{a.Name} {a.Age}");
    Console.WriteLine($"{b.Name} {b.Age}");
}
```

`: this(...)` runs another constructor of the same type first. A field without an initial value whose type has no default (`Name` above) must be assigned on **every path** of every constructor that does not chain, before it is read and before `this` is used (a method call, `this` passed as a value). A local function called, or a `defer` that runs, while such a field is unassigned counts as a read. The arguments of `: this(...)` run before any constructor, so they may use the parameters and constants but no member of the object and no `this` (as in C#). A constructor that chains starts with every field assigned:

```nz error="is not assigned on every path"
package main;

class Token {
    string Text;

    Token(bool empty) {
        if (!empty) {
            Text = "x";
        }
    }
}

void Main(string[] args) { }
```

Creating an object runs, in order: the arguments of `new` from left to right, the field initializers (once, in the constructor that does not chain), the constructor bodies (a chained one first), then the object initializer.

A **primary constructor** (C# 12) writes the parameters after the type's name; the field initializers read them. Every other constructor of the type chains to it with `: this(...)`. Using a primary-constructor parameter in a method (which C# allows) is not supported yet: store it in a field.

```nz output="sum 3\n"
package main;

import std.io.Console;

class Labelled(string label, int a, int b) {
    string Label = label;
    int Total = a + b;
}

void Main(string[] args) {
    var l = new Labelled("sum", 1, 2);
    Console.WriteLine($"{l.Label} {l.Total}");
}
```

### Object initializers

`new T(args) { A = a, B = b }` sets fields and settable properties after the constructor ran, in the order written; `new T { ... }` uses the parameterless constructor. A member is set once per initializer.

```nz output="3 4\n"
package main;

import std.io.Console;

class Point {
    int X;
    int Y;
}

void Main(string[] args) {
    var p = new Point { X = 3, Y = 4 };
    Console.WriteLine($"{p.X} {p.Y}");
}
```

A `required` member (C# 11) is set by the object initializer of every `new`; it has no initial value. A `readonly` field is assigned only by its initial value and by the constructors of its type (through `F` or `this.F`), and never by an object initializer. The parts of a `readonly` struct field are the field: they are not assigned or passed by `ref` elsewhere either. A method called on a `readonly` struct field outside those constructors runs on a copy, as in C#, so the field does not change; a struct that owns an object (a class, a list) cannot be copied, so a method that changes it is refused there:

```nz error="must set the required member"
package main;

class User {
    required string Id;
    readonly int Created;

    User() {
        Created = 1;
    }
}

void Main(string[] args) {
    var u = new User();
}
```

**Properties** use `get` and `set`; a property without `set` cannot be assigned:

```nz
package main;

class Item {
    int Qty { get; set; }
    string? Note { get; set; }
}

void Main(string[] args) { }
```

As in C#, an **auto-property** (`{ get; set; }`, `{ get; }`, `{ get; init; }`) stores its value like a field and may have an initializer (`= value;`). One difference from C#: a struct auto-property is a variable in NZ, so `c.P.X = 1` changes the stored struct, where C# refuses it (CS1612) because a C# property always returns a copy; with a computed property NZ refuses it as C# does. A get-only auto-property is set by its initializer or in a constructor of its type; an `init` property also by an object initializer. One accessor may be `private` (`{ get; private set; }`): outside the type the property is then read-only.

```nz output="Ann 3 1\n"
package main;
import std.io.Console;

public class Account {
    public string Owner { get; }
    public int Visits { get; private set; } = 2;
    public int Id { get; init; }

    public Account(string owner) {
        Owner = owner;
    }

    public void Visit() {
        Visits++;
    }
}

void Main(string[] args) {
    var a = new Account("Ann") { Id = 1 };
    a.Visit();
    Console.WriteLine($"{a.Owner} {a.Visits} {a.Id}");
}
```

A **computed property** has accessors with bodies: `get { ... }` returns the value, `set { ... }` (or `init { ... }`) receives it as `value`. It has no storage of its own. `=> expr` is a getter written as one expression, for the whole property (`int Area => W * H;`) or for one accessor (`get => n;`). Accessors are all bodies or all `;`, never mixed.

```nz output="Temp 20 68 21\n"
package main;
import std.io.Console;

class Thermometer {
    double celsius;

    public double Celsius {
        get { return celsius; }
        set {
            if (value < -273.15) {
                return;
            }
            celsius = value;
        }
    }

    public double Fahrenheit => celsius * 9 / 5 + 32;
}

void Main(string[] args) {
    var t = new Thermometer();
    t.Celsius = 20;
    t.Celsius = -300;
    double f = t.Fahrenheit;
    t.Celsius += 1;
    Console.WriteLine($"Temp 20 {f} {t.Celsius}");
}
```

Reading a computed property calls its getter, assigning calls its setter, and `+=`, `++` and the other compound forms call the getter once and the setter once. As in C#, the receiver and the index of a compound write are evaluated once, first, then the getter, the value and the setter; a value that could change the receiver's variable or field before the setter runs (`c.P += Bump(ref c)`) is refused for now. A computed property is not a variable: it cannot be passed by `ref`, a setter cannot change a struct value that is not a variable (a call result, a list element, another property's value), and a field of a struct value that a getter, an indexer or an operator gave cannot be assigned (`c.P.X = 1`, `g[0].X++`: the change would be lost; C# CS1612), as in C#:

```nz error="not a variable"
package main;

struct Size {
    int w;
    public int W { get { return w; } set { w = value; } }
}

Size Make() { return new Size(); }

void Main(string[] args) {
    Make().W = 3;
}
```

Not supported yet: using the value of an assignment, a compound assignment or `++`/`--` through a computed property inside a larger expression (write it as a statement); a compound assignment whose value may change the receiver's variable or field before the setter runs (`c.P += Bump(c)` where `Bump` takes `ref C`, or any call when the receiver is reached through a `Shared<T>` handle; compute the value into a variable first); a computed property set in an object initializer or marked `required`; a property (computed or auto) inside a property pattern (`c is { P: > 0 }`: property patterns read fields; test `c.P is > 0` instead, which calls the getter once); static properties; accessor bodies in an interface.

An **indexer** lets a class or struct be indexed like an array: `T this[P p] { get { ... } set { ... } }`, as in C#. Its accessors need bodies (it has no storage), the setter receives `value`, and it is written `public` or `private`. A user indexer does not make a type usable in `foreach`.

```nz output="16 0 n3\n"
package main;
import std.io.Console;

class Grid {
    int[] cells = new int[4];

    public int this[int i] {
        get { return cells[i]; }
        set { cells[i] = value; }
    }
}

class Names {
    public string this[int i] => $"n{i}";
}

void Main(string[] args) {
    var g = new Grid();
    g[1] = 5;
    g[1] += 2;
    g[1]++;
    g[1] *= 2;
    var n = new Names();
    Console.WriteLine($"{g[1]} {g[0]} {n[3]}");
}
```

**Operators** can be declared by a class or struct, as in C#: `public static T operator +(A a, B b)`. The operators that can be declared are `+ - ! ~ ++ --` (unary) and `+ - * / % == != < > <= >= & | ^ << >>` (binary); `==` and `!=`, `<` and `>`, `<=` and `>=` are declared in pairs, at least one parameter has the declaring type, and `++`/`--` take and return it. The operands are lent with `in` and never moved; the result is a new value. `a += b` uses `+`, and `a == null` stays the null test.

```nz output="8 6 -6 False True\n"
package main;
import std.io.Console;

struct Vec {
    public int X;

    public static Vec operator +(Vec a, Vec b) => Make(a.X + b.X);
    public static Vec operator *(Vec a, int k) => Make(a.X * k);
    public static Vec operator -(Vec a) => Make(-a.X);
    public static Vec operator ++(Vec a) => Make(a.X + 1);
    public static bool operator ==(Vec a, Vec b) => a.X == b.X;
    public static bool operator !=(Vec a, Vec b) => a.X != b.X;

    public static Vec Make(int x) {
        var v = new Vec();
        v.X = x;
        return v;
    }
}

void Main(string[] args) {
    var a = Vec.Make(2);
    var b = Vec.Make(3);
    var c = a + b * 2;
    a += b;
    a++;
    Console.WriteLine($"{c.X} {a.X} {(-a).X} {a == b} {a != b}");
}
```

Not supported yet: an indexer with several parameters, more than one indexer or more than one operator with the same symbol in a type, an indexer written without `public` or `private`, an indexer or operator in an interface, operators of a generic type, conversion operators (`implicit`, `explicit`) and `true`/`false`, and using the value of a user `++` or `--` (write it as a statement).

### Nested types

A `class`, `struct`, `record`, `enum` or `union` may be declared inside a class, struct, record or union (C#'s nested types). Inside the enclosing type its bare name is enough; from outside it is written `Outer.Inner`, as a type and in expressions (`new Outer.Inner()`, `Outer.Inner.Make()`, `Outer.Mode.Fast`). A nested type is an ordinary type: it has its own members, constructors and static members, it is owned, copied and dropped like a top-level one, and it may be nested further.

```nz output="1 2\n"
package main;

import std.io.Console;

public class Shape {
    public class Point {
        public int X = 1;
        public int Y = 2;
    }

    public Point Origin() { return new Point(); }
}

void Main(string[] args) {
    var p = new Shape().Origin();
    Shape.Point q = p;
    Console.WriteLine($"{q.X} {q.Y}");
}
```

A nested type is **private to its enclosing type** unless it is written `public`: only the enclosing type, and the types declared inside it, can name it. A nested type can also use every private member of the types around it, as in C#; the enclosing type cannot use the private members of a nested type. A type name written inside the enclosing type finds the nested types before the top-level ones, so a nested `Node` hides a top-level `Node` there. Two enclosing types may each declare a nested type of the same name.

```nz error="nested type `Hidden` is private to `Outer`"
package main;

class Outer {
    class Hidden { }
}

void Main(string[] args) {
    var h = new Outer.Hidden();
}
```

```nz output="9\n"
package main;

import std.io.Console;

class Outer {
    int secret = 9;

    public class Peek {
        public int Read(in Outer o) { return o.secret; }
    }
}

void Main(string[] args) {
    Console.WriteLine($"{new Outer.Peek().Read(new Outer())}");
}
```

The nested type of a **generic** type takes the type parameters of the enclosing type first: `Box<int>.Cell` and `Box<string>.Cell` are different types, and inside `Box<T>` the name `Cell` means `Box<T>.Cell`. A nested type may declare parameters of its own (`Outer<int>.Pair<string>`).

```nz output="12\n"
package main;

import std.io.Console;

public class Box<T> {
    public class Cell {
        public T? Value;
    }

    public Cell C = new Cell();
}

void Main(string[] args) {
    var b = new Box<int>();
    b.C.Value = 12;
    Console.WriteLine($"{b.C.Value}");
}
```

A dotted path written inside the generic type gets the same arguments (`I.K` inside `O<T>` is `O<T>.I.K`), and a nested type is reached through an alias of an instantiated type (`type A = O<int>;`, `A.I`).

```nz output="3 5\n"
package main;

import std.io.Console;

class O<T> {
    public class I {
        public class K { public int V = 3; }
        public T? A;
    }

    public int Get() { var k = new I.K(); return k.V; }
}

type Alias = O<int>;

void Main(string[] args) {
    var i = new Alias.I();
    i.A = 5;
    Console.WriteLine($"{new O<string>().Get()} {i.A}");
}
```

A bare name inside a nested type finds the member of the nearest enclosing type that has that name. A static member (a constant, a static field or method) is used as it stands; an instance member there is an error (C# CS0120), because a nested type has no object of its enclosing type of its own. A nested enum may use the constants of its enclosing type by their bare names.

An `import` names a top-level declaration only: import `geo.Shape` and write `Shape.Point`. Importing `geo.Shape.Point` is an error that says so. A nested type cannot have the name of its enclosing type, of a member of it, or of a type parameter of the types around it. A package whose path equals the path of a type that has nested types is an error (the path `a.b.C` would name two things). In the generated C a nested type has its own name, built from its whole path, so it never meets a top-level type such as `Outer_Inner`.

Not supported yet: a type nested in an interface, a `static` nested type (there are no static classes), an enum nested in a generic type, and static members of a type nested in a generic type (a static member of a generic type is not supported at all yet).

## 2.2 Interface & Enum

```nz
package main;

enum Color { Red, Green, Blue }

interface IAnimal {
    void Sound();
}

void Main(string[] args) {
    var c = Color.Red;
}
```

> Note: an `interface` can only be declared so far; the syntax for implementing it (`class A : IAnimal`) does not exist yet.

**Enums with an underlying type and values.** An enum is stored as an integer, `int` unless written otherwise: `enum Level : byte { ... }` accepts any of `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long` and `ulong` (not `nint`, `nuint`, `char` or `bool`). A member without a value is the previous one plus one, starting at 0. A value is a constant expression of integers that fits the underlying type, as in C#. It may name any member of the same enum, before or after it (only a cycle is an error, and the message names the members), and a member of another enum only through a cast, as `(int)Level.High`. Two members may share a value. An enum is copied, never moved. An enum value prints as its member name (the first member with that value), or as the number when no member has it.

```nz output="Green High 11 All 3 A 11 X 2\n"
package main;

import std.io.Console;

enum Color { Red, Green, Blue }
enum Level : byte { Low = 1, Mid = 10, High }
enum Perm : uint { None = 0, Read = 1, Write = 2, All = Read | Write }
enum Fwd { A = B + 1, B = 10 }
enum Other : long { X = (long)Fwd.B * 1000000000 }
enum Mix { P = (int)Level.High, Q = (int)Perm.Write }

void Main(string[] args) {
    Level l = Level.High;
    Perm p = Perm.Read | Perm.Write;
    Console.WriteLine($"{Color.Green} {l} {(int)l} {p} {(uint)p} {Fwd.A} {(int)Fwd.A} {Other.X} {(int)Mix.Q}");
}
```

```nz error="the values of `A` and `B` depend on each other"
package main;

enum E { A = B, B = A }

void Main(string[] args) { }
```

```nz error="the value after `A` does not fit `byte`"
package main;

enum E : byte { A = 255, B }

void Main(string[] args) { }
```

```nz error="cannot implicitly convert `A` to `long`; write `(long)A.Y`"
package main;

enum A { X = 1, Y = 2 }
enum C : long { R = A.Y }

void Main(string[] args) { }
```

**Operators on enums** follow C#. `|`, `&`, `^` and `~` on one enum give that enum; `e + n`, `n + e` and `e - n`, with `n` an integer that fits the underlying type, give the enum; `e1 - e2` gives the underlying type; `++`, `--` and the compound `+= -= |= &= ^=` work on an enum variable; `== != < <= > >=` compare two values of one enum. An integer variable added to an enum must convert implicitly to the underlying type: `Day + int` works for an `int` (or `long`) enum, while an `int` variable added to an enum of `sbyte`, `byte`, `short`, `ushort`, `uint` or `ulong` is an error (a constant that fits works, as `Perm.Read + 2` below). Arithmetic is checked in the underlying type, so a result outside it stops the program: with ``nz: integer overflow`` for an `int`, `uint`, `long` or `ulong` enum, and with ``nz: value does not fit in `Perm` `` for the four smaller ones; `unchecked` wraps. Anything else (`+` on two enums, comparing two different enums) is an error. A format such as `{p:X}` needs an integer, so it is refused: cast first.

```nz output="Exec 3 3 3 3 3 Write True True True 255 True\n"
package main;

import std.io.Console;

enum Perm : byte { None = 0, Read = 1, Write = 2, Exec = 4 }

void Main(string[] args) {
    Perm p = Perm.Read | Perm.Write;
    p |= Perm.Exec;
    p &= ~Perm.Write;
    p ^= Perm.Read;
    Perm q = Perm.Read + 2;
    Perm r = 2 + Perm.Read;
    Perm s = Perm.Exec - 1;
    byte diff = Perm.Exec - Perm.Read;
    var d2 = Perm.Exec - Perm.Read;
    Perm t = Perm.None;
    t++;
    t++;
    Console.WriteLine($"{p} {q} {r} {s} {diff} {d2} {t} {Perm.Read < Perm.Write} {Perm.Read == Perm.Read} {Perm.Exec >= Perm.Write} {~Perm.None} {(Perm.Read | Perm.Write) == (Perm)3}");
}
```

```nz error="cannot apply `+` to `A` and `A`"
package main;

enum A { X }

void Main(string[] args) {
    A a = A.X;
    var r = a + a;
}
```

**Enum casts.** An enum casts to and from any integer type, to and from `char`, `float` and `double`, and to another enum, always through the underlying type. A cast into an enum checks the underlying range, whatever the value means: `(Level)7` is valid even though no member is 7, and prints `7`; a real is truncated first. A constant that does not fit is a compile-time error, a variable stops the program naming the enum, and `unchecked` keeps the low bits of an integer (a real stays checked). Without a cast an enum does not convert to or from an integer: `int i = Level.High;` is an error.

```nz output="200 1 66 200 3 Q\n"
package main;

import std.io.Console;

enum A : byte { X = 1, Y = 200 }
enum B : int { P = 1, Q = 2 }

void Main(string[] args) {
    A a = A.Y;
    B b = (B)a;
    char c = (char)A.X;
    A fromC = (A)'B';
    double d = (double)A.Y;
    A fromD = (A)3.7;
    Console.WriteLine($"{b} {(int)c} {fromC} {d} {fromD} {(B)2.9}");
}
```

```nz error="constant `300` does not fit `A`"
package main;

enum A : byte { X = 1, Y = 200 }

void Main(string[] args) {
    A a = (A)300;
}
```

A `switch` on an enum compares by value, so two arms for members that share a value are an error (``pattern `E.B` is already handled by an earlier arm``), and an enum member is not a pattern for an integer scrutinee.

## 2.3 Generics

```nz
package main;

class Box<T> {
    T? content;
}

void Main(string[] args) { }
```

A generic function call infers the type arguments from its arguments. When nothing can be inferred, write them out: `Empty<int>()`. Here `Empty` has no parameter that mentions `T`, so `Empty()` alone is an error (the message is ``cannot infer the type argument `T` of `Empty` ``).

```nz output="empty\n"
package main;

import std.io.Console;

T? Empty<T>() {
    return null;
}

void Main(string[] args) {
    int? a = Empty<int>();
    if (a == null) {
        Console.WriteLine("empty");
    }
}
```

Explicit type arguments work on a function (`Empty<int>()`) and on a method (`x.First<int>(1)`, `x?.First<int>(1)`). After a name or a member access, a `<` starts a type-argument list only when the matching `>` is followed by one of `(` `)` `]` `}` `:` `;` `,` `.` `?` `==` `!=` `|` `^` `&&` `||`. Otherwise it is the less-than operator. Type arguments must be followed by a call: `Empty<int>` alone is an error. So `a < b > (c)` is a generic call, and `(a) < b > (c)` or `f(x) < b > (c)` are comparisons because the left side is not a plain name.

## 2.4 Visibility

The **first letter of the name** decides the visibility of functions, methods, fields and properties:

| Spelling | Meaning |
|---|---|
| `Add` | visible from outside |
| `add` or `_add` | local |

The keywords `public` and `private` may be written as well. An explicit keyword wins over the letter, except that `public` on a name that starts with a lowercase letter is an error.

A `class`, `struct`, `interface` or `enum` is private (visible inside its package) unless it is written `public`. Local variables are not affected.

```nz error="must start with an uppercase letter"
package main;

public int add() { return 1; }

void Main(string[] args) { }
```

```nz error="local"
package main;

class Counter {
    int count;
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.count;
}
```

```nz
package main;

class Counter {
    int Count;
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.Count;
}
```

## 2.5 Attributes & Tests

An **attribute** is a note in square brackets in front of a declaration. It is compile-time data. The compiler knows `[Test]` (below) and the interop attributes of [1.13](#113-unsafe-code-and-c-interop), `StructLayout`, `FieldOffset`, `DllImport` and `UnmanagedCallersOnly`, which come from `std.interop` and need `import std.interop.*;` or an import of the name, as C# needs its `using`. As in C#, they may also be written with the `Attribute` suffix (`[DllImportAttribute(...)]`) or with the whole name (`[std.interop.DllImport(...)]`, which needs no import). Each built-in attribute is given once per declaration, only where it applies (C#'s CS0579 and CS0592). Any other name is accepted by the parser, kept in the syntax tree (`nzc parse` shows it), warned about, and read by nothing yet.

**Where.** Before a top-level function, `class`, `struct`, `interface` or `enum`, and before a member: a method, a field, a property or an interface member. After the `///` comment of the declaration and before `public` or `private`. Anywhere else is a syntax error: on an enum variant, a parameter, a statement. A local function gets its own message:

```nz fragment error="attributes on local functions are not supported yet"
[Test] int Twice(int x) {
    return x * 2;
}
Console.WriteLine($"{Twice(2)}");
```

**Lists.** One list holds several attributes, `[A, B]`, and several lists can follow each other, `[A] [B]`. An empty list `[]` and a trailing comma in a list are errors. A name may be qualified, `[geo.Tag]`, which is never the built-in `Test`.

**Arguments.** `[Route("/users", method: "GET")]`: arguments are positional or named, and the two may be mixed. As in C#, a named argument has two forms: `name: value` names a constructor parameter and `Name = value` sets a field or property. The built-in attributes say which is which: `EntryPoint` and `Pack` are fields, so `EntryPoint = "puts"` is right and `EntryPoint: "puts"` is an error; a named argument is given once (C#'s CS0643). Each argument is a literal (a number may have a sign), a constant expression of `int`, `long` and `bool` values such as `60 * 60`, or, for `StructLayout`, a member of `LayoutKind`. A call, a variable, a string built with `+` or a real-number expression is an error.

**Targets.** `[return: Tag]` names what the attribute is for. The target is parsed, then refused: `attribute targets are not supported yet`.

```nz error="attribute targets are not supported yet"
package main;

[return: Tag]
string Name() { return "n"; }

void Main(string[] args) { }
```

**Unknown names.** A name that is not a built-in attribute (`Test`, and the interop attributes `StructLayout`, `FieldOffset`, `DllImport` and `UnmanagedCallersOnly`, which, as in C#, need their import, `import std.interop.*;`, or their whole name, `[std.interop.StructLayout(...)]`; C#'s `Attribute` suffix may be written) is a warning, `unknown attribute`, and `-Werror` turns it into an error. The warning is there because the attribute has no effect yet; how attributes will be read is still open.

```nz warning="unknown attribute"
package main;

[Route("/users", method: "GET"), Owner("web")]
[Cache(60 * 60)]
class UsersController {
    [Column("id")]
    int Id;
}

void Main(string[] args) { }
```

### Tests

`[Test]` marks a function as a test. The function must be a top-level `void name()`: no parameters, no type parameters, no result, and not `Main`. `[Test]` takes no arguments, can be written once per function, and cannot be put on a method, a type or any other declaration. Each of these is an error.

```nz error="a test must be"
package main;

[Test]
int Count() { return 1; }

void Main(string[] args) { }
```

A test is an ordinary function. It passes when it returns, and fails when it stops through `Assert` or `Abort` (see 1.7). It calls the code under test and checks the result with `Assert`. This program checks with `nzc check` as it is, and its test also runs with `nzc test`:

```nz
package main;

int Add(int a, int b) {
    return a + b;
}

[Test]
void AddsNumbers() {
    Assert(Add(2, 3) == 5, "2 + 3");
}

[Test]
void AddsNegatives() {
    Assert(Add(-2, -3) == -5, "-2 + -3");
}

void Main(string[] args) { }
```

`nzc check`, `nzc build` and `nzc run` check the tests like any code, so an error in a test body is reported, but the tests are left out of the executable: a normal build contains neither a `[Test]` function nor a top-level function that only tests call, unless other code calls it.

**`nzc test <file.nz|folder> [--release] [-Werror]`** checks the program, builds an executable that holds only the tests, and runs each test in its own process, one after the other, in the order they are written. `Main` is not needed (and not run). Only the tests of the entry package are run; a `[Test]` in an imported package is checked but not run. A test has no name filter and no time limit, and tests do not run in parallel.

One process per test is deliberate: `Assert` and `Abort` end the process without cleanup, so a failing test ends only itself and the next one still runs. The report goes to standard output, one line per test (`test AddsNumbers ... ok`, `test AddsNegatives ... FAILED`), then a block for each failure with the exit code and what the test printed, then a last line `test result: 1 passed, 1 failed`. The output of a passing test is not shown. A failed `Assert` has exit code 1 and the message `nz: assertion failed: <message>`. A crash is shown by its exit code too; on Windows it is in hex, such as `exit code 0xC0000005`. The exit code of `nzc test` is 0 when every test passed, and 1 when one failed or the program has errors. With no `[Test]` at all it prints `nzc: no tests found` on standard error and exits with 0. `-Werror` works as for `check`.

**The leak rule.** The runner starts each test with `NZ_ALLOC_REPORT=1`. A test that returns normally but leaves owned objects or computed strings alive is a failure: `test Name ... FAILED (leaked 1 owned object(s))`, `(leaked 2 string(s))`, or both. Owned values are dropped when their scope ends and strings are freed with their last holder, so a correct program does not trip this rule; it is a safety net for the compiler itself (a cycle of `Shared<T>` handles is the one way to leak, see [Ownership](+/Ownership.md#shared-objects)). String literals never count. The test's own `defer` statements and drops run before the report, so they count as cleanup.

Not available yet: a test name filter, `[Test]` on methods, set-up and tear-down hooks, a time limit for a test, and an `Assert` that lets the test go on.

## 2.6 Tuples

A **tuple** groups two or more values of any types into one value: `(int, string)`. It is written as a type between parentheses, and as an expression with commas, `(7, "seven")`. A tuple has at least two elements and no upper limit; `(int)` is not a tuple and `()` is not a type. An element can itself be a tuple.

**Element access.** The elements are `Item1`, `Item2`, and so on, counted from 1. A tuple type may also name its elements, `(int Lo, int Hi)`, and then `r.Lo` and `r.Item1` are the same element. The names belong to the written type only, never to the value: `(int x, int y)`, `(int a, int b)` and `(int, int)` are **one type**, so they are assigned to each other freely, and `var p = r;` keeps the names of `r` only because `var` copies the type. An element can be assigned, `t.Item1 = 5;`.

**Conversion.** A tuple converts to another tuple of the same length when each element converts (the widening rules of 1.1.1: `(int, int)` to `(long, double)`). It never narrows, and a tuple of another length is a different type. How the arms of `?:` and `switch` convert when they are tuples is not settled yet; the compiler widens element by element there too, and this may change.

**Deconstruction.** `var (a, b) = e;` declares one variable per element of the tuple `e`, each with the type of its element. A `_` in place of a name discards that element. There must be one name per element, and at least two names.

```nz output="7 seven\n7 seven seven\n2 9 9\n1 2 4 10\n"
package main;

import std.io.Console;

(int, string) Pair() {
    return (7, "seven");
}

(int Lo, int Hi) Range(int a, int b) {
    return (a, b);
}

void Main(string[] args) {
    var p = Pair();
    Console.WriteLine($"{p.Item1} {p.Item2}");
    var (n, s) = Pair();
    var (_, text) = Pair();
    Console.WriteLine($"{n} {s} {text}");
    var r = Range(2, 9);
    Console.WriteLine($"{r.Lo} {r.Hi} {r.Item2}");
    (long, double) wide = (1, 2);
    (int, int) same = (3, 4);
    (int x, int y) named = same;
    named.x = 10;
    Console.WriteLine($"{wide.Item1} {wide.Item2} {named.y} {named.Item1}");
}
```

Deconstruction also works in a `for` initializer. It declares new variables only: `(a, b) = (b, a);` (assignment to existing variables), a nested `var ((a, b), c) = e;` and a typed form `(int, string) (a, b) = e;` are not supported, and neither is deconstruction in `foreach`.

```nz fragment error="cannot deconstruct a tuple of 3 elements into 2 names"
var (a, b) = (1, 2, 3);
```

```nz fragment error="only a tuple or a positional record struct can be deconstructed"
var (a, b) = 5;
```

**Rules for the names.** Two elements cannot share a name, and the name `ItemN` is allowed only on position N (`(int Item2, int y)` is an error; `(int Item1, int y)` is fine). Reading an element that does not exist is an error naming the tuple type.

```nz fragment error="has no element `Item3`"
var t = (1, 2);
int z = t.Item3;
```

**What a tuple cannot do (yet).**

- **No `==` or `!=`.** Compare the elements.
- **No formatting.** A tuple in an interpolation hole is refused; format its elements.
- **No `null` element without a type.** `var t = (1, null);` cannot infer the element type; write the type, `(int, string?) t = (1, null);`. A conditional with a tuple on one side and `null` on the other, `b ? (1, 2) : null`, is refused too.
- **No `foreach` with a tuple type.** `foreach ((int, int) p in xs)` is refused as `not supported yet: tuple types`. (The type written in a `foreach` is not checked against the elements of the collection at all yet; a wrong type there is also accepted. This is an old gap that is not settled.)
- **No default value.** A tuple type has no default, so a field of tuple type needs an initializer, or an optional type.

```nz fragment error="cannot apply `==`"
var a = (1, 2);
var b = (1, 2);
bool same = a == b;
```

```nz fragment error="cannot format a tuple in an interpolated string"
var t = (1, 2);
Console.WriteLine($"{t}");
```

**Ownership.** A tuple is copied when all its elements are, and moves as a whole when one of them is owned (a class object, a list, an array); see [Ownership](+/Ownership.md). Moving one element out of an owned tuple is refused, but `in` lends it and deconstruction takes every element at once.

```nz error="cannot move out of tuple element"
package main;

class C {
    int N;
}

void Main(string[] args) {
    var t = (1, new C());
    C c = t.Item2;
}
```

**Tuples and local functions.** A tuple can be a parameter or a result of a local function, and a local function can read a tuple it captures. `(int, int) swap((int, int) p) { ... }` written in a block is a local function, like any other type followed by a name and `(`.

**Limits of the compiler.** A tuple type nests at most 256 levels, and parentheses in an expression at most 10000.

## 2.7 Static members

A class or struct can have member constants, static fields and static methods. A static member belongs to the type, not to an object: it is reached as `Type.Name`, or by its bare name inside the type. A static method has no `this`; inside it, an instance member is an error.

```nz output="4 6\n"
package main;

import std.io.Console;

class Counter {
    static int count;
    const int Step = 2;

    public static unsafe void Hit() {
        count += Step;
    }

    public static unsafe int Count() {
        return count;
    }

    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    unsafe {
        Counter.Hit();
        Counter.Hit();
        Console.WriteLine($"{Counter.Count()} {Counter.Twice(3)}");
    }
}
```

A static member is never reached through an object:

```nz error="is static; write `Counter.Twice(...)`"
package main;

class Counter {
    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    int n = new Counter().Twice(1);
}
```

- A **static field** is a static (see [Statics](#19-statics-and-unsafe)): the same initializer rules, `unsafe` for a mutable one, and an owned one is dropped at exit. It is not part of the object.
- A **member constant** is static already: `static const` is an error.
- A **static method** may use the static members and constants of its type by bare name, and is called by bare name from the type's other methods and local functions.
- A plain `readonly` field (not `static`) is assigned only by its initial value and the constructors of its type (see [Object initializers](#object-initializers)).
- Static members of generic types, static properties, and static members of interfaces are not supported yet.

## 2.8 Type aliases

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

## 2.9 Unions

A union holds exactly one value of one of its case types. It is declared with its cases, in the shape of C# 15:

```nz output="cat Tom\nbird with 2 wings\n2\nTrue dog of 3\n"
package main;

import std.io.Console;

class Cat { public string Name = "Tom"; }
class Dog { public int Age = 3; }
struct Bird { public int Wings = 2; }

union Pet(Cat, Dog, Bird);

string Describe(in Pet p) {
    return p switch {
        Cat c => $"cat {c.Name}",
        Dog d => $"dog of {d.Age}",
        Bird b => $"bird with {b.Wings} wings",
    };
}

void Main(string[] args) {
    Pet p = new Cat();
    Console.WriteLine(Describe(p));
    p = new Bird();
    Console.WriteLine(Describe(p));
    if (p is Bird b) {
        Console.WriteLine($"{b.Wings}");
    }
    Pet q = (Pet)new Dog();
    Console.WriteLine($"{q is Dog} {Describe(q)}");
}
```

**Cases.** A case is any type that is not optional: a built-in type, a class, struct, enum, tuple, another union, or a generic instance (`union Maybe<T>(Some<T>, None);`). A union needs at least two cases, two cases may not be the same type (or become the same type for some type arguments, as `Box<T>` and `Box<int>` can), and a union is not a case of itself. A bare type parameter (`union U<T>(T, Dog)`) is not a case, because it could be the same type as another case. Arrays and interfaces as cases are not supported yet. A union has no fields of its own: the data lives in its case types.

**Conversion.** A value of a case type converts to the union implicitly, as above, and an explicit cast `(Pet)new Dog()` works as in C#. When the value's type is not itself a case, the case it converts to is chosen; when it converts to more than one case, NZ refuses where C# 15 would pick the better one, and asks for a cast to the case that is meant:

```nz error="converts to more than one case"
package main;

union Number(long, double);

void Main(string[] args) {
    Number n = 5;
}
```

`Number n = (long)5;` and `Number m = 2.5;` are accepted. The arms of `?:` and of a `switch` expression take their natural type first, when the arms have one (`int` and `long` give `long`), as C# 9 does; only when they have none is each arm converted to the target type, so `Pet p = c ? new Cat() : new Dog();` is a `Pet`.

**No null, no default.** A union always holds one of its cases. `Pet p = null;` is an error (``a union is never null; declare `Pet?` ``), and so is `default` for a union (`` `Pet` has no default value ``). Write `Pet?` where "maybe none" is meant.

**Storage.** A union is a value: a tag and the case it holds, stored in place with no allocation. It is `Copy` when every case is `Copy`; when a case holds a string (and none is owned) it is counted like a struct of strings, copied freely with one more count on the string it holds; otherwise it is owned: an owned union moves as a whole. Dropping a union drops only the case it holds (see [Ownership](+/Ownership.md)).

**Methods.** A union body may hold methods. `this` is the union, borrowed, and `this switch { ... }` is the usual way to read it:

```nz output="2 4\n"
package main;

import std.io.Console;

class Cat { }
class Dog { }
struct Bird { public int Wings = 2; }

union Pet(Cat, Dog, Bird) {
    public int Legs() {
        return this switch {
            Bird b => b.Wings,
            _ => 4,
        };
    }
}

void Main(string[] args) {
    Pet p = new Bird();
    Pet q = new Dog();
    Console.WriteLine($"{p.Legs()} {q.Legs()}");
}
```

Properties and constants in a union body are not supported yet. A union has no text form and no `==` yet: format the case it holds, and test the case with `is` or `switch` (see "Not yet" in section 7).

## 2.10 Patterns and the `switch` statement

**Type patterns.** On a union, a `switch` arm or an `is` test names a case: `Cat c` binds the case to a new name, `Cat` alone only tests it, and `_` takes anything else. A binding borrows the case: it can be read and changed, not moved out. A type that is not a case of the union is an error (`` `Fish` is not a case of `Pet` ``). The union itself is a type test of the value, as in C#: on a `Pet?`, `o is Pet p` tests for null and binds the union (`n.Left is Tree l` in a recursive union), and on a `Pet` it always matches. On any other type, a type pattern is the tested type itself (`int i` on an `int?` tests for null); a type pattern between classes waits for inheritance.

**Constant, relational and logical patterns** (C# 9). A constant (`0`, `'a'`, `"hi"`, `null`, a named constant or an enum member) matches an equal value; `< c`, `<= c`, `> c` and `>= c` compare numbers, `char`s and enum values with a constant; `not`, `and` and `or` combine patterns, in that order of precedence, and parentheses group them. `is` takes every pattern, and `x is not null` proves `x` is not null where it holds:

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

A `switch` on an optional is allowed: `null` matches the missing value, and every other pattern except `var`, `_` and `not ...` needs a value. As in C#, a constant pattern on an `int?` tests the value, and an `int` constant matches a `double`.

**Property, positional and `var` patterns** (C# 8). `{ X: 0, Y: > 1 }` matches a value whose fields match; `Point { X: 0 }` also tests the type (on a union, the case; on an optional, not null), and `{ }` matches any value that is not null. `(p, q)` matches a tuple element by element, and `Pair(p, q)` a positional `record struct` by its parameters (tuples and positional records, no user `Deconstruct`). `var x` matches every value, `null` included, and binds it; a name after a property or positional pattern binds the matched value (`{ X: > 0 } p`). A variable under `or` or `not` is an error, as in C#.

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

Property patterns read fields (computed properties come with properties); the members of built-in types (`Count`, `Length`) and the objects behind `Shared` and `Arc` handles are not supported yet. C# 10's extended form `{ A.X: 0 }` is not supported yet: write `{ A: { X: 0 } }`.

**Guards** (C# 7). `when` after a pattern adds a condition, which can read the pattern's variables. It runs only when the pattern matched, once, in arm order; when it is false, matching goes on with the next arm. It works on `case` labels too. A guard cannot move, replace or change the value its `switch` tests, nor the list or object that holds it (the later arms still test it), even when only `_` follows; lend the value with `in` instead. The parts inside the tested value and the object behind a `Shared` handle are not held: a guard may change them, and the later arms then see the change, as in C#:

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

**Exhaustiveness.** As in C#, a `switch` expression that does not cover every value is a warning (C#'s CS8509), and at run time a value that no arm matches stops the program with `nz: SwitchExpressionException: Non-exhaustive switch expression failed to match its input.`:

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

On an enum, naming every member is not enough: an enum can hold values that have no name (`(Color)2`), so the switch also needs `_`, as C#'s CS8524 says:

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

A guarded arm covers nothing. A `string` or floating-point constant takes nothing away either, so such a `switch` needs `_` even where C# could prove more. The check stops after 100,000 steps of work: a `switch` that ends with `_` or `var x` is then accepted, only the reachability of its earlier arms left unchecked; one that does not is an error that asks for `_`.

An arm that an earlier arm already covers is an error, as in C#: after `Cat => 4, Dog => 4` on a `Pet`, `_ => 0` is ``pattern `_` is already handled by an earlier arm``, and after `< 10`, the arm `< 5` is too. As in C#, a `_` after `{ }` on a class is allowed (C# counts `null` there).

**The `switch` statement** has C#'s form: sections that start with one or more `case` labels or `default`, and end with `break`, `return`, `continue` or another way out. Control cannot fall from one section into the next; several labels can share one section:

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

The labels take the patterns of the `switch` expression (`case Ok<int> ok:`), and the sections share one declaration space, as in C#. A statement that does not cover every value is no warning: nothing runs when no label matches. `goto case`, a pattern variable in a section with more than one label, and a variable declared in one section and used in another are not supported yet.

**Temporaries.** A value made on the spot that a `switch` or an `is` tests lives until the end of the statement that holds it, so a binding into it is valid for the whole statement; for an `if`, that is both branches.

## 2.11 Arrays and collection expressions

An array `T[]` has a fixed length, lives on the heap and owns its elements. It is made empty with `new T[n]` (every element is the default value of `T`, so `T` needs one: `new string?[3]`, not `new string[3]`), or from its elements, as in C#:

```nz output="3 2 4 3\nimport package\n9 10 2\n"
package main;
import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    int[] a = new int[] { 1, 2, 3 };
    long[] b = new[] { 1, 2L };          // the best common type of the elements
    int[] c = { 4, 5 };                  // only as the initializer of a declared array
    int[] d = [6, 7, 8];                 // a collection expression
    Console.WriteLine($"{a[2]} {b[1]} {c[0]} {d.Length}");

    string[] words = ["import", "package"];
    Console.WriteLine($"{words[0]} {words[1]}");

    List<int> l = [9, 10];               // a collection expression makes a list too
    Console.WriteLine($"{l[0]} {l[1]} {l.Count}");
}
```

`new T[n] { ... }` takes a constant `n` equal to the number of elements. `new[] { ... }` needs elements with a best common type (`new[] { 1, "a" }` is an error). `{ ... }` alone initializes only a declared array variable or field (`var a = { 1 };` is an error, as in C#).

A collection expression `[a, b]` takes its type from where it stands: a declared variable or field, an assignment, a `return`, an argument, a `?:` arm (from the target, or else from the other arm: `var q = c ? [2] : b;`), an element of a typed tuple (`(int[], int) t = ([1, 2], 3);`), or the value of `new Shared<int[]>(...)`. Its target is an array or a `List<T>`. As in C# 12, it has no type of its own, so `var xs = [1, 2];` is an error:

```nz fragment error="there is no target type for the collection expression"
var xs = [1, 2];
```

**Ownership.** The elements are evaluated left to right, then the array or list is made and each element is moved into it (a class object, a list, an array) or counted (a string, a `Shared<T>` handle), as when it is passed to `Add`. A variable moved into a collection cannot be used afterwards, and an element of another collection cannot be moved out to make a new one:

```nz fragment error="use of moved value `n`"
var n = new List<int>();
List<int>[] all = [n];
int k = n.Count;
```

**Spread.** As in C# 12, an element written `..e` copies every element of `e` (an array, a `List<T>` or a span) into the new collection, in order. The source is only read: it keeps its own elements.

```nz output="1 2 3 4 5\n2\n"
package main;
import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    int[] a = [1, 2];
    List<int> l = [3];
    int[] all = [..a, ..l, 4, 5];
    Console.WriteLine($"{all[0]} {all[1]} {all[2]} {all[3]} {all[4]}");
    Console.WriteLine($"{a.Length}");
}
```

Numbers, `bool`, `char` and enums are copied; a string or a `Shared<T>` handle gets a count of its own, as when it is passed by value. Elements that are owned (class objects, arrays, lists) cannot be spread, because each would get a second owner:

```nz fragment error="its elements are owned and would get a second owner"
List<int[]> rows = [[1], [2]];
List<int[]> copy = [..rows];
```

### Indexes from the end and ranges

As in C#, `^n` is an `Index` counted from the end (`^1` is the last element) and `a..b` is a `Range` from `a` up to, not including, `b`; either bound may be left out (`1..`, `..^1`, `..`). `Index` and `Range` are values like numbers: they can be stored, passed and returned, and an `int` converts to an `Index`. An array, a `List<T>` and a `string` take an `Index`; a `string` also takes a `Range` and gives a new string, as `Substring` does:

```nz output="4 2 o\nhel ell lo\n1..^1\n"
package main;
import std.io.Console;

void Main(string[] args) {
    int[] a = [1, 2, 3, 4];
    string s = "hello";
    Index last = ^1;
    Console.WriteLine($"{a[last]} {a[^3]} {s[^1]}");
    Range r = 1..^1;
    Console.WriteLine($"{s[..3]} {s[1..4]} {s[^2..]}");
    Console.WriteLine($"{r}");
}
```

`^0` is one past the end, so `a[^0]` stops the program with `index out of range`; a negative value after `^` stops with .NET's `ArgumentOutOfRangeException`, and so does a range outside the string. A range binds tighter than `+` and `-` (as in C#): `1..n-1` is `(1..n) - 1`, an error; write `1..(n - 1)`. A `List<T>` cannot be sliced (C# has no `List<T>` slice), and slicing an array is not supported yet: whether it gives a new array (C#) or a borrowed view is not decided yet.

### Spans

A `Span<T>` is a view of elements another value owns: a pointer and a length, as in C#. It is a parameter type: `Span<T>` lets the function read and write the elements, `in Span<T>` only read them. An array converts to a span of its elements, and so does a collection expression (`Sum([1, 2])`); a span parameter can be indexed (also with `^`), measured with `.Length`, iterated with `foreach`, sliced with a range (giving a span over part of the same storage) and passed on to another span parameter:

```nz output="10 5 7\n1 0 0 4\n"
package main;
import std.io.Console;

int Sum(in Span<int> s) {
    int t = 0;
    foreach (int x in s) {
        t += x;
    }
    return t;
}

int Inner(in Span<int> s) {
    return Sum(s[1..^1]);
}

void Clear(Span<int> s) {
    for (int i = 0; i < s.Length; i++) {
        s[i] = 0;
    }
}

void ClearMiddle(Span<int> s) {
    Clear(s[1..^1]);
}

void Main(string[] args) {
    int[] a = [1, 2, 3, 4];
    Console.WriteLine($"{Sum(a)} {Inner(a)} {Sum([3, 4])}");
    ClearMiddle(a);
    Console.WriteLine($"{a[0]} {a[1]} {a[2]} {a[3]}");
}
```

A span lives only as long as the call it is passed to: it cannot be stored in a local variable, a field, a static, an array or a list, returned, or used as a type argument; this is stricter than C#, which allows span locals. Real C# lets a function write the elements of an `in Span<T>`; in NZ `in` is read-only, as for every `in` parameter. `ReadOnlySpan<T>` is not supported yet. How a span borrows its source is in [Ownership](+/Ownership.md#spans-borrow-for-one-call).

### `params`

The last parameter can be `params Span<T>` or, as in C#, `params T[]`. A call then passes either one array, span or collection expression for it, or any number of values of type `T`, which are put in a new array:

```nz output="0 6 9 2\n2 0\n"
package main;
import std.io.Console;

int Sum(params Span<int> xs) {
    int t = 0;
    foreach (int x in xs) {
        t += x;
    }
    return t;
}

int Count(params string[] words) {
    return words.Length;
}

void Main(string[] args) {
    int[] a = [1, 1];
    Console.WriteLine($"{Sum()} {Sum(1, 2, 3)} {Sum([4, 5])} {Sum(a)}");
    Console.WriteLine($"{Count("x", "y")} {Count()}");
}
```

The values are moved into the new array (or counted, for strings and handles). For a `params Span<T>` the array is a temporary, freed with its elements at the end of the statement; a `params T[]` array is given to the function, as any array passed by value. `params` on any other parameter, or on a type that is not a span or an array, is an error.

# 3. Null Safety

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

Operators on an optional number, `char`, `bool` or enum are **lifted**, as in C#. An arithmetic, bitwise, shift or unary operator gives `null` when an operand is `null`, otherwise the plain operator's result as an optional (`int? + int` is an `int?`, `byte? + byte` an `int?`); the operation runs only when every operand has a value, with the plain operator's promotion and checked arithmetic. `<`, `>`, `<=` and `>=` give a `bool`: `false` when an operand is `null` (so `a <= b` is `false` when both are `null`, while `a == b` is `true`). On `bool?`, `&` and `|` follow C#'s three-valued table: `false & null` is `false`, `true | null` is `true`, and the other combinations with `null` give `null`; `&&` and `||` take only `bool`. Compound assignment and `++`/`--` lift too (`a += 1` leaves a `null` `a` as it is). A `string?` is not lifted, but `+` with a string operand is concatenation, as in C#: a null operand counts as `""`, so `s + "x"` on a null `s` is `"x"` (a `string`, never null), and `s += "x"` on a null `string?` stores `"x"`. An object is not lifted: check it first.

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

`a?[i]` reads an element of an array, a `List<T>`, a `string` or a span that may be null, as in C#: it is `null` when `a` is, and then the index is not evaluated; otherwise it is the element, as an optional (`int?`, `char?`, `string?`). An array reached through `?.` works the same way (`box?.Items?[0]`).

```nz fragment output="none\n2 3\n"
List<int>? l = null;
int? first = l?[0];
Console.WriteLine(first == null ? "none" : "some");
l = [2, 3];
Console.WriteLine($"{l?[0]} {l?[^1]}");
```

The full rules, including when a check stops being valid, are in [Null Safety](+/Null-Safety.md).

# 4. Ownership & Borrowing

The core tier of the memory model is checked by the compiler. The full description, with every rule and error, is in [Ownership](+/Ownership.md). In short:

- **One owner.** A class object, list or array has one owner. Assigning it or passing it without a modifier **moves** it, and using the old variable is an error:

```nz error="moved"
package main;

class Box {
    int N;
}

void Take(Box b) { }

void Main(string[] args) {
    var b = new Box();
    Take(b);
    Take(b);
}
```

- **Moving out of a field or a list element is an error**: the object or list keeps owning it. `var t = l[i];` of a class element is a read-only alias of the element that ends when the collection may change (see Ownership, "Reading an element through an alias").
- **Borrowing lasts one call.** `in` lends for reading, `ref` lends for writing; there are no local borrows. An `in` parameter is a read-only reference, as in C#, also for value types: a change to the caller's storage during the call (by an `unsafe` callee writing a static, or through another handle to a shared object) is seen through it, except through an `in string`, which keeps the value the call started with (see [Ownership: Strings](+/Ownership.md#strings)). One `ref` (or moved) argument cannot share storage with another argument of the same call, and a `foreach` freezes the collection it walks.
- **Local functions borrow what they read.** A local function reads the variables around it as a read-only borrow that lasts for each call; it cannot move, assign or lend them by `ref`, and each call counts as a read of every variable it reads ([Ownership](+/Ownership.md)).
- **Freed at scope exit.** Owned objects are freed when the owner leaves scope, is replaced or is moved into a consuming call; `defer`s and drops of a scope run in one reverse registration order. Inside one drop the object is freed first, then its fields in declaration order, each completely before the next (see [Ownership: The order inside one drop](+/Ownership.md#the-order-inside-one-drop)); real C# has no deterministic destructor order, so this order is NZ's own. Strings are reference-counted: every holder keeps a count and the last one frees the bytes (see [Ownership: Strings](+/Ownership.md#strings)).
- **A union moves as a whole.** A union whose cases are all `Copy` is `Copy`, one with a string case (and no owned one) is counted like a struct of strings, and any other is owned; dropping it drops the case it holds. A `switch` or `is` borrows the union it tests, and a binding (`Cat c`) borrows the case, so a case cannot be moved out of a union.
- **Generics are checked once**, with `T` as an owned type. A generic function that takes an element out of `List<T>` is refused even if only `int` uses it.
- Forms the checker cannot prove safe are refused with `not supported yet:` and a way to rewrite them.

# 5. Memory Model

NZ uses two memory tiers layered on top of one shared core, not global modes:

| Tier | Form | Used for |
|---|---|---|
| 1. No GC | ownership and borrowing | kernels, game loops, hot paths |
| 2. ARC | `Arc<T>`, `Shared<T>`, `WeakReference<T>` | shared structures, graphs |

The explanation and the design reasons are in [Memory Model](+/Memory-Model.md).

Tier 2 lives in `std.memory`:
- `Shared<T>` shares an object on one thread;
- `Arc<T>` shares a read-only object, with atomic counts;
- `WeakReference<T>` reaches a `Shared<T>` object without keeping it alive.

`new Shared<T>(v)` moves `v` in, and copying a handle shares the object and counts it. The object changes through any handle, whose members, indexer and `foreach` reach the payload. Borrows of its owned parts are checked when the program runs, and a change while one is lent stops the program. `w.TryGetTarget()` gives a `Shared<T>?`, and `Shared.StrongCount(s)` reads the count. Cycles of strong handles leak visibly (live count, `nzc test`); a `WeakReference<T>` back edge breaks them. The rules, with examples, are in [Ownership: Shared objects](+/Ownership.md#shared-objects), and `examples/graph.nz` is a whole program.

# 6. Standard Library

This section is written automatically from the compiler's table of builtin types. A builtin type is only visible after it is imported. The rules every member follows (what happens on failure, who owns a result, string positions, ordering, differences from .NET) are in [Standard Library](%2B/Standard-Library.md).

<!-- GEN:std -->
### Package `std.io`

#### `Console`

Standard input, output and error.

```nz-api
import std.io.Console;
```

| Member | Signature | Description |
|---|---|---|
| `WriteLine` (static) | `void WriteLine(string)`<br>`void WriteLine(bool)`<br>`void WriteLine(char)`<br>`void WriteLine(sbyte)`<br>`void WriteLine(byte)`<br>`void WriteLine(short)`<br>`void WriteLine(ushort)`<br>`void WriteLine(int)`<br>`void WriteLine(uint)`<br>`void WriteLine(long)`<br>`void WriteLine(ulong)`<br>`void WriteLine(nint)`<br>`void WriteLine(nuint)`<br>`void WriteLine(float)`<br>`void WriteLine(double)`<br>`void WriteLine(string?)`<br>`void WriteLine()` | Writes the value as text and a line break to standard output. |
| `Write` (static) | `void Write(string)`<br>`void Write(bool)`<br>`void Write(char)`<br>`void Write(sbyte)`<br>`void Write(byte)`<br>`void Write(short)`<br>`void Write(ushort)`<br>`void Write(int)`<br>`void Write(uint)`<br>`void Write(long)`<br>`void Write(ulong)`<br>`void Write(nint)`<br>`void Write(nuint)`<br>`void Write(float)`<br>`void Write(double)`<br>`void Write(string?)` | Writes the value as text to standard output. |
| `ReadLine` (static) | `string? ReadLine()` | The next line of standard input without its line break, or `null` at the end of input. |
| `Error.WriteLine` (static) | `void WriteLine(string)`<br>`void WriteLine(bool)`<br>`void WriteLine(char)`<br>`void WriteLine(sbyte)`<br>`void WriteLine(byte)`<br>`void WriteLine(short)`<br>`void WriteLine(ushort)`<br>`void WriteLine(int)`<br>`void WriteLine(uint)`<br>`void WriteLine(long)`<br>`void WriteLine(ulong)`<br>`void WriteLine(nint)`<br>`void WriteLine(nuint)`<br>`void WriteLine(float)`<br>`void WriteLine(double)`<br>`void WriteLine(string?)`<br>`void WriteLine()` | Writes the value as text and a line break to standard error. |
| `Error.Write` (static) | `void Write(string)`<br>`void Write(bool)`<br>`void Write(char)`<br>`void Write(sbyte)`<br>`void Write(byte)`<br>`void Write(short)`<br>`void Write(ushort)`<br>`void Write(int)`<br>`void Write(uint)`<br>`void Write(long)`<br>`void Write(ulong)`<br>`void Write(nint)`<br>`void Write(nuint)`<br>`void Write(float)`<br>`void Write(double)`<br>`void Write(string?)` | Writes the value as text to standard error. |

#### `File`

Whole files: read and write bytes, UTF-8 text or lines; test, delete and copy.

```nz-api
import std.io.File;
```

| Member | Signature | Description |
|---|---|---|
| `ReadAllBytes` (static) | `byte[] ReadAllBytes(string)` | The file's bytes, as a new array. |
| `ReadAllText` (static) | `string ReadAllText(string)` | The file's text, as UTF-8 (a byte order mark is skipped; invalid sequences become U+FFFD). |
| `ReadAllLines` (static) | `string[] ReadAllLines(string)` | The file's lines without their line breaks (`\n`, `\r\n` or `\r`). |
| `WriteAllBytes` (static) | `void WriteAllBytes(string, byte[])` | Makes or replaces the file with the bytes. |
| `WriteAllText` (static) | `void WriteAllText(string, string?)` | Makes or replaces the file with the text as UTF-8, without a byte order mark. |
| `AppendAllText` (static) | `void AppendAllText(string, string?)` | Adds the text as UTF-8 to the end of the file, making it if it is missing. |
| `Exists` (static) | `bool Exists(string?)` | Whether the path names an existing file (not a folder); `false` for null or an empty path. |
| `Delete` (static) | `void Delete(string)` | Deletes the file; a missing file is no error, a missing folder is. |
| `Copy` (static) | `void Copy(string, string)`<br>`void Copy(string, string, bool)` | Copies the file; an existing destination is an error unless `overwrite` is `true`. |
| `TryReadAllText` (static) | `Result<string, IOError> TryReadAllText(string)` | `File.ReadAllText`, returning `Err` instead of stopping. |
| `TryReadAllLines` (static) | `Result<string[], IOError> TryReadAllLines(string)` | `File.ReadAllLines`, returning `Err` instead of stopping. |
| `TryReadAllBytes` (static) | `Result<byte[], IOError> TryReadAllBytes(string)` | `File.ReadAllBytes`, returning `Err` instead of stopping. |
| `TryWriteAllText` (static) | `Result<bool, IOError> TryWriteAllText(string, string?)` | `File.WriteAllText`; `Ok(true)` when it wrote. |
| `TryWriteAllBytes` (static) | `Result<bool, IOError> TryWriteAllBytes(string, byte[])` | `File.WriteAllBytes`; `Ok(true)` when it wrote. |
| `TryAppendAllText` (static) | `Result<bool, IOError> TryAppendAllText(string, string?)` | `File.AppendAllText`; `Ok(true)` when it wrote. |
| `TryDelete` (static) | `Result<bool, IOError> TryDelete(string)` | `File.Delete`; `Ok(true)` when it returned (a missing file is no error). |

#### `Path`

Paths as strings: join, split into folder, name and extension, make absolute.

```nz-api
import std.io.Path;
```

| Member | Signature | Description |
|---|---|---|
| `Combine` (static) | `string Combine(string, string)`<br>`string Combine(string, string, string)`<br>`string Combine(string, string, string, string)` | The parts joined with the separator; a rooted part replaces what comes before it, an empty one is left out. |
| `GetFileName` (static) | `string? GetFileName(string?)` | The part after the last separator. |
| `GetFileNameWithoutExtension` (static) | `string? GetFileNameWithoutExtension(string?)` | The file name without its last `.` and what follows. |
| `GetExtension` (static) | `string? GetExtension(string?)` | The last `.` of the file name and what follows (empty when there is none or nothing follows). |
| `GetDirectoryName` (static) | `string? GetDirectoryName(string?)` | The part before the last separator; null for null, an empty path or a root. |
| `ChangeExtension` (static) | `string? ChangeExtension(string?, string?)` | The path with its extension replaced (removed for a null `extension`). |
| `IsPathRooted` (static) | `bool IsPathRooted(string?)` | Whether the path starts at a root (a separator, or a drive on Windows). |
| `GetFullPath` (static) | `string GetFullPath(string)` | The path made absolute against the current directory, `.` and `..` removed. |
| `DirectorySeparatorChar` (static) | `char DirectorySeparatorChar { get; }` | `/`, or `\` on Windows. |
| `AltDirectorySeparatorChar` (static) | `char AltDirectorySeparatorChar { get; }` | `/` on every OS. |
| `PathSeparator` (static) | `char PathSeparator { get; }` | The separator in path lists such as `PATH`: `:`, or `;` on Windows. |
| `VolumeSeparatorChar` (static) | `char VolumeSeparatorChar { get; }` | `/`, or `:` on Windows. |

#### `Directory`

Folders: make, test, list (sorted) and delete.

```nz-api
import std.io.Directory;
```

| Member | Signature | Description |
|---|---|---|
| `CreateDirectory` (static) | `void CreateDirectory(string)` | Makes the folder and every missing folder above it; an existing folder is no error. |
| `Exists` (static) | `bool Exists(string?)` | Whether the path names an existing folder; `false` for null or an empty path. |
| `GetFiles` (static) | `string[] GetFiles(string)`<br>`string[] GetFiles(string, string)` | The files in the folder matching `searchPattern` as .NET matches it (`*`, `?`, `*.*` for all; it may start with a sub folder), sorted ordinally. |
| `GetDirectories` (static) | `string[] GetDirectories(string)`<br>`string[] GetDirectories(string, string)` | The folders in the folder (matching `searchPattern`), sorted ordinally. |
| `Delete` (static) | `void Delete(string)`<br>`void Delete(string, bool)` | Deletes the folder: it must be empty unless `recursive` is `true`. |
| `GetCurrentDirectory` (static) | `string GetCurrentDirectory()` | The current directory, as a full path. |
| `TryCreateDirectory` (static) | `Result<bool, IOError> TryCreateDirectory(string)` | `Directory.CreateDirectory`; `Ok(true)` when the folder exists afterwards. |
| `TryGetFiles` (static) | `Result<string[], IOError> TryGetFiles(string)`<br>`Result<string[], IOError> TryGetFiles(string, string)` | `Directory.GetFiles`, returning `Err` instead of stopping. |
| `TryDelete` (static) | `Result<bool, IOError> TryDelete(string)`<br>`Result<bool, IOError> TryDelete(string, bool)` | `Directory.Delete`; `Ok(true)` when the folder is gone. |

### Package `std.collections`

#### `List<T>`

A growable list that owns its elements.

```nz-api
import std.collections.List;
```

| Member | Signature | Description |
|---|---|---|
| constructor | `new List<T>()`<br>`new List<T>(int capacity)` | An empty list, or an empty one with room for `capacity` elements (a negative capacity stops). |
| `Add` | `void Add(T)` | Adds the element at the end; the list owns it from then on. |
| `Count` | `int Count { get; }` | The number of elements. |
| `Clear` | `void Clear()` | Removes and drops every element. |
| `RemoveAt` | `void RemoveAt(int)` | Removes and drops the element at `index`; the ones after it move down. |
| `Insert` | `void Insert(int, T)` | Inserts the element at `index` (`0` to `Count`); the list owns it from then on. |
| `Contains` | `bool Contains(T)` | Whether an element equals `item` (elements of a hash key type only). |
| `Remove` | `bool Remove(T)` | Removes and drops the first element equal to `item`; `false` when there is none (elements of a hash key type only). |
| `Sort` | `void Sort()` | Orders the elements ascending (an integer type, `char`, `float`, `double` or `string`; strings ordinal, `NaN` first). |
| `IndexOf` | `int IndexOf(T)` | The index of the first element equal to `item`, or -1 (elements of a hash key type only). |
| `ToArray` | `T[] ToArray()` | A new array with copies of the elements (elements that are copied only). |
| indexer | `this[int] -> T` | Reads or writes the element (of a dictionary: the value of the key). |

#### `Dictionary<TKey, TValue>`

A hash table that owns its keys and values; `foreach` visits them in insertion order.

```nz-api
import std.collections.Dictionary;
```

| Member | Signature | Description |
|---|---|---|
| `Add` | `void Add(TKey, TValue)` | Adds the pair; a present key stops (`ArgumentException`). The dictionary owns both from then on. |
| `Remove` | `bool Remove(TKey)` | Removes the key and drops its value; `false` when the key is not present. |
| `Clear` | `void Clear()` | Removes and drops every pair. |
| `ContainsKey` | `bool ContainsKey(TKey)` | Whether the key is present. |
| `TryGetValue` | `TValue? TryGetValue(TKey)` | A copy of the key's value, or null when the key is missing (C#'s `out` value is the result); only for values that are copied. |
| `GetValueOrDefault` | `TValue GetValueOrDefault(TKey)`<br>`TValue GetValueOrDefault(TKey, TValue)` | A copy of the key's value, or `defaultValue`, or the zero value (`0`, `false`, null); only for values that are copied, and the one-argument form not for `string` values (.NET gives null). |
| `Count` | `int Count { get; }` | The number of pairs. |
| indexer | `this[TKey] -> TValue` | Reads or writes the element (of a dictionary: the value of the key). |

#### `KeyValuePair<TKey, TValue>`

A key and its value, borrowed from a `Dictionary` by a `foreach` variable.

```nz-api
import std.collections.KeyValuePair;
```

| Member | Signature | Description |
|---|---|---|
| `Key` | `TKey Key { get; }` | The key. |
| `Value` | `TValue Value { get; }` | The value, borrowed: it stays the dictionary's. |

#### `HashSet<T>`

A set of distinct elements; `foreach` visits them in insertion order.

```nz-api
import std.collections.HashSet;
```

| Member | Signature | Description |
|---|---|---|
| `Add` | `bool Add(T)` | Adds the element; `false` when it is already present. |
| `Remove` | `bool Remove(T)` | Removes the element; `false` when it is not present. |
| `Contains` | `bool Contains(T)` | Whether the element is present. |
| `Clear` | `void Clear()` | Removes every element. |
| `Count` | `int Count { get; }` | The number of elements. |

### Package `std.diag`

#### `Diag`

Allocation diagnostics.

```nz-api
import std.diag.Diag;
```

| Member | Signature | Description |
|---|---|---|
| `LiveObjects` (static) | `long LiveObjects()` | The number of owned objects allocated and not yet dropped. |
| `LiveStrings` (static) | `long LiveStrings()` | The number of computed strings alive (string literals never count). |

### Package `std.memory`

#### `Shared<T>`

A shared object on one thread, reference counted.

```nz-api
import std.memory.Shared;
```

| Member | Signature | Description |
|---|---|---|
| `StrongCount` (static) | `long StrongCount(Shared<T>)` | The number of strong handles to the object. |
| `WeakCount` (static) | `long WeakCount(Shared<T>)` | The number of weak handles to the object. |

#### `Arc<T>`

A shared read-only object, atomically counted.

```nz-api
import std.memory.Arc;
```

| Member | Signature | Description |
|---|---|---|
| `StrongCount` (static) | `long StrongCount(Arc<T>)` | The number of handles to the object. |

#### `WeakReference<T>`

A handle that does not keep the object alive.

```nz-api
import std.memory.WeakReference;
```

| Member | Signature | Description |
|---|---|---|
| `TryGetTarget` | `Shared<T>? TryGetTarget()` | A strong handle to the object while it is alive, otherwise `null`. |

### Package `std.interop`

#### `NativeMemory`

```nz-api
import std.interop.NativeMemory;
```

| Member | Signature | Description |
|---|---|---|
| `Alloc` (static) | `void* Alloc(nuint)` |  |
| `AllocZeroed` (static) | `void* AllocZeroed(nuint, nuint)` |  |
| `Realloc` (static) | `void* Realloc(void*, nuint)` |  |
| `Free` (static) | `void Free(void*)` |  |
| `Copy` (static) | `void Copy(void*, void*, nuint)` |  |
| `Clear` (static) | `void Clear(void*, nuint)` |  |

#### `StructLayout`

```nz-api
import std.interop.StructLayout;
```

| Member | Signature | Description |
|---|---|---|

#### `LayoutKind`

```nz-api
import std.interop.LayoutKind;
```

| Member | Signature | Description |
|---|---|---|

#### `FieldOffset`

```nz-api
import std.interop.FieldOffset;
```

| Member | Signature | Description |
|---|---|---|

#### `DllImport`

```nz-api
import std.interop.DllImport;
```

| Member | Signature | Description |
|---|---|---|

#### `UnmanagedCallersOnly`

```nz-api
import std.interop.UnmanagedCallersOnly;
```

| Member | Signature | Description |
|---|---|---|

### Package `std.text`

#### `StringBuilder`

A string built piece by piece.

```nz-api
import std.text.StringBuilder;
```

| Member | Signature | Description |
|---|---|---|
| constructor | `new StringBuilder()`<br>`new StringBuilder(string value)`<br>`new StringBuilder(int capacity)` | An empty builder, one holding the string, or an empty one with room for `capacity` bytes. |
| `Append` | `StringBuilder Append(string)`<br>`StringBuilder Append(bool)`<br>`StringBuilder Append(char)`<br>`StringBuilder Append(sbyte)`<br>`StringBuilder Append(byte)`<br>`StringBuilder Append(short)`<br>`StringBuilder Append(ushort)`<br>`StringBuilder Append(int)`<br>`StringBuilder Append(uint)`<br>`StringBuilder Append(long)`<br>`StringBuilder Append(ulong)`<br>`StringBuilder Append(nint)`<br>`StringBuilder Append(nuint)`<br>`StringBuilder Append(float)`<br>`StringBuilder Append(double)`<br>`StringBuilder Append(string?)` | Appends the value as an interpolation hole prints it; null appends nothing. |
| `AppendLine` | `StringBuilder AppendLine()`<br>`StringBuilder AppendLine(string)`<br>`StringBuilder AppendLine(string?)` | Appends the string (if any) and `Environment.NewLine`. |
| `Clear` | `StringBuilder Clear()` | Removes everything; the room is kept. |
| `Length` | `int Length { get; }` | The length in UTF-8 bytes. |
| `ToString` | `string ToString()` | The text, as a new string. |

#### `Encoding`

Text encodings: `Encoding.UTF8`.

```nz-api
import std.text.Encoding;
```

| Member | Signature | Description |
|---|---|---|
| `UTF8.GetBytes` (static) | `byte[] GetBytes(string)` | The string's UTF-8 bytes, as a new array. |
| `UTF8.GetString` (static) | `string GetString(byte[])`<br>`string GetString(byte[], int, int)` | The bytes decoded as UTF-8 (invalid sequences become U+FFFD), as a new string. |
| `UTF8.GetByteCount` (static) | `int GetByteCount(string)` | The number of UTF-8 bytes of the string. |

### Package `std.buffers.binary`

#### `BinaryPrimitives`

Byte order: `ReverseEndianness`.

```nz-api
import std.buffers.binary.BinaryPrimitives;
```

| Member | Signature | Description |
|---|---|---|
| `ReverseEndianness` (static) | `int ReverseEndianness(int)`<br>`sbyte ReverseEndianness(sbyte)`<br>`byte ReverseEndianness(byte)`<br>`short ReverseEndianness(short)`<br>`ushort ReverseEndianness(ushort)`<br>`uint ReverseEndianness(uint)`<br>`long ReverseEndianness(long)`<br>`ulong ReverseEndianness(ulong)` | The value with its bytes in the other order. |

### Package `std`

#### `Environment`

The process: arguments, exit status, environment variables.

```nz-api
import std.Environment;
```

| Member | Signature | Description |
|---|---|---|
| `GetCommandLineArgs` (static) | `string[] GetCommandLineArgs()` | The program's path, then its arguments, as a new array. |
| `Exit` (static) | `void Exit(int)` | Ends the process with the status at once, as `Abort` does: no drops, no `defer`. |
| `ExitCode` (static) | `int ExitCode { get; set; }` | The status a normal return from `void Main` exits with; `int Main`'s result wins. |
| `NewLine` (static) | `string NewLine { get; }` | `\r\n` on Windows, `\n` elsewhere. |
| `GetEnvironmentVariable` (static) | `string? GetEnvironmentVariable(string)` | The variable's value, or `null` when it is not set. |

#### `Math`

Numeric functions and constants.

```nz-api
import std.Math;
```

| Member | Signature | Description |
|---|---|---|
| `Abs` (static) | `int Abs(int)`<br>`sbyte Abs(sbyte)`<br>`short Abs(short)`<br>`long Abs(long)`<br>`nint Abs(nint)`<br>`float Abs(float)`<br>`double Abs(double)` | The absolute value; the minimum value of a signed integer type stops (OverflowException). |
| `Max` (static) | `int Max(int, int)`<br>`sbyte Max(sbyte, sbyte)`<br>`byte Max(byte, byte)`<br>`short Max(short, short)`<br>`ushort Max(ushort, ushort)`<br>`uint Max(uint, uint)`<br>`long Max(long, long)`<br>`ulong Max(ulong, ulong)`<br>`nint Max(nint, nint)`<br>`nuint Max(nuint, nuint)`<br>`float Max(float, float)`<br>`double Max(double, double)` | The larger of two numbers; NaN if either is NaN. |
| `Min` (static) | `int Min(int, int)`<br>`sbyte Min(sbyte, sbyte)`<br>`byte Min(byte, byte)`<br>`short Min(short, short)`<br>`ushort Min(ushort, ushort)`<br>`uint Min(uint, uint)`<br>`long Min(long, long)`<br>`ulong Min(ulong, ulong)`<br>`nint Min(nint, nint)`<br>`nuint Min(nuint, nuint)`<br>`float Min(float, float)`<br>`double Min(double, double)` | The smaller of two numbers; NaN if either is NaN. |
| `Clamp` (static) | `int Clamp(int, int, int)`<br>`sbyte Clamp(sbyte, sbyte, sbyte)`<br>`byte Clamp(byte, byte, byte)`<br>`short Clamp(short, short, short)`<br>`ushort Clamp(ushort, ushort, ushort)`<br>`uint Clamp(uint, uint, uint)`<br>`long Clamp(long, long, long)`<br>`ulong Clamp(ulong, ulong, ulong)`<br>`nint Clamp(nint, nint, nint)`<br>`nuint Clamp(nuint, nuint, nuint)`<br>`float Clamp(float, float, float)`<br>`double Clamp(double, double, double)` | The value limited to `min`..`max`; `min > max` stops (ArgumentException). |
| `Sign` (static) | `int Sign(int)`<br>`int Sign(sbyte)`<br>`int Sign(short)`<br>`int Sign(long)`<br>`int Sign(nint)`<br>`int Sign(float)`<br>`int Sign(double)` | -1, 0 or 1; NaN stops (ArithmeticException). |
| `Sqrt` (static) | `double Sqrt(double)` | The square root. |
| `Pow` (static) | `double Pow(double, double)` | `x` raised to the power `y`. |
| `Floor` (static) | `double Floor(double)` | The largest integral value not above. |
| `Ceiling` (static) | `double Ceiling(double)` | The smallest integral value not below. |
| `Truncate` (static) | `double Truncate(double)` | The integral part. |
| `Round` (static) | `double Round(double)`<br>`double Round(double, int)` | Rounded to the nearest (with `digits` decimals); a half rounds to even. |
| `Log` (static) | `double Log(double)` | The natural logarithm. |
| `Log2` (static) | `double Log2(double)` | The base 2 logarithm. |
| `Log10` (static) | `double Log10(double)` | The base 10 logarithm. |
| `Exp` (static) | `double Exp(double)` | `e` raised to the power. |
| `Sin` (static) | `double Sin(double)` | The sine of an angle in radians. |
| `Cos` (static) | `double Cos(double)` | The cosine of an angle in radians. |
| `Tan` (static) | `double Tan(double)` | The tangent of an angle in radians. |
| `Atan2` (static) | `double Atan2(double, double)` | The angle whose tangent is `y / x`. |
| `PI` (static) | `double PI { get; }` | π (a constant in C#; a read-only value here). |
| `E` (static) | `double E { get; }` | e (a constant in C#; a read-only value here). |
| `Tau` (static) | `double Tau { get; }` | 2π (a constant in C#; a read-only value here). |

#### `StringComparison`

```nz-api
import std.StringComparison;
```

| Member | Signature | Description |
|---|---|---|
| `CurrentCulture` | `= 0` | |
| `CurrentCultureIgnoreCase` | `= 1` | |
| `InvariantCulture` | `= 2` | |
| `InvariantCultureIgnoreCase` | `= 3` | |
| `Ordinal` | `= 4` | |
| `OrdinalIgnoreCase` | `= 5` | |

#### `StringSplitOptions`

```nz-api
import std.StringSplitOptions;
```

| Member | Signature | Description |
|---|---|---|
| `None` | `= 0` | |
| `RemoveEmptyEntries` | `= 1` | |
| `TrimEntries` | `= 2` | |

#### `BitConverter`

Values to and from bytes, in the host byte order.

```nz-api
import std.BitConverter;
```

| Member | Signature | Description |
|---|---|---|
| `ToBoolean` (static) | `bool ToBoolean(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToInt16` (static) | `short ToInt16(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToUInt16` (static) | `ushort ToUInt16(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToInt32` (static) | `int ToInt32(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToUInt32` (static) | `uint ToUInt32(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToInt64` (static) | `long ToInt64(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToUInt64` (static) | `ulong ToUInt64(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToSingle` (static) | `float ToSingle(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `ToDouble` (static) | `double ToDouble(byte[], int)` | The value stored at `startIndex`, in the host byte order (little-endian). |
| `GetBytes` (static) | `byte[] GetBytes(int)`<br>`byte[] GetBytes(bool)`<br>`byte[] GetBytes(short)`<br>`byte[] GetBytes(ushort)`<br>`byte[] GetBytes(uint)`<br>`byte[] GetBytes(long)`<br>`byte[] GetBytes(ulong)`<br>`byte[] GetBytes(float)`<br>`byte[] GetBytes(double)` | The value's bytes in the host byte order, as a new array; a `char` is refused (an NZ `char` is no UTF-16 unit: write `(ushort)c`). |
| `IsLittleEndian` (static) | `bool IsLittleEndian { get; }` | `true` on every NZ target (a big-endian build stops at compile time). |
<!-- /GEN -->

### Package `std.result`

`Result<T, E>`, `Ok<T>` and `Err<E>` are written in NZ code (C# style) and are visible in every program without an `import`; a program's own type of the same name hides them. They are described with the errors chapter, section 1.7.

# 7. Current Behavior Notes

The C backend is young. These are facts about how the compiler behaves today, not settled design; any of them may change.

**`string.Length` counts UTF-8 bytes**, not characters:

```nz output="6\n"
package main;

import std.io.Console;

void Main(string[] args) {
    string s = "héllo";
    Console.WriteLine($"{s.Length}");
}
```

**`foreach` keeps its collection borrowed while the body runs**: the body can read the list it walks, but adding to it, assigning an element or the list, or moving it is an error. Collect the changes and apply them after the loop:

```nz error="collection is borrowed by the loop"
package main;

import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(1);
    foreach (int v in l) {
        if (l.Count < 4) {
            l.Add(v + 1);
        }
        Console.Write($"{v} ");
    }
    Console.WriteLine("");
}
```

**A list element cannot be passed by `ref`**, as in C#: a `List<T>` element is read and written through the list's indexer, so it is not a variable. An array element is a variable and can be passed by `ref`. To let a function change a list element, copy it into a variable, pass the variable and store it back. The NZ code (C# style) below swaps an array element with a list element that way:

```nz output="3 2 1\n"
package main;

import std.io.Console;
import std.collections.List;

void Swap(ref int a, ref int b) {
    int t = a;
    a = b;
    b = t;
}

void Main(string[] args) {
    int[] xs = new int[2];
    xs[0] = 1;
    xs[1] = 2;
    List<int> ys = new List<int>();
    ys.Add(3);
    int y = ys[0];
    Swap(xs[0], y);
    ys[0] = y;
    Console.WriteLine($"{xs[0]} {xs[1]} {ys[0]}");
}
```

```nz error="is an element of a list, read through its indexer, not a variable"
package main;

import std.collections.List;

void Bump(ref int x) {
    x++;
}

void Main(string[] args) {
    List<int> ys = new List<int>();
    ys.Add(3);
    Bump(ys[0]);
}
```

**A struct in a `List<T>`**: assigning a field, `l[0].X = v`, writes into the list. Calling a method, `l[0].M()`, depends on the struct. A struct whose fields are all copied freely (numbers, `bool`, `char`, `string`, enums, such structs) is copied, and the method runs on the copy, so what it changes is lost:

```nz output="5\n"
package main;

import std.io.Console;
import std.collections.List;

struct P {
    int X;
    void Bump() { X = X + 1; }
}

void Main(string[] args) {
    List<P> ps = new List<P>();
    ps.Add(new P());
    ps[0].X = 5;
    ps[0].Bump();
    Console.WriteLine($"{ps[0].X}");
}
```

A struct that owns something (a class object, a list, an array, or a struct that does) is move-only: it cannot be copied, so the method runs on the value stored in the list and its changes stay. The same holds for such a struct reached through `x?.M()`, a `foreach` variable or a pattern variable:

```nz output="6\n"
package main;

import std.io.Console;
import std.collections.List;

struct Q {
    int X;
    List<int> Log = new List<int>();
    void Bump() { X = X + 1; }
}

void Main(string[] args) {
    List<Q> qs = new List<Q>();
    qs.Add(new Q());
    qs[0].X = 5;
    qs[0].Bump();
    Console.WriteLine($"{qs[0].X}");
}
```

**`using` has no cleanup method yet.** `using (var r = ...) { ... }` evaluates its value once, makes it visible in the block and frees it, as an owned local, when the block ends; there is no interface for a user-defined cleanup method. Use `defer` inside the block for other cleanup.

**Deep recursion in a compiled program crashes** with a stack overflow and no NZ message. When standard output is piped it is fully buffered, so output printed before the crash can be lost too.

**Nesting limits.** Input nested very deeply is refused with a message, never a crash. The parser stops at 1000 levels of parentheses, of prefix operators and casts, or of types (`expression nested too deeply`); the checker stops at 1200 levels of expression (`code is nested too deeply to check`). A third limit is on the C that the compiler writes: every call, parenthesis, cast and block adds to a budget of 2500 units, because clang itself crashes on very deep C. An expression can pass `nzc check` and still be refused by `nzc build` and `nzc run` with `expression nested too deeply to compile`; a cast of a parenthesised value adds the most, so a chain of a few hundred nested casts is over the budget. Split such an expression with local variables.

**A temporary handle lives until the end of its statement.** A `Shared<T>` made inside a statement and not kept is released when the statement ends, not earlier, so a `WeakReference<T>` made from it can still reach the object within the same statement. Keep the result of `TryGetTarget()` to keep the object alive; after the statement, a weak handle to an object nobody holds gives `null`:

```nz output="True\nTrue\n"
package main;

import std.io.Console;
import std.memory.Shared;
import std.memory.WeakReference;

class Node { int N = 7; }

Shared<Node> Make() { return new Shared<Node>(new Node()); }

void Main(string[] args) {
    var t = new WeakReference<Node>(Make()).TryGetTarget();
    Console.WriteLine($"{t != null}");
    var w = new WeakReference<Node>(Make());
    Console.WriteLine($"{w.TryGetTarget() == null}");
}
```

**Dropping a deep structure needs only a bounded stack.** Owned and shared structures of any shape (a chain, a list of children, a field of a struct, objects of alternating types, a tree deep on either side) are dropped through a worklist in the runtime, not by recursion, so their depth is limited by memory. Fields drop in declaration order, so the owning fields declared after a chain's link (a field of the object's own type, the next handle of a `Shared<T>` or `Arc<T>` chain, or any other field that leads on) wait until the rest of the chain has dropped. A chain whose link is the last field that owns something leaves nothing waiting. Otherwise, while the worklist is short (a tree, a short chain), each waiting field (an object, collection, handle, string with heap text, or owning part of a struct, tuple or union) takes one worklist entry of 32 bytes. Once the worklist holds 1024 entries, an object's waiting fields are copied instead into one record, and the records of a chain of one type are linked together under a single entry. For a class object the record is a block as large as the object, allocated right after the object is freed, so the allocator can give back that same block and a long chain takes about the memory it held while it drops. For the struct payload of a `Shared<T>` or `Arc<T>` chain the record holds only the waiting fields, since the handle's block stays while a `WeakReference<T>` to it lives. A chain through objects of alternating types keeps one entry and one record per object instead of one entry per field. Records cost one allocation per object: in our measurements dropping such a chain was 13 to 17% slower than pushing entries, and a tree dropped while the worklist was already that long about 20% slower (each node of its left spines makes a record). Deep recursion in your own code still can overflow the stack (see above).

**Not yet** (each of these is refused or missing today, and none is a crash):

- `[Flags]`: an enum value that combines members prints as its number (`Read | Write` is `3`), not as `Read, Write`, and there is no `HasFlag`.
- The constant graph does not read enum members: `const int K = (int)E.A;` is refused with ``E.A is not a constant``. Enum values themselves may use constants.
- Members on enums: `E.A.ToString()` is an error. Print an enum with `{e}` in an interpolation.
- `sizeof(nint)` and `sizeof(nuint)`: the size depends on the target, so a constant cannot hold it. `sizeof` of the other eight integer types and of an enum works.
- Unions as text and with `==`: `$"{p}"` on a union is refused (`not supported yet: a union as text`), and so is `p == q`; format or compare the case it holds after `is` or `switch`. Properties and constants in a union body, arrays and interfaces as cases, and type patterns between classes are not supported yet. Patterns: list patterns (`[a, .., z]`), C# 10's extended property patterns (`{ A.X: 0 }`), property patterns on built-in types and handles, a user `Deconstruct`, and a pattern variable used after the `if` that tests `is not` (C#'s wider scope) are not supported yet. A union is not the payload of a handle: `Shared<Pet>` and `Arc<Pet>` are refused (`` `Shared<Pet>` is not supported; wrap the value in a class ``); put the union in a field of a class and share that class.
- `switch` patterns above the `long` range: on a `ulong` value, a literal pattern above `long.MaxValue` (`9223372036854775808`) is refused, and so is a suffixed literal pattern (`18446744073709551615UL`). A named constant works: `const ulong Big = 18446744073709551615UL;` and then `u switch { Big => ..., _ => ... }`.
