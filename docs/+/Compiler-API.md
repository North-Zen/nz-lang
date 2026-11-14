# Compiler API

The NZ compiler is a library as well as the `nzc` command. Its public interface, `nz_compiler::api` (the library package `nz-compiler` in `compiler/`; the `nzc` command is its own package in `nzc/`), is what the language server and other tools build on, in the way Roslyn serves C#. This page is the contract. The three compilers the project keeps (Rust, C# and the self-hosted NZ one) give the same shape, so a client written against one can move to another.

Version: `0.1.7` (`nz_compiler::api::VERSION`). Every change is listed in `compiler/API-CHANGES.md`. While the version is `0.x`, a breaking change bumps the minor version.

## Positions

- An **offset** is a UTF-8 byte offset into one file, starting at 0. Offsets never cross files.
- A **range** (`TextRange`) is `start..end` in offsets, with `start <= end`. A **location** (`Location`) is a file path plus a range.
- A **line index** (`LineIndex`) turns an offset into a 0-based line and a 0-based column, and back. The column is counted in UTF-8 bytes (`line_col`, `offset`) or in UTF-16 code units (`line_col_utf16`, `offset_utf16`), which is what the Language Server Protocol uses.
- `\n`, `\r\n` and a lone `\r` each end a line, and a line's break belongs to that line. A UTF-8 byte order mark at the start of a file takes no column.
- An offset past the end of the text is the end. An offset inside a character is that character's start. A column past the end of its line is the end of the line, before the break.

## Types and members

Names are snake_case in Rust and NZ and PascalCase in C#. Methods marked "(path)" take a file path. A relative path is read from the workspace root.

| Rust and NZ | C# | What it is | Members |
|---|---|---|---|
| `Workspace` | `Workspace` | The mutable host: unsaved texts over the disk, and the last compilation of each program | `new(root)`, `set_text(path, text)`, `remove_text(path)`, `disk_changed()`, `compilation(entry, mode)`, `forget(entry, mode)` |
| `CheckMode` | `CheckMode` | What a check is for | `Program` (needs `Main`), `Tests` (the `[Test]` functions), `Library` (a package on its own) |
| `Compilation` | `Compilation` | An immutable snapshot of one program: the entry package and what it imports | `diagnostics()`, `files()`, `text(path)`, `line_index(path)`, `mode()`, `syntax_tree(path)`, `semantic_model(path)`, `references(symbol)`, `emit(options)` |
| `Diagnostic` | `Diagnostic` | One problem | `code()`, `severity()`, `is_error()`, `message()`, `location()`, `related()`, `render(compilation)` |
| `DiagnosticCode` | `DiagnosticCode` | `NZ` and four digits | `number()`, text form `NZ1003` |
| `Severity` | `Severity` | | `Error`, `Warning` |
| `SyntaxTree` | `SyntaxTree` | The syntax of one file | `text()`, `tokens()`, `line_index()`, `has_errors()`, `items()`, `declarations()`, `node_at(offset)` |
| `SyntaxToken` | `SyntaxToken` | One token with its trivia | `kind()`, `range()`, `leading()`, `trailing()` |
| `SyntaxTokenKind` | `SyntaxTokenKind` | | `Keyword`, `Identifier`, `Literal`, `Punctuation`, `EndOfFile` |
| `Trivia` | `Trivia` | Text between tokens | `kind()`, `range()` |
| `TriviaKind` | `TriviaKind` | | `Whitespace`, `LineBreak`, `LineComment`, `DocComment`, `BlockComment`, `ByteOrderMark` |
| `SyntaxNode` | `SyntaxNode` | A view of one node | `kind()`, `range()`, `name_range()` |
| `SyntaxKind` | `SyntaxKind` | | `Item`, `Member`, `Statement`, `Expression`, `Name` |
| `SemanticModel` | `SemanticModel` | What the code of one file means | `type_at(offset)`, `symbol_at(offset)`, `declarations()`, `locals()`, `references(symbol)`, `symbols_in_scope(offset)` |
| `Symbol` | `Symbol` | One declaration; two symbols are equal when they come from the same declaration | `kind()`, `name()`, `container()`, `declaration()`, `signature()` |
| `SymbolKind` | `SymbolKind` | | `Type`, `Function`, `Method`, `Field`, `Property`, `Parameter`, `Local`, `LocalFunction`, `EnumVariant`, `Constant`, `Static` |
| `TypeDisplay` | `TypeDisplay` | A type as hover shows it (`List<string>?`) | `text()` |
| `EmitOptions`, `Backend`, `EmitResult` | same | Output of a checked compilation | `Backend::C11`; `EmitResult { c_source, diagnostics }` |
| `run_on_compiler_stack(work)` | `Compiler.RunOnLargeStack` | Runs work on the compiler's large stack | for clients walking deeply nested code |
| `keywords()`, `builtin_type_names()`, `is_identifier(text)` | `Words` | The words of the language | usable keywords (no reserved words), built-in type names, whether a text may be a declared name |

