# Standard Library

This page explains the rules that hold for every member of NZ's standard library. The full member list is generated from the compiler's table in [Appendix A of the guide](../appendix-a-standard-library.md#appendix-a-standard-library-reference).

## Packages

The library is split into packages: `std`, `std.io`, `std.collections`, `std.text` and `std.buffers.binary`. `Console` is in `std.io` (`import std.io.Console;`).

| Package | Types |
|---|---|
| `std` | `Environment`, `Math`, `BitConverter`, `StringComparison`, `StringSplitOptions` |
| `std.io` | `Console`, `File`, `Path`, `Directory` |
| `std.collections` | `List<T>`, `Dictionary<TKey,TValue>`, `KeyValuePair<TKey,TValue>`, `HashSet<T>`, `Stack<T>`, `Queue<T>` |
| `std.text` | `StringBuilder`, `Encoding` |
| `std.buffers.binary` | `BinaryPrimitives` |
| `std.diag`, `std.memory` | `Diag`; `Shared<T>`, `Arc<T>`, `WeakReference<T>` |

A type is visible after it is imported (`import std.io.File;`, or a whole package with `import std.collections.*;`). The keyword types (`string`, `char`, `int`, `double`, ...) need no import; their members (`s.Substring(1)`, `int.Parse(s)`, `char.IsDigit(c)`) work.

## When something fails

When a standard library member fails, it stops the program: it writes `nz: <ExceptionType>: <the exception's standard message>` to standard error and exits with status 1, as `Abort` does (no drops, no `defer`). For example:

```
nz: FileNotFoundException: Could not find file '/home/me/data.txt'.
nz: KeyNotFoundException: The given key 'x' was not present in the dictionary.
nz: ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection. (Parameter 'index')
```

(the last from `list.RemoveAt(9)`). Some failures have NZ's own text:

- the indexer of an array or a list (`xs[9]`, `list[9]`): `nz: index out of range`;
- a string position inside a multi-byte character: `nz: ArgumentOutOfRangeException: Index is inside a UTF-8 sequence. (Parameter 'startIndex')` (or the member's own parameter);
- the indexer of a string (`s[9]`) stops with `IndexOutOfRangeException` and its standard text: `nz: IndexOutOfRangeException: Index was outside the bounds of the array.`;
- a culture `StringComparison` value (`CurrentCulture`, `InvariantCulture`, ...), which NZ does not support;
- `File.ReadAllText`/`ReadAllLines` of a file with a UTF-16 or UTF-32 byte order mark: `nz: NotSupportedException: The file '...' starts with a UTF-16 or UTF-32 byte order mark; File.ReadAllText reads UTF-8 only.`

Messages name full paths: a relative path in the program shows as the full path from the working directory, so a documented example that prints such a message uses an absolute path. A program avoids the stop by checking first: `File.Exists(path)`, `d.ContainsKey(k)`, `int.TryParse(s)`. The `File` and `Directory` members that a command-line tool needs also have `Try` twins that return a `Result` instead of stopping (next section).

The `TryX` members return the value as an optional: `int? n = int.TryParse(s);`, `int? v = d.TryGetValue(k);`. The compiler makes you check the optional before you use it.

## Members that return a result: `Try` twins

The throwing-style members stop the program, as above. NZ adds `Try`-prefixed twins for the I/O members a tool needs. A twin takes the same arguments, does the same work, and returns a `Result<T, IOError>` (the built-in `Result` of section [6.2](../06-exceptions-and-error-handling.md#62-handling-errors) of the guide, visible without an import) instead of stopping. With the `?` operator an error is passed on: `string text = File.TryReadAllText(path)?;`.

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

Deleting a file that is not there is not an error: `File.TryDelete` of a missing file is an `Ok`. A member that returns nothing returns `Result<bool, IOError>` whose `Ok` value is always `true` (there is nothing else to report; use `?` or a `switch` on the `Err`). The result owns its string, array or message like any other result (see "Who owns a result"), and an unused result is dropped at the end of its statement.

`IOError` is a small struct with two fields; both `IOError` and `IOErrorKind` are visible in every file without an import:

| Field | Type | Meaning |
|---|---|---|
| `Message` | `string` | exactly the exception's standard message, the text the stopping twin prints after `nz: <ExceptionType>: ` |
| `Kind` | `IOErrorKind` | `NotFound` (a missing file, `FileNotFoundException`), `DirectoryNotFound` (a missing folder on the way, `DirectoryNotFoundException`), `AccessDenied` (`UnauthorizedAccessException`: no permission, or a folder where a file is needed), `AlreadyExists` (an `IOException` whose message says the target already exists, such as `CreateDirectory` over a file), `Other` (every other failure: an empty path, a folder that is not empty, a full disk, ...) |

The checks a member makes on its arguments become errors too: `File.TryReadAllText("")` returns an `Err` of kind `Other` holding `The value cannot be an empty string. (Parameter 'path')`, where `File.ReadAllText("")` stops with that text. Only a failure the OS reports is caught: running out of memory still stops the program.

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
- `StringBuilder.Append` returns the builder itself, so `sb.Append("a").Append(1);` works; the result may be called on again or ignored, not stored (it would be a second owner).
- A collection owns what it holds. `list.Add(x)` and `d[k] = v` move the value in; `RemoveAt`, `Remove`, `Clear` and replacing a value drop what they take out.
- `foreach` borrows: the loop variable cannot be moved out, and the collection cannot change while the loop runs (`collection is borrowed by the loop`). Over a `Dictionary` the variable is a `KeyValuePair` that points into the dictionary; read `kv.Key` and `kv.Value` inside the loop.
- `TryGetValue` and `GetValueOrDefault` copy a value out, so they need values that are copied (numbers, `bool`, `char`, enums, strings, optionals, structs of those). For owned values use `ContainsKey` and the indexer. `GetValueOrDefault(key)` also needs a type whose zero is a value (numbers, `bool`, `char`, `string`, optionals, enums, and structs of those): for a `string` value a missing key gives `""`, because an NZ `string` is never null; a `string?` value gives null. Use `TryGetValue` when you need to tell a missing key from an empty string.
- `KeyValuePair` needs no import for `foreach (var kv in d)`; to write the type out (`foreach (KeyValuePair<string, int> kv in d)`) import it.

## Strings: positions are bytes

NZ strings are UTF-8. Every position counts bytes: `Length`, `IndexOf`, `Substring`, `s[i]`, `PadLeft`, `PadRight`. For ASCII text a position is a character count; for other text a character can take several bytes (`"été".PadRight(8)` adds 3 spaces, because the string is 5 bytes long), as `examples/pe_info` shows for a non-ASCII section name. `s[i]` gives the `char` (a Unicode scalar value) that starts at byte `i`; a position inside a multi-byte character stops the program. `foreach (char c in s)` visits characters. Invalid UTF-8 from outside (files, arguments, standard input) becomes U+FFFD.

## Culture: always ordinal

Comparisons, searches and case changes never depend on the machine's culture: `StartsWith`, `IndexOf`, `CompareTo`, `ToUpper` behave as their `Ordinal`/`Invariant` forms. The overloads that take a `StringComparison` accept `Ordinal` and `OrdinalIgnoreCase` and stop for the culture values. `char.IsLetter` and the other classifications use the Unicode tables.

## `ToString()` of values

A number, `bool`, `char` or enum has `ToString()` with no arguments: the text an interpolation `{x}` gives. An enum value gives its member's name, or its number when no member has that value (`(K)7`). The result is a new `string` that the caller owns, like any other string a member returns. `Console.Write(k)` and `Console.WriteLine(k)` (and the `Console.Error` writers) accept an enum value too and print the same text; NZ has no `object`, so this is a narrow rule for these members (an optional enum is not accepted).

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

- `Dictionary` and `HashSet` visit their entries in insertion order, also after removals. The order is the same on every run and machine.
- `Directory.GetFiles` and `GetDirectories` return their entries sorted ordinally (by UTF-8 bytes), not in file-system order.

Hashing never affects the order. Hashes are keyed with a random seed chosen once per process (from the OS: `getrandom`, `arc4random_buf`, `BCryptGenRandom`), so crafted keys cannot make a table slow. If the OS gives no randomness, a weaker seed made from the time, the process id and addresses is used.

`NZ_HASH_SEED=<decimal number>` fixes the seed, to reproduce a run exactly. It turns the protection off: whoever knows the seed can choose keys that all collide (40,000 such keys take seconds instead of milliseconds), so never set it for a program that reads keys from others. A value that is not a decimal number below 2^64 stops the program.

## Sorting and removing

`List<T>.Sort()` orders ascending; the elements are moved, never copied, so each element keeps its one owner. A `double` or `float` list puts `NaN` first. `List<T>.Remove(item)` removes and drops the first element equal to `item` and returns whether it found one; equality is the one `Contains` and `IndexOf` use. `string.ToCharArray()` is a new `char[]` with one `char` (Unicode scalar value) per character. `double.IsNaN`, `IsInfinity`, `IsFinite`, `IsPositiveInfinity` and `IsNegativeInfinity` (and the same on `float`) test a number.

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

`Dictionary` keys, `HashSet` elements and the elements `List.Contains`/`IndexOf` compare must be an integer type, `char`, `bool`, an enum or `string`. Other types, and type parameters, are not available: they need interfaces (`GetHashCode`/`Equals`), which are not available yet.

## Text encoding and files

`Encoding.UTF8` is the only encoding. `File.ReadAllText` and `ReadAllLines` skip a UTF-8 byte order mark and stop for a UTF-16 or UTF-32 one; `WriteAllText` writes UTF-8 without a byte order mark. `BitConverter` reads and writes little-endian, which every NZ target is (`BitConverter.IsLittleEndian` is `true`); use `BinaryPrimitives.ReverseEndianness` for big-endian fields.

Deleting never follows links: `Directory.Delete(path, true)` removes a link it meets, inside the tree or as `path` itself, and never what the link points to. On Linux and macOS this holds even when another program swaps folders and links while the delete runs: the tree is removed through open folder handles. On Windows the delete goes by path: a folder swapped for a junction between the listing and the step into it would be followed (see the Windows list below).

A recursive delete keeps one open folder handle per level of the tree on Linux and macOS, so a tree deeper than the process's open-file limit (often 1024) stops with `IOException: Too many open files`.

## Notes

| Topic | NZ | Why |
|---|---|---|
| `Console.Error` | only the receiver of `Write`/`WriteLine` | NZ has no `TextWriter` yet |
| `Encoding.UTF8` | only the receiver of its members | UTF-8 is the only encoding |
| `Directory.CreateDirectory` | returns nothing | no `DirectoryInfo` yet |
| `File.ReadAllText` and the other I/O members | they stop the program; `File.TryReadAllText` and the other `Try` twins return `Result<T, IOError>` | NZ has no exceptions to catch; see "Members that return a result" |
| `TryParse`, `TryGetValue` | `int? TryParse(s)`, `TValue? TryGetValue(k)` | NZ has no `out` |
| `BitConverter.ToChar`, `GetBytes(char)` | not offered: `GetBytes('a')` is an error (write `GetBytes((ushort)c)` for two bytes) | an NZ `char` is a Unicode scalar value, not a 16-bit unit |
| `File.ReadAllText`/`ReadAllLines` of a UTF-16 or UTF-32 file | stop with `NotSupportedException` | UTF-8 is the only encoding |
| string positions | UTF-8 bytes | see above |
| culture | comparisons are ordinal and invariant | the same output on every machine |
| `Dictionary`/`HashSet` order | insertion order | see above |
| changing a dictionary inside its `foreach` (`d[k] = v` on an existing key, `Remove`), or `Remove` on a `HashSet` inside its `foreach` | refused when compiling | the loop reads the table in place |
| `List<string>.Sort()` | orders ordinally (UTF-16 code unit order, as `string.CompareOrdinal`) | see "Culture: always ordinal" |
| `List<T>.Sort()`, `Remove`, `Contains`, `IndexOf` | `Sort` for an integer type, `char`, `float`, `double` and `string`; `Remove`, `Contains` and `IndexOf` for a hash key type; others are refused when compiling | interfaces are not available yet |
| `GetValueOrDefault(key)` on `string` values | gives `""` for a missing key | an NZ `string` is never null; its default is the empty string (use `TryGetValue` to get a `null` for a missing key) |
| `var r = q;` inside `if (q.HasValue)` | `r` is an `int`, the type `q` is narrowed to, so `r.HasValue` is an error | NZ narrows the variable itself |
| `ToCharArray()` | one `char` per Unicode scalar value (`"é😀a"` has 3) | an NZ `char` is a Unicode scalar value |
| `ToString()` of a `[Flags]` enum value | the number (`3`), and `[Flags]` gets the warning ``unknown attribute `Flags` `` | attributes have no effect yet |
| `nzc run f.nz -- <args>` | refuses an argument that is not valid UTF-8 (exit code 2; the program does not start) | an NZ `string` is UTF-8 |

## Windows paths not yet proven by CI

The Windows code is written and reviewed but has not run yet: Windows CI is blocked. Until it runs, these are the open points (W1-W12):

| # | What | To prove on Windows |
|---|---|---|
| W1 | `File.Copy` onto another name of the source (hard link, 8.3 short name, case alias, `\\?\` prefix) | refused by file identity, source intact (`file_copy_onto_a_hard_link_of_itself_keeps_the_source`, `file_copy_onto_a_case_alias_keeps_the_source`) |
| W2 | recursive delete is path-based; a junction inside the tree or as `path` is removed, never entered | `directory_delete_recursive_keeps_junction_targets`; a racing junction swap is not covered (see above) |
| W3 | paths of 260 units or more get the `\\?\` prefix | create, read, delete and `GetFullPath` at a 300-unit path with long paths disabled |
| W4 | names with an unpaired surrogate become U+FFFD | `GetFiles`, `File.Exists`, `Directory.Delete` on such a name |
| W5 | `*` or `?` in a path given to `File.Delete`/`Directory.Delete` | nothing removed, with the standard error message |
| W6 | `BCryptGenRandom` is declared by hand and linked with `#pragma comment(lib, "bcrypt.lib")` (MSVC target; MinGW would need `-lbcrypt`) | dictionary programs link; two runs get different seeds |
| W7 | listing with `_wfindfirst64`: an empty folder, a drive root (`C:\`), a UNC share | `GetFiles` on each |
| W8 | drive-relative `C:foo`, UNC roots, `/` and `\` as separators | the `std_path` tests with their Windows expectations |
| W9 | `File.Copy` does not copy the read-only attribute (the Windows `CopyFileW` does) | copy of a read-only file |
| W10 | pattern matching ignores ASCII case only | the pattern fixture on Windows |
| W11 | exact output comparisons with `\r\n` line ends | `example_pe_info_reads_fixture` on Windows |
| W12 | no `-D_POSIX_C_SOURCE` off Linux | Windows and macOS builds compile `nz.h` without errors |

## Split, Format, Keys and Values, Stack and Queue

- **`Split`** takes a separator `char`, a `string`, a `char[]` or a `string[]` (with an optional `StringSplitOptions`), or nothing. A `char[]` or `string[]` splits at any of its elements (the first element of a `string[]` that matches at a position wins; empty strings never match), and no separator or an empty array splits at white space. `Split(params char[])` has no `params` here: pass an array (`s.Split(new char[] { ',', ';' })`).
- **`string.Format(format, args...)`** works for a constant format string (a literal or a `const`): `{index}`, `{index,alignment}`, `{index:format}`, `{{` and `}}`. A format is `F`, `D`, `X` or `x` with an optional precision, the formats an interpolation takes (`N`, `C`, `P`, `E`, `G` and custom formats are not supported yet); an alignment pads with spaces to a width in bytes, right-aligned when positive (`{0,3}` of `"é"` pads with one space, because `é` is two bytes). Every argument is evaluated once, in order. An index or a width of 1,000,000 or more is refused. A malformed format string is a compile-time error carrying the `FormatException` message (`Input string was not in a correct format.`, or `Index (zero based) must be greater than or equal to zero and less than the size of the argument list.`); a format string that is not a constant is refused, since NZ has no `object[]` to carry the arguments: use an interpolation `$"..."`.
- **`Dictionary.Keys` and `Dictionary.Values`** give a new `List` with a copy of each key or value, in insertion order: enumerate it with `foreach`, ask its `Count`, index it. It is a snapshot, so changing the dictionary afterwards does not change it, and the dictionary can change while a `foreach` walks the snapshot. `Values` needs values that are copied (as `TryGetValue` does); for owned values walk the pairs, `foreach (var kv in d)`, and read `kv.Value`.
- **`Stack<T>`** (`Push`, `Pop`, `Peek`, `TryPop`, `TryPeek`, `Count`, `Clear`) and **`Queue<T>`** (`Enqueue`, `Dequeue`, `Peek`, `TryDequeue`, `TryPeek`, `Count`, `Clear`) own their elements as a list does: `Push` and `Enqueue` move the element in, `Pop` and `Dequeue` hand it to the caller, and dropping the collection drops the rest. `Pop`, `Peek`, `Dequeue` on an empty collection stop with `InvalidOperationException: Stack empty.` or `Queue empty.`; the `Try` members give `T?`. `Peek` and `TryPeek` copy, so they need elements that are copied (use `Pop`/`Dequeue` for owned ones). `Dequeue` is O(1). They have no indexer and cannot be walked with `foreach` yet.
