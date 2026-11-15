# 0. Introduction

> NZ is still in development and is not stable. Anything in this documentation can change.

**NZ** is a *systems* programming language that is memory safe and has **no garbage collector by default**. Its syntax deliberately follows C#, so it reads naturally to anyone who knows C#, while its memory model is built on **ownership** and **borrowing**: a strong architecture with a syntax that is easy to read.

NZ is designed for many kinds of programs: kernels (`no_runtime`), web/app servers, and games.

**Syntax reference.** This guide is a tour. For a construct-by-construct reference (its syntax and a short example), see [Syntax Reference](+/Syntax-Reference.md).

NZ programs **compile to C and run** through `clang` (`nzc run`), after passing the lexer, parser and semantic analysis. Before Milestone 0.1 the compiler is written in Rust and translates NZ to C; this describes the current implementation. **Ownership and borrow checking are active**: owned objects are freed deterministically, and strings are freed with their last holder (see [Ownership](+/Ownership.md) and [Compiler](+/Compiler.md)).

## 0.1 Installation and Hello World

**Requirements:**

- **The compiler**: `nzc` is built from the project's repository (not public yet).
- **Git**
- **clang**: the C compiler that `nzc` uses to build programs (set `NZ_CC` to use another one).
- **Code editor**: any; the `.nz` extension has no dedicated editor support yet.

**Running NZ code with `nzc run`:**

1. Get the code and enter its folder.
2. Run `nzc run <file.nz|folder>`. The compiler checks the program (every error is printed at once with its line and column), translates it to C, builds it with `clang`, and runs it.
3. `nzc check <file.nz|folder>` only checks and prints `ok`; `nzc build <file.nz|folder>` leaves an executable (for a folder, in its `bin` folder). A folder is a package (see [20.1](20-module-system-and-visibility.md#201-packages)).

```bash
# in the compiler repository (its source is not public yet)
nzc run examples/hello.nz
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

Every file starts with a `package` line. In a program made of a folder, the package is named after the folder that holds the file (`package main;` for a folder called `main`); see [20.1](20-module-system-and-visibility.md#201-packages). `import` lines come next, then the declarations.

The `examples/` folder holds more programs that run the same way, for example `../examples/fibonacci.nz` and `../examples/shapes.nz`. `../examples/stack.nz` (a stack and a queue) and `../examples/bst.nz` (a binary search tree) build linked structures and free every node they own.

## 0.2 Toolchain

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

`nzc test` runs the functions marked `[Test]`; see [11.1](11-testing-and-debugging.md#111-unit-testing).

**Debug and release builds.** As with `dotnet build` and `dotnet build -c Release`, `build`, `run` and `test` make a debug build unless `--release` is given. A debug build compiles the C without optimisation and with debug info (`-g`), and the C carries `#line` directives that name the `.nz` file and line of every statement, so gdb, lldb, the Visual Studio debugger and sanitizer reports show NZ source: `break main.nz:12` stops on line 12 of `main.nz`. On Windows the debug info is a `.pdb` file written beside the executable. A release build compiles the C with `-O2` (and `-fwrapv -fno-strict-aliasing`, so the optimiser keeps every behaviour of the debug build) and without debug info: it runs faster and prints exactly what the debug build prints. `--release` is accepted by `build`, `run` and `test`.

**Warnings.** A warning is a problem that does not stop the build. It prints as `Warning:` with the source line. After all the diagnostics of a run comes one summary line that names the path as you typed it, such as `main.nz: 1 warning(s)` or `main.nz: 1 error(s), 2 warning(s)`. `check` still ends with `ok` and exit code 0 when there are only warnings. The warnings are: `private` on a name that starts with an uppercase letter, which is public anyway; a `package` line that differs from its folder; an import of the package of the same file (see [20.1](20-module-system-and-visibility.md#201-packages)); a `///` documentation comment that is not attached to a declaration (see **Comments** below); and an attribute name the compiler does not know (see [24.1](24-reflection-and-meta-programming.md#241-attributes)). The first one:

```nz warning="starts with an uppercase letter"
package main;

private int Add() {
    return 1;
}

void Main(string[] args) { }
```

The exit code of `nzc` is 0 on success, 1 when the program has errors (or when `-Werror` turned a warning into one), and 2 when the command line itself is wrong (a missing or unknown command or option). When a file has a syntax error, or the loader cannot find a package or a file, `check` prints the errors and the summary line, while `build` and `run` print the errors only, with no summary line, and stop before checking. When the loader cannot read anything at all (a path that does not exist, a folder with no `.nz` files) the message is `nzc: <message>` with exit code 1 and no summary. `-Werror` is accepted by `check`, `build`, `run` and `test`, not by `parse` or `docs`: it prints the same warnings, ends with `1 warning(s) treated as errors`, and exits with 1.

## 0.3 Characteristics, Rules and Naming

> ⚠️ Note: NZ is **case-sensitive**, and the first letter of a name decides the visibility of functions and members, and types are private unless they are written `public` (see [20.3](20-module-system-and-visibility.md#203-access-modifiers)).

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
- **Familiar syntax**, with three rules of its own:
  1. `package a.b;` at the top of every file and `import a.b.C;` replace `namespace`. A folder is a package, and `import` brings in one name, an alias, a group or a whole package (see [20.1](20-module-system-and-visibility.md#201-packages)).
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
NZ has 63 keywords: `import` `package` `void` `public` `private` `class` `struct` `enum` `interface` `var` `new` `this` `if` `else` `while` `for` `foreach` `in` `break` `continue` `return` `switch` `case` `default` `is` `ref` `using` `defer` `unsafe` `true` `false` `null` `abstract` `base` `const` `do` `nameof` `operator` `override` `params` `protected` `readonly` `sealed` `sizeof` `static` `type` `union` `virtual` `where` `async` `await` `move` `lock` `yield` `extern` `volatile` `fixed` `stackalloc` `try` `catch` `finally` `throw` `derive`.
<!-- /GEN -->

[Contents](README.md) · Next: [1. Basic](01-basic.md#1-basic)