Every public enum may gain variants (in Rust it is `#[non_exhaustive]`), so a client always handles the rest. Structs that only the compiler builds may gain fields.

## Rules a client can count on

- **Syntax trees.** The tokens of a file and their trivia rebuild the file byte for byte.
  - A token's trailing trivia runs to the end of its line, including the line break. All other trivia before a token is its leading trivia. So a comment on a line of its own belongs to the token after it.
  - The last token is `EndOfFile`.
  - A declaration's range starts at its first attribute. Its `///` comment stays leading trivia.
- **Node at an offset.** `node_at` gives a declared name when the offset is in that name, or at its end. Otherwise it gives the innermost expression, statement, member or item.
- **References.** `references` gives each written name: the member name in `a.Name`, a type name wherever a type is written (`Point p`, `Box<int>`, a parameter's type, `new Point()`), and a name an `import` line names. It does not include the declaration itself. A type parameter (`T`) is no type and has no symbol.
- **Nested types.** A type declared inside another is a declaration of its own: its `container()` is the enclosing type's path (`app.Outer`, not the package), `declarations()` lists it right after the members of the type that holds it, `SyntaxTree::items()` lists top-level declarations only, and `SyntaxTree::declarations()` lists every declaration in source order. A written `Outer.Inner` is a reference to each of its segments, so hover, go to definition, references and rename work on `Outer` and on `Inner` separately.
- **Names in scope.** `symbols_in_scope` gives the locals declared before the offset (`is T name` bindings and the variables of B4c patterns, with their types, too; a `foreach` variable over a `Span<T>` has type `T`), the parameters of the functions around it, the members of the type around it, the types declared inside the types around it, the top-level declarations of its package and the names its file imports (an aliased import under the declared name).
- **Reuse.** A workspace gives the same compilation again while no text has changed and no disk change has been reported (`disk_changed`).
- **Emit.** `emit` gives exactly what `nzc` compiles: the program for `Program`, the test build for `Tests`. A `Library` emits nothing. A compilation with errors emits nothing.

## Diagnostic codes

Every diagnostic has a stable code. A code never changes its meaning and is never reused.

| Range | Phase |
|---|---|
| `NZ0xxx` | lexer and parser |
| `NZ1xxx` | names and types |
| `NZ2xxx` | ownership and borrowing |
| `NZ3xxx` | packages and the loader |
| `NZ4xxx` | attributes and tests |
| `NZ5xxx` | constants and statics |
| `NZ9xxx` | internal: `NZ9000` is a compiler crash, or a failure of the C backend that is a compiler bug |

`nzc` prints messages without their codes for now. One kind of message has one code, whatever names it carries or however long the cycle or list it describes is.

## Not yet in the API

- Names in scope from a glob import of a `std` package (`import std.io.*;`).
- Analyzers and source generators.
