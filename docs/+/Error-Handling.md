# Error Handling

This guide explains what NZ code (C# style) does when something goes wrong, today. Read it to judge what is right and what is wrong. Sections 1 to 7 describe what the compiler does now. Section 8 describes a plan that is **not implemented**.

Every statement in sections 1 to 7 is checked by tests:

- Every `nz` block runs through the docs test (see [Compiler](Compiler.md)): a block with `output="..."` is built and run and must print exactly that; a block with `error="..."` must be refused with that message.
- A block that stops the program cannot be an `output` block, because the docs test wants exit code 0. These blocks carry `stderr="..."` (and sometimes `exit="..."` and `stdout="..."`). A second test builds and runs them and compares what the program printed on standard error, its exit code and its standard output with the text in the block.
- Three rows of the table in section 2 are marked with a dagger. They need a huge memory size or a deep call stack, so no test runs them. They were run by hand.

Section 8 is not checked by any test, because the feature does not exist.

## 1. Overview

NZ has two kinds of errors.

1. **Program errors** stop the program at once. They are bugs: an integer that overflows, a division by zero, an index outside an array, a failed `Assert`. The program writes one line to standard error, `nz: <message>`, and exits with code 1. There is nothing to catch: NZ has no `try`, `catch` or `throw` today.
2. **Expected failures** are values. A function that can fail returns `Result<T, E>`: either `Ok` with the value or `Err` with the error. The caller reads it with `switch`, or passes the error on with the `?` operator. The standard library has `Try` members (`File.TryReadAllText`) that return a `Result` instead of stopping.

A third thing looks like an error but is not one: a mistake the compiler can see. Using a `T?` without a check, assigning `null` to a `string`, or a constant that overflows is refused before the program runs:

```nz fragment error="possibly null"
string? s = args.Length > 5 ? "x" : null;
Console.WriteLine(s.Length);
```

```nz fragment error="cannot assign `null` to `string`"
string s = null;
```

```nz fragment error="constant expression overflows `int`"
int n = 2147483647 + 1;
```

## 2. Program errors that stop the program

When the program stops, standard output is flushed first, so what was printed before the stop is not lost. Then `nz: <message>` and a newline go to standard error, and the process exits with code 1. No cleanup runs: owned values are not dropped and `defer` statements do not run (section 6).

The message has one of two shapes:

- **A language error** has no .NET exception name: `nz: integer overflow`. The language itself found the problem.
- **A standard library error** carries the name of the exception .NET would throw, and .NET's message: `nz: KeyNotFoundException: The given key 'x' was not present in the dictionary.` The text after `nz: ` is `<ExceptionName>: <message>`.

In the examples a value is read from `args.Length` (which is 0) so the compiler cannot compute the result itself and refuse it earlier.

### Errors of the language

| Error | Printed on standard error | Example | Exit code |
|---|---|---|---|
| Integer overflow in `+`, `-`, `*` or unary `-` | `nz: integer overflow` | `int n = big + args.Length + 1;` with `big` = 2147483647 | 1 |
| The smallest value divided by `-1` | `nz: integer overflow` | `m / d` with `m` = int.MinValue and `d` = -1 | 1 |
| Division or remainder by zero | `nz: division by zero` | `10 / z`, `10 % z` with `z` = 0 | 1 |
| A value that does not fit the target of a cast | ``nz: value does not fit in `byte` `` (names the target type) | `(byte)big` with `big` = 300 | 1 |
| A value that does not fit after `+=`, `++` and the like | ``nz: value does not fit in `byte` `` | `byte b = 250; b += 10;` | 1 |
| A real that is not a number or too big, cast to an integer | ``nz: value does not fit in `int` `` | `(int)(1.0 / z)` with `z` = 0 | 1 |
| A number cast into an enum outside the enum's underlying type | ``nz: value does not fit in `Level` `` (names the enum) | `(Level)n` with `n` = 300 and `enum Level : byte` | 1 |
| An array or list index outside the collection | `nz: index out of range` | `xs[3]` on an array of 3, `l[1]` on a list of 1 | 1 |
| A negative array length | `nz: negative array length` | `new int[n]` with `n` = -1 | 1 |
| `Value` of an optional that holds `null` | `nz: InvalidOperationException: Nullable object must have a value.` | `n.Value` on an `int?` that is `null` | 1 |
| A `switch` expression where no arm matches | `nz: SwitchExpressionException: Non-exhaustive switch expression failed to match its input.` | `x switch { 1 => "a", 2 => "b" }` with `x` = 3 | 1 |
| A shared object changed while a part of it is lent | `nz: shared object changed while it is lent` | adding to a list through one `Shared` handle while a `foreach` walks it through another | 1 |
| Not enough memory (dagger) | `nz: out of memory` | `new long[2000000000]` on a machine that refuses 16 GB | 1 |
| A string longer than 2,147,483,647 bytes made by an operator (dagger) | `nz: string too long` | `s = s + s` 31 times | 1 |
| Calls nested too deep (dagger) | nothing: the process crashes with a segmentation fault | a function that calls itself forever | the shell shows 139 (the process is killed by a signal, it does not exit with 1) |
| `Abort(message)` | `nz: <message>` | section 3 | 1 |
| A failed `Assert(c, message)` | `nz: assertion failed: <message>` | section 3 | 1 |

The programs for the rows. First the arithmetic ones:

```nz fragment stderr="nz: integer overflow"
int big = 2147483647;
int n = big + args.Length + 1;
Console.WriteLine($"{n}");
```

```nz fragment stderr="nz: integer overflow"
int m = -2147483647 - 1 + args.Length;
int d = -1 - args.Length;
Console.WriteLine($"{m / d}");
```

```nz fragment stderr="nz: division by zero"
int z = args.Length;
Console.WriteLine($"{10 / z}");
```

```nz fragment stderr="nz: division by zero"
int z = args.Length;
Console.WriteLine($"{10 % z}");
```

Casts and narrowing:

```nz fragment stderr="nz: value does not fit in `byte`"
int big = 300 + args.Length;
byte b = (byte)big;
Console.WriteLine($"{b}");
```

```nz fragment stderr="nz: value does not fit in `byte`"
byte b = 250;
b += 10;
Console.WriteLine($"{b}");
```

```nz fragment stderr="nz: value does not fit in `int`"
double z = args.Length;
Console.WriteLine($"{1.0 / z}");
int n = (int)(1.0 / z);
```

```nz stderr="nz: value does not fit in `Level`"
package main;

import std.io.Console;

enum Level : byte { Low, High }

void Main(string[] args) {
    int n = 300 + args.Length;
    Level l = (Level)n;
    Console.WriteLine($"{l}");
}
```

Indexes and lengths. The stop comes after the first line was printed, and that line is not lost:

```nz fragment stderr="nz: index out of range" stdout="before\n"
int[] xs = new int[3];
Console.WriteLine("before");
Console.WriteLine($"{xs[3 + args.Length]}");
```

```nz fragment stderr="nz: index out of range"
var l = new List<int>();
l.Add(1);
Console.WriteLine($"{l[1 + args.Length]}");
```

```nz fragment stderr="nz: negative array length"
int n = -1 - args.Length;
int[] xs = new int[n];
Console.WriteLine($"{xs.Length}");
```

Optionals, `switch` and shared objects:

```nz fragment stderr="nz: InvalidOperationException: Nullable object must have a value."
int? n = args.Length > 5 ? 1 : null;
Console.WriteLine($"{n.Value}");
```

The compiler warns about the `switch` that does not cover every value, and the program stops when it runs:

```nz fragment warning="does not cover every value" stderr="nz: SwitchExpressionException: Non-exhaustive switch expression failed to match its input."
int x = 3 + args.Length;
string s = x switch { 1 => "a", 2 => "b" };
Console.WriteLine(s);
```

```nz stderr="nz: shared object changed while it is lent" stdout="1\n"
package main;

import std.io.Console;
import std.collections.List;
import std.memory.Shared;

class Node {
    List<int> Items = new List<int>();
}

void Main(string[] args) {
    var a = new Shared<Node>(new Node());
    var b = a;
    a.Items.Add(1);
    foreach (var x in a.Items) {
        Console.WriteLine($"{x}");
        b.Items.Add(2);
    }
}
```

The other lend messages (`shared object lent for writing while a part of it is lent for reading` and its relatives) are in [Ownership](Ownership.md#lends-are-checked-when-the-program-runs).

### Errors of the standard library

A standard library member that would throw in .NET stops the program with `nz: <ExceptionName>: <.NET's message>`. Messages that name a path name the full path, as .NET's do. These rows were run:

| Call | Printed on standard error |
|---|---|
| `d["x"]` on a `Dictionary` without that key | `nz: KeyNotFoundException: The given key 'x' was not present in the dictionary.` |
| `d.Add("x", 2)` when `"x"` is there | `nz: ArgumentException: An item with the same key has already been added. Key: x` |
| `int.Parse("abc")` | `nz: FormatException: The input string 'abc' was not in a correct format.` |
| `int.Parse("99999999999")` | `nz: OverflowException: Value was either too large or too small for an Int32.` |
| `Math.Abs(int.MinValue)` | `nz: OverflowException: Negating the minimum value of a twos complement number is invalid.` |
| `"abc".Substring(5)` | `nz: ArgumentOutOfRangeException: startIndex cannot be larger than length of string. (Parameter 'startIndex')` |
| `list.RemoveAt(9)` on an empty list | `nz: ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection. (Parameter 'index')` |
| `"abc"[5]` | `nz: IndexOutOfRangeException: Index was outside the bounds of the array.` |
| `File.ReadAllText("/no-such-file-nz.txt")` | `nz: FileNotFoundException: Could not find file '/no-such-file-nz.txt'.` |

Notice that the index of an array or list stops with the language's own `nz: index out of range`, and the index of a string with .NET's `IndexOutOfRangeException`.

```nz stderr="nz: KeyNotFoundException: The given key 'x' was not present in the dictionary."
package main;

import std.io.Console;
import std.collections.Dictionary;

void Main(string[] args) {
    var d = new Dictionary<string, int>();
    Console.WriteLine($"{d["x"]}");
}
```

```nz stderr="nz: ArgumentException: An item with the same key has already been added. Key: x"
package main;

import std.collections.Dictionary;

void Main(string[] args) {
    var d = new Dictionary<string, int>();
    d.Add("x", 1);
    d.Add("x", 2);
}
```

```nz fragment stderr="nz: FormatException: The input string 'abc' was not in a correct format."
Console.WriteLine($"{int.Parse("abc")}");
```

```nz fragment stderr="nz: OverflowException: Value was either too large or too small for an Int32."
Console.WriteLine($"{int.Parse("99999999999")}");
```

```nz fragment stderr="nz: ArgumentOutOfRangeException: startIndex cannot be larger than length of string. (Parameter 'startIndex')"
string s = "abc";
Console.WriteLine(s.Substring(5 + args.Length));
```

```nz fragment stderr="nz: ArgumentOutOfRangeException: Index was out of range. Must be non-negative and less than the size of the collection. (Parameter 'index')"
var l = new List<int>();
l.RemoveAt(9);
```

```nz fragment stderr="nz: IndexOutOfRangeException: Index was outside the bounds of the array."
string s = "abc";
Console.WriteLine($"{s[5 + args.Length]}");
```

```nz stderr="nz: FileNotFoundException: Could not find file '/no-such-file-nz.txt'."
package main;

import std.io.File;

void Main(string[] args) {
    string text = File.ReadAllText("/no-such-file-nz.txt");
}
```

```nz stderr="nz: OverflowException: Negating the minimum value of a twos complement number is invalid."
package main;

import std.io.Console;
import std.Math;

void Main(string[] args) {
    int m = -2147483647 - 1 + args.Length;
    Console.WriteLine($"{Math.Abs(m)}");
}
```

### Arithmetic is checked, `unchecked` turns it off

C# wraps around on overflow unless the code is `checked`. NZ is the other way round: arithmetic and casts are checked unless the code is written inside `unchecked`, as in C#:

```nz output="44 -2147483648\n"
package main;

import std.io.Console;

void Main(string[] args) {
    int big = 300 + args.Length;
    byte b = unchecked((byte)big);
    int a = 2147483647 + args.Length;
    int c = unchecked(a + 1);
    Console.WriteLine($"{b} {c}");
}
```

`checked(...)` is accepted too and means the default, so `checked(a + 1)` with `a` = 2147483647 stops with `nz: integer overflow`.

Real numbers never stop on division by zero. `1.0 / z` with `z` = 0.0 is `Infinity`, as in C# (the cast of that to `int` is the stop in the table).

### What `Main` can return

`Main` returns `void` or `int`. A program that returns from `void Main` exits with 0. `int Main` exits with the number it returns. Any other return type is refused. `Environment.Exit(code)` ends the program at once with `code`, flushing standard output, and like `Abort` it runs no cleanup.

```nz stderr="" exit="3" stdout="exiting 3\n"
package main;

import std.io.Console;

int Main(string[] args) {
    Console.WriteLine("exiting 3");
    return 3;
}
```

```nz stderr="" exit="4"
package main;

import std.Environment;

void Main(string[] args) {
    Environment.Exit(4);
}
```

```nz error="entry point `Main` must be"
package main;

Result<int, string> Main(string[] args) {
    return new Ok<int>(1);
}
```

A program error is never returned from `Main`: the exit code of a stop is always 1. Use `int Main` for any other exit code.

## 3. Abort and Assert

`Abort` and `Assert` are built-in functions. They need no `import`.

- `Abort(message)` stops the program with `nz: <message>`.
- `Assert(condition, message)` does nothing when the condition is true. When it is false it stops the program with `nz: assertion failed: <message>`.

Both exit with code 1. The message of `Assert` is evaluated only when the condition is false. Use `Assert` for a promise that must hold (a bug if it does not) and `Abort` when the program reaches a place it must never reach. For a failure the caller could handle, return a `Result` (section 4).

```nz stderr="nz: stop here" stdout="start\n"
package main;

import std.io.Console;

void Main(string[] args) {
    Console.WriteLine("start");
    Abort("stop here");
}
```

```nz stderr="nz: assertion failed: need more arguments"
package main;

void Main(string[] args) {
    Assert(args.Length > 5, "need more arguments");
}
```

`Abort` never returns, so a function that returns a value may end with `Abort(...)` and have no `return`. `Abort` and `Assert` are calls with no value, so `var x = Abort("m");` is an error. The README (section 1.7) lists the current limits of how the compiler treats code after `Abort`.

## 4. Result, Ok, Err and the `?` operator

A function that can fail in an expected way returns `Result<T, E>`. It holds either `Ok<T>` with the value or `Err<E>` with the error. `Result`, `Ok` and `Err` need no `import`. They are declared in NZ code (C# style):

```nz-api
public union Result<T, E>(Ok<T>, Err<E>);
public record struct Ok<T>(T Value);
public record struct Err<E>(E Error);
```

`E` can be any type: a `string`, a number, a struct, a class of your own. `new Ok<int>(1)` and `new Err<string>("text")` convert to `Result<int, string>` where one is expected, for example in a `return`. Read a result with `switch`, as with any union (the `switch` statement and the `switch` expression):

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

A `switch` that forgets a case gets a warning. A `Result` is an ordinary value: it is not a stop and nothing forces you to look at it. A discarded `Result` is dropped at the end of its statement (section 6).

### Passing an error on with `?`

Writing `?` after an expression means: if this is an error, leave the function with that error now. On a `Result<T, E>`, `?` gives the `Ok` value, or makes the enclosing function return the `Err`. On a `T?`, `?` gives the value, or makes the function return `null`. C# has no such operator (it throws exceptions); NZ takes it from Rust.

```nz output="digits 1234\nbad character at 2 in 12x4\n"
package main;

import std.io.Console;

class ParseError {
    string Text = "";
    int Position;
}

Result<int, ParseError> ParseDigit(string text, int position) {
    char c = text[position];
    if (c < '0' || c > '9') {
        var e = new ParseError();
        e.Text = text;
        e.Position = position;
        return new Err<ParseError>(e);
    }
    return new Ok<int>((int)c - (int)'0');
}

Result<int, ParseError> Number(string text) {
    int number = 0;
    for (int i = 0; i < text.Length; i++) {
        int digit = ParseDigit(text, i)?;
        number = number * 10 + digit;
    }
    return new Ok<int>(number);
}

string Show(Result<int, ParseError> r) => r switch {
    Ok<int> ok => $"digits {ok.Value}",
    Err<ParseError> e => $"bad character at {e.Error.Position} in {e.Error.Text}",
};

void Main(string[] args) {
    Console.WriteLine(Show(Number("1234")));
    Console.WriteLine(Show(Number("12x4")));
}
```

`?` on an optional works the same way, and on a `string?` too. The function must return an optional:

```nz output="2 -1 -1\n3 -1\n"
package main;

import std.io.Console;

int? Half(int n) {
    if (n % 2 != 0) {
        return null;
    }
    return n / 2;
}

int? Quarter(int n) {
    int h = Half(n)?;
    int q = Half(h)?;
    return q;
}

string? Name(int id) {
    if (id == 1) {
        return "one";
    }
    return null;
}

int? NameLength(int id) {
    string s = Name(id)?;
    return s.Length;
}

void Main(string[] args) {
    Console.WriteLine($"{Quarter(8) ?? -1} {Quarter(6) ?? -1} {Quarter(3) ?? -1}");
    Console.WriteLine($"{NameLength(1) ?? -1} {NameLength(2) ?? -1}");
}
```

### `?` as a statement and in loops

`?` may be a whole statement. `Check()?;` runs the call, leaves the function if it is an error, and throws the `Ok` value away. This suits the `Result<bool, IOError>` of the `Try` members (section 5), whose `Ok` value carries nothing. In a loop, the error leaves the function, not just the loop:

```nz output="valid 3\ninvalid: negative\n"
package main;

import std.io.Console;

Result<bool, string> Check(int n) {
    if (n < 0) {
        return new Err<string>("negative");
    }
    return new Ok<bool>(true);
}

Result<int, string> Validate(int[] values) {
    foreach (var v in values) {
        Check(v)?;
    }
    return new Ok<int>(values.Length);
}

string Show(Result<int, string> r) => r switch {
    Ok<int> ok => $"valid {ok.Value}",
    Err<string> e => $"invalid: {e.Error}",
};

void Main(string[] args) {
    Console.WriteLine(Show(Validate(new int[] { 1, 2, 3 })));
    Console.WriteLine(Show(Validate(new int[] { 1, -2, 3 })));
}
```

Only `?` can be a statement this way. `Check();` alone keeps its `Result` as a discarded value and does not look at it.

### What `?` accepts, and its limits

- **The operand** is a `Result<T, E>` (the built-in one) or a `T?` (an optional: `int?`, `string?`, an optional object, and so on). Anything else is an error.
- **The function** that holds the `?` must return a matching type. For a `Result<T, E>` operand, the function returns a `Result` whose error type is exactly `E`; the success types may differ. There is no conversion between error types. For a `T?` operand, the function returns an optional (any `U?`). A `Result` operand in a function that returns an optional, or the other way round, is an error.
- **`Main` cannot use `?`**, because `Main` returns `void` or `int`.
- **`?` cannot stand inside a `defer`**, nor in a field, constant or static initializer.
- **`?` can stand anywhere an expression can**: in an argument, a condition, an arm of `?:`, `??`, `&&`, `||` or `switch`, in an interpolation hole and in a loop header. In a local function it leaves that local function.
- **Reading `?` next to the conditional operator.** After an expression, a `?` followed by a token that can start an expression is the conditional operator `c ? a : b`, as in C#. So `f()? - 1` is a conditional (and an error without its `:`). Write `(f()?) - 1` to pass the error on and then subtract.
- **`x?` does not make `x` non-null afterwards.**

Each limit is checked:

```nz error="needs the function to return a `Result`; `Main` returns `void`"
package main;

Result<int, string> F() {
    return new Ok<int>(1);
}

void Main(string[] args) {
    int n = F()?;
}
```

```nz error="`?` cannot be used inside `defer`"
package main;

Result<int, string> F() {
    return new Ok<int>(1);
}

Result<int, string> G() {
    defer F()?;
    return new Ok<int>(1);
}

void Main(string[] args) {
}
```

```nz error="needs the function to return a `Result`; `G` returns `int?`"
package main;

Result<int, string> F() {
    return new Ok<int>(1);
}

int? G() {
    int n = F()?;
    return n;
}

void Main(string[] args) {
}
```

```nz error="`?` needs a `Result` or an optional value; found `int`"
package main;

void G() {
    int n = 5;
    int m = n?;
}

void Main(string[] args) {
}
```

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

## 5. The Try twins and IOError

A standard library member that does I/O stops the program when it fails, as .NET's member would throw (section 2). The members a command-line tool needs also have a `Try` twin. A twin takes the same arguments and does the same work, but returns a `Result<T, IOError>` instead of stopping. The `Try` members are NZ's own: .NET has none.

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

A member that returns nothing in .NET returns `Result<bool, IOError>` whose `Ok` value is always `true`. Use `?` or a `switch` on the `Err`. As in .NET, deleting a file that is not there is not an error (`File.TryDelete` of a missing file is `Ok`).

`IOError` is a small struct. `IOError` and `IOErrorKind` are visible in every file without an import.

| Field | Type | Meaning |
|---|---|---|
| `Message` | `string` | exactly the text .NET puts in the exception message; the stopping member prints it after `nz: <ExceptionName>: ` |
| `Kind` | `IOErrorKind` | what went wrong, in the table below |

| `IOErrorKind` | When | .NET exception |
|---|---|---|
| `NotFound` | a file is missing | `FileNotFoundException` |
| `DirectoryNotFound` | a folder on the way is missing | `DirectoryNotFoundException` |
| `AccessDenied` | no permission, or a folder where a file is needed | `UnauthorizedAccessException` |
| `AlreadyExists` | the target already exists, such as `CreateDirectory` over a file | `IOException` |
| `Other` | everything else: an empty path, a folder that is not empty, a full disk | the other exceptions |

The checks .NET makes on its arguments become errors too: `File.TryReadAllText("")` is an `Err` of kind `Other` holding `The value cannot be an empty string. (Parameter 'path')`, where `File.ReadAllText("")` stops with that text. Only a failure the operating system reports is returned. Running out of memory still stops the program.

This program shows each kind on a Linux machine. It prints only the kind, because the messages hold full paths:

```nz output="NotFound\nDirectoryNotFound\nOther\nAccessDenied\nAlreadyExists\nok\n"
package main;

import std.io.Console;
import std.io.File;
import std.io.Directory;

string Kind(Result<string, IOError> r) => r switch {
    Ok<string> ok => "ok",
    Err<IOError> e => $"{e.Error.Kind}",
};

string KindOfCreate(Result<bool, IOError> r) => r switch {
    Ok<bool> ok => "ok",
    Err<IOError> e => $"{e.Error.Kind}",
};

void Main(string[] args) {
    Console.WriteLine(Kind(File.TryReadAllText("no-such-file-nz-guide.txt")));
    Console.WriteLine(Kind(File.TryReadAllText("no-such-folder-nz-guide/a.txt")));
    Console.WriteLine(Kind(File.TryReadAllText("")));
    Console.WriteLine(Kind(File.TryReadAllText("/")));
    Console.WriteLine(KindOfCreate(Directory.TryCreateDirectory("/etc/passwd")));
    Console.WriteLine(KindOfCreate(File.TryDelete("no-such-file-nz-guide.txt")));
}
```

With `?`, a failed read is passed on, and the caller decides:

```nz output="no such file\nok 5\n"
package main;

import std.io.Console;
import std.io.File;

Result<int, IOError> Count(string path) {
    string text = File.TryReadAllText(path)?;
    return new Ok<int>(text.Length);
}

Result<int, IOError> RoundTrip() {
    File.TryWriteAllText("nz-error-guide-tmp.txt", "hello")?;
    int n = Count("nz-error-guide-tmp.txt")?;
    File.TryDelete("nz-error-guide-tmp.txt")?;
    return new Ok<int>(n);
}

string Describe(Result<int, IOError> r) => r switch {
    Ok<int> ok => $"ok {ok.Value}",
    Err<IOError> e => e.Error.Kind == IOErrorKind.NotFound ? "no such file" : $"failed: {e.Error.Message}",
};

void Main(string[] args) {
    Console.WriteLine(Describe(Count("no-such-file-nz-guide.txt")));
    Console.WriteLine(Describe(RoundTrip()));
}
```

## 6. How errors interact with ownership

NZ frees memory without a garbage collector: a value is dropped when its owner leaves scope. Errors keep this rule in the two cases:

- **A `?` that finds an error leaves the function exactly as `return` would.** Every local variable that owns something and every `defer` of every open scope is dropped or run, innermost scope first, and so is what the half-built statement holds. Nothing leaks. The details are in [Ownership: `?` leaves the function like `return`](Ownership.md#-leaves-the-function-like-return).
- **A stop does not clean up.** `Abort`, a failed `Assert` and every runtime error end the process on the spot. Owned values are not dropped, `defer` statements do not run, and the `NZ_ALLOC_REPORT=1` report of live objects is not printed. The operating system takes the memory back when the process ends.

The first program fails in the middle of a loop, and the count of live objects is back to what it was:

```nz output="sum 10\nbad character at 2 in 12x4\nlive objects: 0\n"
package main;

import std.io.Console;
import std.collections.List;
import std.diag.Diag;

class Reading {
    int Value;
}

class ParseError {
    string Text = "";
    int Position;
}

Result<int, ParseError> ParseDigit(string text, int position) {
    char c = text[position];
    if (c < '0' || c > '9') {
        var e = new ParseError();
        e.Text = text;
        e.Position = position;
        return new Err<ParseError>(e);
    }
    return new Ok<int>((int)c - (int)'0');
}

Result<int, ParseError> SumDigits(string text) {
    var seen = new List<Reading>();
    int total = 0;
    for (int i = 0; i < text.Length; i++) {
        var r = new Reading();
        r.Value = ParseDigit(text, i)?;
        total += r.Value;
        seen.Add(r);
    }
    return new Ok<int>(total);
}

string Show(Result<int, ParseError> r) => r switch {
    Ok<int> ok => $"sum {ok.Value}",
    Err<ParseError> e => $"bad character at {e.Error.Position} in {e.Error.Text}",
};

void Main(string[] args) {
    long start = Diag.LiveObjects();
    Console.WriteLine(Show(SumDigits("1234")));
    Console.WriteLine(Show(SumDigits("12x4")));
    Console.WriteLine($"live objects: {Diag.LiveObjects() - start}");
}
```

In `SumDigits("12x4")` the list `seen` holds two `Reading` objects, and the third, `r`, was just made, when `?` leaves. All three are freed, and so is the list.

The error value is moved out, not copied: the `ParseError` made by `ParseDigit` is the same object the caller reads in `Show`, and it is freed when the caller's `Result` is.

A `Result` you do not look at is also dropped. The `Ok` payload or the `Err` payload is freed at the end of the statement:

```nz output="live objects: 0\n"
package main;

import std.io.Console;
import std.diag.Diag;

class Box {
    int N = 1;
}

Result<Box, string> Make(bool ok) {
    if (ok) {
        return new Ok<Box>(new Box());
    }
    return new Err<string>("no");
}

void Main(string[] args) {
    long start = Diag.LiveObjects();
    Make(true);
    Make(false);
    Console.WriteLine($"live objects: {Diag.LiveObjects() - start}");
}
```

The second program shows the other side. After a stop, `defer` does not run and the object is not dropped:

```nz stderr="nz: stopping" stdout=""
package main;

import std.io.Console;

class Box {
    int N = 1;
}

void Main(string[] args) {
    Box b = new Box();
    defer Console.WriteLine("deferred");
    Abort("stopping");
}
```

Nothing named `deferred` is printed.

## 7. Side by side with C#

NZ code (C# style) reads like C#, but its errors work differently. Each row says what C# does and what NZ does.

### Same as C#

| Topic | C# | NZ |
|---|---|---|
| Result of a failed `int.Parse` | throws `FormatException` with the message shown in section 2 | stops the program with the same exception name and message |
| Missing dictionary key | `KeyNotFoundException` | the same name and message, but the program stops |
| `Nullable<T>.Value` of `null` | `InvalidOperationException: Nullable object must have a value.` | the same text, but the program stops |
| Division of reals by zero | `Infinity` or `NaN` | the same |
| `switch` expression with no matching arm | `SwitchExpressionException` | the same name, but the program stops |
| `unchecked(...)` | wraps around | the same |
| Exit code of `Main` | `int Main` returns the process exit code; `void Main` gives 0 | the same |
| Constant that overflows, constant divided by zero | a compile-time error | the same |

### Different from C#

| Topic | C# | NZ |
|---|---|---|
| `try`, `catch`, `finally`, `throw` | exist | **a gap today**: they do not exist, and a program error ends the program. Planned to match C# (section 8) |
| `Exception` and its classes | exist | **a gap today**: `Exception` is an unknown type. Planned to match C# (section 8) |
| An uncaught error | prints a stack trace and ends the program with a code that depends on the system | prints one line, `nz: <message>` or `nz: <ExceptionName>: <message>`, and exits with 1 |
| Cleanup when an error ends the program | `finally` blocks and `using` run while the exception unwinds, if something catches it; with no catch, it is not guaranteed | no cleanup: no drop, no `defer` (section 6) |
| Integer overflow | wraps unless `checked` | stops unless `unchecked` |
| A cast to a narrower integer | wraps unless `checked` | stops unless `unchecked` |
| Division by zero of integers | `DivideByZeroException` | `nz: division by zero` |
| Index outside an array | `IndexOutOfRangeException` | `nz: index out of range` |
| Index outside a `List<T>` | `ArgumentOutOfRangeException` | `nz: index out of range` |
| `null` | any reference type may be `null`; a nullable annotation is only a warning | no `null` in a non-optional type: `string s = null;` is an error; `T?` is an optional that the compiler makes you check before use |
| Using a null reference | `NullReferenceException` at run time | cannot happen for a non-optional; an unchecked `T?` is refused when compiling (only `Nullable` `.Value` can stop, as above) |
| No-throw form of a failing call | `int.TryParse(s, out v)` and similar `Try` members | errors are values: `File.TryReadAllText` returns `Result<string, IOError>`, with `?` to pass the error on. This stays when exceptions arrive (section 8) |
| `File.ReadAllText` of a missing file | throws | **a gap today**: stops the program. Planned to throw as .NET does (section 8) |
| `?` operator | no such operator (`?.` and `??` are other operators) | `x?` leaves the function with the error |
| `int.TryParse(s, out v)` | returns `bool`, value in `out` | `int? n = int.TryParse(s);` returns an optional; there is no `out` |
| Checking a promise | `Debug.Assert` is removed from release builds | `Assert` is built in and always on; `Abort` stops the program with a message |
| Main with another return type, for example `Task` or a result | allowed | `Main` is `void` or `int`; a `Result` cannot be returned |

```nz fragment error="expected Semi"
throw new Exception("x");
```

```nz fragment error="unknown type `Exception`"
Exception e = null;
```

## 8. Planned (not implemented)

This section describes a plan. **None of it works today**, and it carries no promise of a date. It is not checked by any test.

The plan is to give NZ C#'s exceptions, with C# syntax: `throw`, `try`, `catch` and `finally`.

- **Program errors** (an overflow, a division by zero, an index out of range, a failed `Assert`) would throw an exception that a `try`/`catch` can catch. An uncaught exception would still end the program.
- **Standard library members would throw as .NET's do.** `File.ReadAllText` would throw `FileNotFoundException`, a bad `int.Parse` would throw `FormatException`, and so on, instead of stopping the program.
- **The no-throw option stays.** The `Try` twins (`File.TryReadAllText` returning `Result<T, IOError>`, `int.TryParse` returning an optional) and `Result` with the `?` operator remain, for code that wants a plain value instead of an exception. This is the same pair .NET offers with `Parse` and `TryParse`.

A sketch, in a `text` block because it does not compile today:

```text
string ReadOrDefault(string path) {
    try {
        return File.ReadAllText(path);
    }
    catch (FileNotFoundException e) {
        return "";
    }
    finally {
        Console.WriteLine("done");
    }
}

Result<string, IOError> ReadNoThrow(string path) {
    string text = File.TryReadAllText(path)?;
    return new Ok<string>(text);
}
```

When this is done, sections 1 to 7 will change, and this guide will be updated with it.
