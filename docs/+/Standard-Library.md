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
- the indexer of a string (`s[9]`) keeps .NET's name and text: `nz: IndexOutOfRangeException: Index was outside the bounds of the array.`;
- a culture `StringComparison` value (`CurrentCulture`, `InvariantCulture`, ...), which NZ does not support;
- `File.ReadAllText`/`ReadAllLines` of a file with a UTF-16 or UTF-32 byte order mark: `nz: NotSupportedException: The file '...' starts with a UTF-16 or UTF-32 byte order mark; File.ReadAllText reads UTF-8 only.`

Messages name full paths, as .NET's do: a relative path in the program shows as the full path from the working directory, so a documented example that prints such a message uses an absolute path. A program avoids the stop by checking first, as C# code often does: `File.Exists(path)`, `d.ContainsKey(k)`, `int.TryParse(s)`. The `File` and `Directory` members that a command-line tool needs also have `Try` twins that return a `Result` instead of stopping (next section).

C#'s `TryX(..., out T value)` members return the value as an optional instead: `int? n = int.TryParse(s);`, `int? v = d.TryGetValue(k);`. The compiler makes you check the optional before you use it.

## Members that return a result: `Try` twins

Real C# has no such members: .NET throws `FileNotFoundException` and C# code catches it. NZ keeps the throwing-style members as they are (they stop the program, as above) and adds `Try`-prefixed twins for the I/O members a tool needs. A twin takes the same arguments, does the same work, and returns a `Result<T, IOError>` (the built-in `Result` of section 1.7 of the guide, visible without an import) instead of stopping. With the `?` operator an error is passed on: `string text = File.TryReadAllText(path)?;`.

| Member | Returns | Same work as |
|---|---|---|
| `File.TryReadAllText(path)` | `Result<string, IOError>` | `File.ReadAllText` |
| `File.TryReadAllLines(path)` | `Result<string[], IOError>` | `File.ReadAllLines` |
| `File.TryReadAllBytes(path)` | `Result<byte[], IOError>` | `File.ReadAllBytes` |
| `File.TryWriteAllText(path, contents)` | `Result<bool, IOError>` | `File.WriteAllText` |
| `File.TryWriteAllBytes(path, bytes)` | `Result<bool, IOError>` | `File.WriteAllBytes` |
| `File.TryAppendAllText(path, contents)` | `Result<bool, IOError>` | `File.AppendAllText` |
| `File.TryDelete(path)` | `Result<bool, IOError>` | `File.Delete` |
| `Directory.TryCreateDirectory(path)` | `Result<bool, IOError>` | `Directory.CreateDirectory` |
| `Directory.TryGetFiles(path)`, `(path, searchPattern)` | `Result<string[], IOError>` | `Directory.GetFiles` |
| `Directory.TryDelete(path)`, `(path, recursive)` | `Result<bool, IOError>` | `Directory.Delete` |

As in .NET, deleting a file that is not there is not an error: `File.TryDelete` of a missing file is an `Ok`. A member that returns nothing in .NET returns `Result<bool, IOError>` whose `Ok` value is always `true` (there is nothing else to report; use `?` or a `switch` on the `Err`). The result owns its string, array or message like any other result (see "Who owns a result"), and an unused result is dropped at the end of its statement.

`IOError` is a small struct with two fields; both `IOError` and `IOErrorKind` are visible in every file without an import:

