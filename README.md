<p align="center">
  <img src="assets/nz-logo-filled.svg" alt="NZ" width="650">
</p>

# NZ

**NZ** (North Zen) is a systems programming language that is memory safe and has **no garbage collector by default**. Its syntax deliberately follows C#, so it reads naturally to anyone who knows C#, while its memory model is built on **ownership** and **borrowing**: a strong architecture with a syntax that is easy to read.

NZ is designed for many kinds of programs: kernels (`no_runtime`), web/app servers, and games. NZ programs compile to C and run through `clang`. Before Milestone 0.1 the compiler is written in Rust and translates NZ to C; this describes the current implementation.

> NZ is still in development and is not stable. Anything in this documentation can change.

This repository holds the language documentation only; the compiler source is not public yet. Supported platform for now: Linux x86_64.

```nz
package main;

import std.io.Console;

void Main(string[] args) {
    Console.WriteLine("Hello, NZ");
}
```

## Why NZ

- **Memory safe without a GC.** Ownership and borrowing are checked at compile time.
- **Readable.** A familiar syntax: `class`, `struct`, `interface`, generics, properties, `var`.
- **Visibility from the name.** `Add` is visible outside its package, `add` is local.
- **No surprise nulls.** `null` is valid only for optional types (`T?`), and a `T?` must be checked before use.
- **Layered memory model.** No GC at the core, opt-in `Arc<T>` for shared data.

## Documentation

- [Language guide](docs/README.md): the full tour of the language. Every `nz` code block in it is checked by the real compiler.
- [Syntax reference](docs/+/Syntax-Reference.md): every construct with its syntax and an example
- [Ownership](docs/+/Ownership.md), [Memory model](docs/+/Memory-Model.md), [Null safety](docs/+/Null-Safety.md)
- [Standard library](docs/+/Standard-Library.md)
- [Compiler notes](docs/+/Compiler.md) and [Compiler API](docs/+/Compiler-API.md)

These pages are published from the development repository after each milestone of work, so they describe what the compiler does today.

## License

Apache License 2.0, see [LICENSE](LICENSE). Programs you compile with NZ are yours, under any license you choose. The name "NZ" and its logos are not covered by that license.
