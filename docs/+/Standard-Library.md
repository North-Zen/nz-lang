# Standard Library

NZ's standard library follows .NET: the same type names, member names, overloads and messages, so NZ code (C# style) reads like C#. This page explains the rules that hold for every member. The full member list is generated from the compiler's table in [section 6 of the README](../README.md#6-standard-library).

## Packages

A .NET namespace becomes a package: `System` is `std`, `System.IO` is `std.io`, `System.Collections.Generic` is `std.collections`, `System.Text` is `std.text`, `System.Buffers.Binary` is `std.buffers.binary`. One exception: `Console`, which .NET has in `System`, is in `std.io` (`import std.io.Console;`), where NZ programs have always found it.

| Package | Types |
|---|---|
| `std` | `Environment`, `Math`, `BitConverter`, `StringComparison`, `StringSplitOptions` |
| `std.io` | `Console`, `File`, `Path`, `Directory` |
| `std.collections` | `List<T>`, `Dictionary<TKey,TValue>`, `KeyValuePair<TKey,TValue>`, `HashSet<T>` |
| `std.text` | `StringBuilder`, `Encoding` |
| `std.buffers.binary` | `BinaryPrimitives` |
| `std.diag`, `std.memory` | `Diag`; `Shared<T>`, `Arc<T>`, `WeakReference<T>` |

A type is visible after it is imported (`import std.io.File;`, or a whole package with `import std.collections.*;`). The keyword types (`string`, `char`, `int`, `double`, ...) need no import; their members (`s.Substring(1)`, `int.Parse(s)`, `char.IsDigit(c)`) work as in C#.

## When something fails

Where .NET throws, a standard library member stops the program: it writes `nz: <ExceptionType>: <.NET's message>` to standard error and exits with status 1, as `Abort` does (no drops, no `defer`). For example:

```
nz: FileNotFoundException: Could not find file '/home/me/data.txt'.
nz: KeyNotFoundException: The given key 'x' was not present in the dictionary.
nz: ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection. (Parameter 'index')
```

(the last from `list.RemoveAt(9)`). Some failures have NZ's own text, because .NET's situation does not exist or NZ checks something .NET does not:

- the indexer of an array or a list (`xs[9]`, `list[9]`): `nz: index out of range` (.NET: `IndexOutOfRangeException`, `ArgumentOutOfRangeException`);
- a string position inside a multi-byte character: `nz: ArgumentOutOfRangeException: Index is inside a UTF-8 sequence. (Parameter 'startIndex')` (or the member's own parameter);
- a culture `StringComparison` value (`CurrentCulture`, `InvariantCulture`, ...), which NZ does not support;
- `File.ReadAllText`/`ReadAllLines` of a file with a UTF-16 or UTF-32 byte order mark: `nz: NotSupportedException: The file '...' starts with a UTF-16 or UTF-32 byte order mark; File.ReadAllText reads UTF-8 only.`

Messages name full paths, as .NET's do. A program avoids the stop by checking first, as C# code often does: `File.Exists(path)`, `d.ContainsKey(k)`, `int.TryParse(s)`. Members that return a result instead of failing are a later batch (once `Result` exists).

C#'s `TryX(..., out T value)` members return the value as an optional instead: `int? n = int.TryParse(s);`, `int? v = d.TryGetValue(k);` (decision S2). The compiler makes you check the optional before you use it.

## Who owns a result

- A string, array, list or other object a member returns is new: the caller owns it, and changing it never changes the argument (`Substring`, `Split`, `ToArray`, `File.ReadAllBytes`).
- `StringBuilder.Append` returns the builder itself, so `sb.Append("a").Append(1);` works as in C#; the result may be called on again or ignored, not stored (it would be a second owner).
- A collection owns what it holds. `list.Add(x)` and `d[k] = v` move the value in; `RemoveAt`, `Remove`, `Clear` and replacing a value drop what they take out.
- `foreach` borrows: the loop variable cannot be moved out, and the collection cannot change while the loop runs (`collection is borrowed by the loop`). Over a `Dictionary` the variable is a `KeyValuePair` that points into the dictionary; read `kv.Key` and `kv.Value` inside the loop.
- `TryGetValue` and `GetValueOrDefault` copy a value out, so they need values that are copied (numbers, `bool`, `char`, enums, strings, optionals, structs of those). For owned values use `ContainsKey` and the indexer. `GetValueOrDefault(key)` also needs a type whose zero is a value: .NET's default of a `string` is null, which an NZ `string` cannot hold, so use `TryGetValue` or `GetValueOrDefault(key, fallback)` there; a `string?` value is fine (its default is null).
- `KeyValuePair` needs no import for `foreach (var kv in d)`; to write the type out (`foreach (KeyValuePair<string, int> kv in d)`) import it.

## Strings: positions are bytes

NZ strings are UTF-8. Every position counts bytes: `Length`, `IndexOf`, `Substring`, `s[i]`, `PadLeft`, `PadRight`. For ASCII text this is exactly C#'s answer; for other text widths differ (`"été".PadRight(8)` adds 3 spaces, C# 5), as `examples/pe_info` shows for a non-ASCII section name. `s[i]` gives the `char` (a Unicode scalar value) that starts at byte `i`; a position inside a multi-byte character stops the program. `foreach (char c in s)` visits characters. Invalid UTF-8 from outside (files, arguments, standard input) becomes U+FFFD, as .NET's decoder does.

## Culture: always ordinal

Comparisons, searches and case changes never depend on the machine's culture: `StartsWith`, `IndexOf`, `CompareTo`, `ToUpper` behave as their `Ordinal`/`Invariant` forms. The overloads that take a `StringComparison` accept `Ordinal` and `OrdinalIgnoreCase` and stop for the culture values. `char.IsLetter` and the other classifications use the Unicode tables .NET uses.

## Order you can rely on

- `Dictionary` and `HashSet` visit their entries in insertion order, also after removals. .NET's order is unspecified (it reuses removed slots), so a C# program that depends on it is already fragile; an NZ program gets the same order on every run and machine.
- `Directory.GetFiles` and `GetDirectories` return their entries sorted ordinally (by UTF-8 bytes), not in file-system order.

Hashing never affects the order. Hashes are keyed with a random seed chosen once per process (from the OS: `getrandom`, `arc4random_buf`, `BCryptGenRandom`), so crafted keys cannot make a table slow. If the OS gives no randomness, a weaker seed made from the time, the process id and addresses is used.

`NZ_HASH_SEED=<decimal number>` fixes the seed, to reproduce a run exactly. It turns the protection off: whoever knows the seed can choose keys that all collide (40,000 such keys take seconds instead of milliseconds), so never set it for a program that reads keys from others. A value that is not a decimal number below 2^64 stops the program.

## Keys and elements

`Dictionary` keys, `HashSet` elements and the elements `List.Contains`/`IndexOf` compare must be an integer type, `char`, `bool`, an enum or `string`. Other types, and type parameters, need interfaces (`GetHashCode`/`Equals`), which come later.

## Text encoding and files

`Encoding.UTF8` is the only encoding. `File.ReadAllText` and `ReadAllLines` skip a UTF-8 byte order mark and stop for a UTF-16 or UTF-32 one; `WriteAllText` writes UTF-8 without a byte order mark. `BitConverter` reads and writes little-endian, which every NZ target is (`BitConverter.IsLittleEndian` is `true`); use `BinaryPrimitives.ReverseEndianness` for big-endian fields.

Deleting never follows links: `Directory.Delete(path, true)` removes a link it meets, inside the tree or as `path` itself, and never what the link points to. On Linux and macOS this holds even when another program swaps folders and links while the delete runs: the tree is removed through open folder handles. On Windows the delete goes by path, like .NET's: a folder swapped for a junction between the listing and the step into it would be followed (see the Windows list below).

A recursive delete keeps one open folder handle per level of the tree on Linux and macOS, so a tree deeper than the process's open-file limit (often 1024) stops with `IOException: Too many open files`. .NET is limited by the path length instead.

## Differences from .NET

| .NET | NZ | Why |
|---|---|---|
| `Console.Error` is a `TextWriter` value | only the receiver of `Write`/`WriteLine` | NZ has no `TextWriter` yet |
| `Encoding.UTF8` is an `Encoding` value | only the receiver of its members | UTF-8 is the only encoding |
| `Directory.CreateDirectory` returns a `DirectoryInfo` | returns nothing | no `DirectoryInfo` yet |
| `TryParse(s, out v)`, `TryGetValue(k, out v)` | `int? TryParse(s)`, `TValue? TryGetValue(k)` | NZ has no `out` |
| `BitConverter.ToChar`, `GetBytes(char)` | not offered: `GetBytes('a')` is an error (write `GetBytes((ushort)c)` for C#'s two bytes) | a C# `char` is a UTF-16 unit, an NZ `char` a scalar value |
| `File.ReadAllText`/`ReadAllLines` decode UTF-16 and UTF-32 files (by their byte order mark) | stop with `NotSupportedException` | UTF-8 is the only encoding |
| string positions in UTF-16 units | UTF-8 bytes | see above |
| culture-sensitive defaults | ordinal and invariant | the same output on every machine |
| `Dictionary`/`HashSet` order unspecified | insertion order | see above |
| changing a dictionary inside its `foreach` (`d[k] = v` on an existing key, `Remove`), or `Remove` on a `HashSet` inside its `foreach` | refused when compiling | the loop reads the table in place |
| `GetValueOrDefault(key)` on `string` values gives null | refused; use `TryGetValue` or the two-argument form | an NZ `string` is never null |

## Windows paths not yet proven by CI

The Windows code is written and reviewed but has not run yet: Windows CI is blocked. Until it runs, these are the open points (final review, W1-W12):

| # | What | To prove on Windows |
|---|---|---|
| W1 | `File.Copy` onto another name of the source (hard link, 8.3 short name, case alias, `\\?\` prefix) | refused by file identity, source intact (`file_copy_onto_a_hard_link_of_itself_keeps_the_source`, `file_copy_onto_a_case_alias_keeps_the_source`) |
| W2 | recursive delete is path-based; a junction inside the tree or as `path` is removed, never entered | `directory_delete_recursive_keeps_junction_targets`; a racing junction swap is not covered (see above) |
| W3 | paths of 260 units or more get the `\\?\` prefix | create, read, delete and `GetFullPath` at a 300-unit path with long paths disabled |
| W4 | names with an unpaired surrogate become U+FFFD | `GetFiles`, `File.Exists`, `Directory.Delete` on such a name |
| W5 | `*` or `?` in a path given to `File.Delete`/`Directory.Delete` | nothing removed, .NET's message |
| W6 | `BCryptGenRandom` is declared by hand and linked with `#pragma comment(lib, "bcrypt.lib")` (MSVC target; MinGW would need `-lbcrypt`) | dictionary programs link; two runs get different seeds |
| W7 | listing with `_wfindfirst64`: an empty folder, a drive root (`C:\`), a UNC share | `GetFiles` on each |
| W8 | drive-relative `C:foo`, UNC roots, `/` and `\` as separators | the `std_path` tests with their Windows expectations |
| W9 | `File.Copy` does not copy the read-only attribute (.NET's `CopyFileW` does) | copy of a read-only file |
| W10 | pattern matching ignores ASCII case only | the pattern fixture against .NET on Windows |
| W11 | exact output comparisons with `\r\n` line ends | `example_pe_info_reads_fixture` on Windows |
| W12 | no `-D_POSIX_C_SOURCE` off Linux | Windows and macOS builds compile `nz.h` without errors |