| Field | Type | Meaning |
|---|---|---|
| `Message` | `string` | exactly the text .NET puts in the exception message, the text the stopping twin prints after `nz: <ExceptionType>: ` |
| `Kind` | `IOErrorKind` | `NotFound` (a missing file, .NET's `FileNotFoundException`), `DirectoryNotFound` (a missing folder on the way, `DirectoryNotFoundException`), `AccessDenied` (`UnauthorizedAccessException`: no permission, or a folder where a file is needed), `AlreadyExists` (an `IOException` whose message says the target already exists, such as `CreateDirectory` over a file), `Other` (every other failure: an empty path, a folder that is not empty, a full disk, ...) |

The checks .NET makes on its arguments become errors too: `File.TryReadAllText("")` returns an `Err` of kind `Other` holding `The value cannot be an empty string. (Parameter 'path')`, where `File.ReadAllText("")` stops with that text. Only a failure the OS reports is caught: running out of memory still stops the program.

```nz output="no such file\n"
package main;

import std.io.Console;
import std.io.File;

Result<int, IOError> Count(string path) {
    string text = File.TryReadAllText(path)?;
    return new Ok<int>(text.Length);
}

string Describe(Result<int, IOError> r) => r switch {
    Ok<int> ok => $"{ok.Value} characters",
    Err<IOError> e => e.Error.Kind == IOErrorKind.NotFound ? "no such file" : $"failed: {e.Error.Message}",
};

void Main(string[] args) {
    Console.WriteLine(Describe(Count("no-such-file-nz-docs.txt")));
}
```

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

## `ToString()` of values

A number, `bool`, `char` or enum has `ToString()` with no arguments, as in C#: the text an interpolation `{x}` gives. An enum value gives its member's name, or its number when no member has that value (`(K)7`). The result is a new `string` that the caller owns, like any other string a member returns. `Console.Write(k)` and `Console.WriteLine(k)` (and the `Console.Error` writers) accept an enum value too and print the same text; NZ has no `object`, so this is a narrow rule for these members (an optional enum is not accepted).

```nz output="a 5 B 1 7\n"
package main;
import std.io.Console;

enum K { A, B = 5 }

void Main(string[] args) {
    char c = 'a';
    int n = 5;
    K k = K.B;
    Console.WriteLine($"{c.ToString()} {n.ToString()} {k.ToString()} {K.A.ToString().Length} {((K)7).ToString()}");
}
```

## Order you can rely on

- `Dictionary` and `HashSet` visit their entries in insertion order, also after removals. .NET's order is unspecified (it reuses removed slots), so a C# program that depends on it is already fragile; an NZ program gets the same order on every run and machine.
- `Directory.GetFiles` and `GetDirectories` return their entries sorted ordinally (by UTF-8 bytes), not in file-system order.

Hashing never affects the order. Hashes are keyed with a random seed chosen once per process (from the OS: `getrandom`, `arc4random_buf`, `BCryptGenRandom`), so crafted keys cannot make a table slow. If the OS gives no randomness, a weaker seed made from the time, the process id and addresses is used.

`NZ_HASH_SEED=<decimal number>` fixes the seed, to reproduce a run exactly. It turns the protection off: whoever knows the seed can choose keys that all collide (40,000 such keys take seconds instead of milliseconds), so never set it for a program that reads keys from others. A value that is not a decimal number below 2^64 stops the program.

## Sorting and removing

`List<T>.Sort()` orders ascending; the elements are moved, never copied, so each element keeps its one owner. A `double` or `float` list puts `NaN` first, as .NET does. `List<T>.Remove(item)` removes and drops the first element equal to `item` and returns whether it found one; equality is the one `Contains` and `IndexOf` use. `string.ToCharArray()` is a new `char[]` with one `char` (Unicode scalar value) per character. `double.IsNaN`, `IsInfinity`, `IsFinite`, `IsPositiveInfinity` and `IsNegativeInfinity` (and the same on `float`) are as in .NET.

```nz output="c a b|2|3|True\n"
package main;
import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    var names = new List<string>();
    names.Add("b");
    names.Add("c");
    names.Add("a");
    names.Sort();
    bool removed = names.Remove("c");
    char[] letters = "abc".ToCharArray();
    Console.WriteLine($"{letters[2]} {names[0]} {names[1]}|{names.Count}|{letters.Length}|{removed}");
}
```

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
| `File.ReadAllText` and the other I/O members throw; C# code catches the exception | they stop the program; `File.TryReadAllText` and the other `Try` twins return `Result<T, IOError>` (not in .NET) | NZ has no exceptions to catch; see "Members that return a result" |
| `TryParse(s, out v)`, `TryGetValue(k, out v)` | `int? TryParse(s)`, `TValue? TryGetValue(k)` | NZ has no `out` |
| `BitConverter.ToChar`, `GetBytes(char)` | not offered: `GetBytes('a')` is an error (write `GetBytes((ushort)c)` for C#'s two bytes) | a C# `char` is a UTF-16 unit, an NZ `char` a scalar value |
| `File.ReadAllText`/`ReadAllLines` decode UTF-16 and UTF-32 files (by their byte order mark) | stop with `NotSupportedException` | UTF-8 is the only encoding |
| string positions in UTF-16 units | UTF-8 bytes | see above |
| culture-sensitive defaults | ordinal and invariant | the same output on every machine |
| `Dictionary`/`HashSet` order unspecified | insertion order | see above |
| changing a dictionary inside its `foreach` (`d[k] = v` on an existing key, `Remove`), or `Remove` on a `HashSet` inside its `foreach` | refused when compiling | the loop reads the table in place |
| `List<string>.Sort()` orders by the culture | orders ordinally (UTF-16 code unit order, as `string.CompareOrdinal`) | see "Culture: always ordinal" |
| `List<T>.Sort()` for any `T` that is `IComparable`; `Remove` and `Contains` for any `T` with `Equals` | `Sort` for an integer type, `char`, `float`, `double` and `string`; `Remove`, `Contains` and `IndexOf` for a hash key type; others are refused when compiling | interfaces come later |
| `GetValueOrDefault(key)` on `string` values gives null | refused; use `TryGetValue` or the two-argument form | an NZ `string` is never null |
| `var r = q;` inside `if (q.HasValue)` makes `r` an `int?` | `r` is an `int`, the type `q` is narrowed to, so `r.HasValue` is an error | NZ narrows the variable itself |
| `ToCharArray()` gives one `char` per UTF-16 code unit (`"é😀a"` has 4) | one `char` per Unicode scalar value (`"é😀a"` has 3) | a C# `char` is a UTF-16 unit, an NZ `char` a scalar value |
| `ToString()` of a `[Flags]` enum value names each flag (`R, W`) | the number (`3`), and `[Flags]` gets the warning ``unknown attribute `Flags` `` | attributes have no effect yet |
| `dotnet run -- <args>` passes any argument the system gives | `nzc run f.nz -- <args>` refuses an argument that is not valid UTF-8 (exit code 2; the program does not start) | an NZ `string` is UTF-8 |

## Windows paths not yet proven by CI

The Windows code is written and reviewed but has not run yet: Windows CI is blocked. Until it runs, these are the open points (W1-W12):

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
