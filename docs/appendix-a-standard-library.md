# Appendix A. Standard library reference

> NZ is still in development and is not stable. Anything in this documentation can change.

This section is written automatically from the compiler's table of builtin types. A builtin type is only visible after it is imported. The rules every member follows (what happens on failure, who owns a result, string positions, ordering) are in [Standard Library](%2B/Standard-Library.md).

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
| `GetFiles` (static) | `string[] GetFiles(string)`<br>`string[] GetFiles(string, string)` | The files in the folder matching `searchPattern` (`*`, `?`, `*.*` for all; it may start with a sub folder), sorted ordinally. |
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
| `TryGetValue` | `TValue? TryGetValue(TKey)` | A copy of the key's value, or null when the key is missing; only for values that are copied. |
| `GetValueOrDefault` | `TValue GetValueOrDefault(TKey)`<br>`TValue GetValueOrDefault(TKey, TValue)` | A copy of the key's value, or `defaultValue`, or the zero value (`0`, `false`, null); only for values that are copied, and the one-argument form not for `string` values. |
| `Keys` | `List<TKey> Keys { get; }` | The keys in insertion order, as a new list of copies (a snapshot: later changes of the dictionary do not reach it). |
| `Values` | `List<TValue> Values { get; }` | The values in insertion order, as a new list of copies (a snapshot); only for values that are copied. |
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

#### `Stack<T>`

A last-in, first-out collection that owns its elements.

```nz-api
import std.collections.Stack;
```

| Member | Signature | Description |
|---|---|---|
| constructor | `new Stack<T>()`<br>`new Stack<T>(int capacity)` | An empty stack, or an empty one with room for `capacity` elements (a negative capacity stops). |
| `Push` | `void Push(T)` | Puts the element on top; the stack owns it from then on. |
| `Pop` | `T Pop()` | Removes the top element and gives it to the caller; an empty stack stops (`InvalidOperationException`: Stack empty.). |
| `Peek` | `T Peek()` | A copy of the top element (elements that are copied only); an empty stack stops. |
| `TryPop` | `T? TryPop()` | Removes the top element and gives it to the caller, or null when the stack is empty. |
| `TryPeek` | `T? TryPeek()` | A copy of the top element, or null when the stack is empty (elements that are copied only). |
| `Clear` | `void Clear()` | Removes and drops every element. |
| `Count` | `int Count { get; }` | The number of elements. |

#### `Queue<T>`

A first-in, first-out collection that owns its elements.

```nz-api
import std.collections.Queue;
```

| Member | Signature | Description |
|---|---|---|
| constructor | `new Queue<T>()`<br>`new Queue<T>(int capacity)` | An empty queue, or an empty one with room for `capacity` elements (a negative capacity stops). |
| `Enqueue` | `void Enqueue(T)` | Adds the element at the end; the queue owns it from then on. |
| `Dequeue` | `T Dequeue()` | Removes the first element and gives it to the caller; an empty queue stops (`InvalidOperationException`: Queue empty.). |
| `Peek` | `T Peek()` | A copy of the first element (elements that are copied only); an empty queue stops. |
| `TryDequeue` | `T? TryDequeue()` | Removes the first element and gives it to the caller, or null when the queue is empty. |
| `TryPeek` | `T? TryPeek()` | A copy of the first element, or null when the queue is empty (elements that are copied only). |
| `Clear` | `void Clear()` | Removes and drops every element. |
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
| `PI` (static) | `double PI { get; }` | π (a read-only value). |
| `E` (static) | `double E { get; }` | e (a read-only value). |
| `Tau` (static) | `double Tau { get; }` | 2π (a read-only value). |

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

`Result<T, E>`, `Ok<T>` and `Err<E>` are written in NZ code and are visible in every program without an `import`; a program's own type of the same name hides them. They are described with the errors chapter, section [6.2](06-exceptions-and-error-handling.md#62-handling-errors).

Previous: [25. Ecosystem and frameworks](25-ecosystem-and-frameworks.md#25-ecosystem-and-frameworks) · [Contents](README.md) · Next: [Appendix B. Current behavior notes](appendix-b-current-behavior.md#appendix-b-current-behavior-notes)
