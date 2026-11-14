# 6. Exceptions and error handling

> NZ is still in development and is not stable. Anything in this documentation can change.

The complete guide is [Error Handling](+/Error-Handling.md).

The complete guide to errors, with every runtime stop, the `Try` members and the rules, is [Error Handling](+/Error-Handling.md). This section is the short form.

## 6.1 Creating errors

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

**Current limits.** For the check that every path returns a value, `Abort` ends the path. For the other flow analyses it does not, except null narrowing: definite assignment and move analysis treat the code after an `Abort` as reachable (after `if (x == null) { Abort("x"); }`, `x` is not null). Valid guard code can be rejected because of this. Here the path that moves `n` stops at `Abort`, yet the last line is an error:

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

## 6.2 Handling errors

See [Error Handling: `try` and `finally`](+/Error-Handling.md#try-and-finally) and [Syntax Reference](+/Syntax-Reference.md).

### `Result`

A function that can fail returns `Result<T, E>`: either `Ok<T>` with the value or `Err<E>` with the error. They are declared in NZ code in the built-in package `std.result` and need no `import`:

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

A `switch` that forgets a case gets the warning of section [1.6.1](01-basic-operators-control-flow.md#161-conditionals), and a type that is not a case is an error:

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

`Ok` and `Err` are positional `record struct`s: `record struct Point(int X, int y);` declares a struct whose public fields are its parameters, whatever their case. `new Point(1, 2)` gives them their values in order, `new Point()` gives each its default, and `var (a, b) = p;` takes them apart in order:

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

`new Point()` needs a default for every parameter: a non-optional class or struct has none in NZ, because a non-optional type never holds `null` (section [1.7](01-basic-null-safety-io.md#17-null-safety)). So `new Ok<C>()` with a class `C`, and `new P()` for `record struct P(C Item)`, are errors that ask for the values (`new Ok<C>(Value)`); declare the parameter optional (`C? Item`) to allow `new P()`. A `string` parameter is the empty string in `new P()`. `record class P(int X);` (or `record P(int X);`) is a positional record class: the same public fields and `new P(1)`, but it is not taken apart with `var (a, b) = p;`. `with`, record equality and `ToString` are not supported yet. A program's own type named `Result` hides the built-in one, and `?` (below) works only on the built-in one.

### Passing errors on with `?`

The postfix operator `?` passes an error on. On a `Result<T, E>` it gives the `Ok` value, or makes the enclosing function return the `Err` at once; on a `T?` it gives the value, or makes the function return `null`.

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
- `?` may also be a statement on its own: `Check()?;` evaluates the call, leaves the function with the error exactly as `var x = Check()?;` would, and discards the success value (an owned value, such as a string, a list or a class instance, is released at the end of the statement). Only a `?` can be a statement this way: `Check();` keeps its `Result` as a discarded value and `x + 1;` or `x;` is still an error.

**Reading `?` next to the conditional operator.** After an expression, a `?` followed by a token that can start an expression is the conditional operator `c ? a : b`; otherwise it passes an error on. So `f()? - 1` is a conditional (and an error without its `:`); write `(f()?) - 1` to pass the error on and then subtract. The same holds for `+`, `-`, `*`, `&`, `!`, `~`, `(`, `[`, `^` and `..` after the `?`. `?.`, `?[` and `??` are operators of their own, so `f()?.x` is a null-conditional member access and `f()? ?? 0` passes the error on, then coalesces. In an interpolation hole with a format, put the operator in parentheses: `$"{(f()?):D2}"`.

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

## 6.3 Using `throw`

This section is not written yet.

Outline:

- `throw`
- `catch`
- Exception types

Previous: [5. Object-oriented programming](05-object-oriented-programming.md#5-object-oriented-programming) · [Contents](README.md) · Next: [7. Concurrency and parallel programming](07-concurrency-and-parallel-programming.md#7-concurrency-and-parallel-programming)
