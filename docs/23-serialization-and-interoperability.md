# 23. Serialization and interoperability

> NZ is still in development and is not stable. Anything in this documentation can change.

## 23.1 C interop

### C layout: `[StructLayout]`

`[StructLayout(LayoutKind.Sequential)]` on a struct of unmanaged fields gives it C's layout: the fields in declaration order with C alignment. `Pack = n` (0 for the natural layout, 1, 2, 4, 8, 16, 32, 64, 128) lowers the alignment. The generated C checks every offset with `_Static_assert(offsetof(...))`. A struct used in a C signature must carry the attribute. The attributes come from `std.interop` (`import std.interop.*;`).

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

A field that `Pack` puts below its natural alignment is used by name only: its address would be a misaligned pointer, so `&`, `fixed`, a `ref` or `in` argument, a `this ref` receiver and a method call on it are not supported yet. `char` cannot be a field of a laid-out struct (NZ's `char` is 4 bytes); use `byte`, `ushort` or `uint`. `LayoutKind.Explicit`, `[FieldOffset]`, `LayoutKind.Auto`, and the attribute on a class or a generic struct are not supported yet.

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

A C function is declared at the top level with `[DllImport]` and `extern`, without `static` (top-level functions take none) and without a body. The first argument names the library: `"c"` and `"m"` add nothing, any other name links `-l<name>`; `nzc build`, `run` and `test` take `-L <dir>` (repeatable) for a library outside the system folders. `EntryPoint = "name"` gives the C name when it differs. Every call is written inside `unsafe`, even without a pointer in the signature, because nothing checks the C side.

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

A C signature takes the integer types, `float`, `double`, `bool`, pointers, enums (as their underlying type) and `[StructLayout]` structs by value or by pointer, and `void` as a result; `ref` and `in` are passed as pointers. Not `string`, classes, arrays, optionals, tuples, or `char` (use `byte` or `uint` for a C `char`), and no variable argument list. Two sizes to watch: NZ `bool` is C's one-byte `bool`, and NZ `long` is always 64 bits, while C's `long` is 32 bits on Windows.

Names are checked because they are written into the generated C and the link line. A C name must be a C identifier, not a C keyword and not `main`; names the generated code or the runtime use are refused. A library must be a plain name (letters, digits, `_`, `.`, `+`, `-`); a path is not supported yet. The C name is given to the linker as is, which works on Linux and Windows x64; a target that prefixes C symbols with `_` (macOS, 32-bit Windows) is not supported yet.

### Functions C can call: `[UnmanagedCallersOnly]`

`[UnmanagedCallersOnly(EntryPoint = "add_i32")] int AddI32(int a, int b) { ... }` at the top level is a C function of exactly that name. `EntryPoint` is required; the signature follows the rules above, without `ref` or `in` (take a pointer); NZ code cannot call it. An export has external linkage and would replace a C function of the same name for the whole program, so its name may not be one C keeps:
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

`nzc check`, `nzc build` and `nzc run` take `--no-runtime`: the program links the C library only, and its C includes `nz_core.h` (checked arithmetic and its failure path, optionals of the built-in types, `NativeMemory`) instead of `nz.h`. The entry point is `void Main()` or `int Main()` (the `int` is the exit status); `string[] args` needs the runtime. A `string` value or interpolation, `Console`, `Diag`, a class (also through its static members), an array, a `List<T>` or a handle, `Abort` and `Assert` are refused, each with ``needs the NZ runtime; not available with --no-runtime``. Numbers, `bool`, `char`, enums, structs of those, pointers, statics, local functions, `extern` calls, `stackalloc` and `NativeMemory` work, and integer arithmetic stays checked: an overflow prints `nz: integer overflow` and exits with 1.

Text for C is a UTF-8 literal, `"hi\n"u8` (or `U8`, also after a raw `"""..."""`). A raw literal that spans lines holds the source's line breaks, which the repository keeps as `\n` on every system (`.gitattributes`). Its type in NZ is `byte*`: a pointer to static, read-only bytes ending with a NUL, usable only inside `unsafe`, with or without the runtime. The bytes are read-only: writing through the pointer is undefined and usually crashes.

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

Each of these is refused with a message that says so: function pointers (`delegate*`), `LayoutKind.Explicit` and `[FieldOffset]`, a variable `stackalloc` length, `volatile` fields, pointers as type arguments, array elements, tuple elements or optional values, a C function with a variable argument list (C's `...`), `[LibraryImport]` (use `[DllImport]`), `fixed` on a `string`, `CallingConvention` and the other `DllImport` fields (the calling convention is always C's), and a library given as a path.

## 23.2 Serialization

This section is not written yet.

Previous: [22. File system and OS interaction](22-file-system-and-os-interaction.md#22-file-system-and-os-interaction) · [Contents](README.md) · Next: [24. Reflection and meta-programming](24-reflection-and-meta-programming.md#24-reflection-and-meta-programming)
