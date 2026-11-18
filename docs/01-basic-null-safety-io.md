# 1. Basic: Null safety, input and output

Part of chapter [1. Basic](01-basic.md#1-basic).

## 1.7 Null safety

`null` is only valid for optional types `T?`.

```nz fragment error="null"
string name = null;
```

A `T?` value must be checked before it is used. The compiler recognises several forms of check:

```nz fragment
string? name = null;

var length = name?.Length;
var shown = name ?? "anonymous";

if (name != null) {
    int n = name.Length;
}

if (name is string s) {
    Console.WriteLine(s);
}
```

Using it without a check is rejected:

```nz fragment error="possibly null"
string? name = null;
int n = name.Length;
```

Operators on an optional number, `char`, `bool` or enum are **lifted**. An arithmetic, bitwise, shift or unary operator gives `null` when an operand is `null`, otherwise the plain operator's result as an optional (`int? + int` is an `int?`, `byte? + byte` an `int?`); the operation runs only when every operand has a value, with the plain operator's promotion and checked arithmetic. `<`, `>`, `<=` and `>=` give a `bool`: `false` when an operand is `null` (so `a <= b` is `false` when both are `null`, while `a == b` is `true`). On `bool?`, `&` and `|` follow a three-valued table: `false & null` is `false`, `true | null` is `true`, and the other combinations with `null` give `null`; `&&` and `||` take only `bool`. Compound assignment and `++`/`--` lift too (`a += 1` leaves a `null` `a` as it is). A `string?` is not lifted, but `+` with a string operand is concatenation: a null operand counts as `""`, so `s + "x"` on a null `s` is `"x"` (a `string`, never null), and `s += "x"` on a null `string?` stores `"x"`. An object is not lifted: check it first.

```nz fragment output="6 [] True False [] False\n"
int? a = 5;
int? none = null;
bool? unknown = null;
a += 1;
Console.WriteLine($"{a} [{a + none}] {a > 5} {none < 5} [{unknown & true}] {unknown & false}");
```

Two spelling rules involve `?`. In `x is int?` the `?` belongs to the type only when the next token cannot start an expression, so `x is int ? a : b` is a conditional, not a test against `int?`. With a space, `x is int? [1] : b` is a conditional whose arm is the collection expression `[1]`. And `?[` (no space) is one token: in a type, `string?[] names` and `new string?[3]` are arrays of optional strings; after a value, `a?[0]` is null-conditional indexing (below), so `x is int?[0]` indexes the `bool` `x is int`, which is an error.

```nz fragment output="yes\n"
int x = 1;
string r = x is int ? "yes" : "no";
Console.WriteLine(r);
```

`a?[i]` reads an element of an array, a `List<T>`, a `string` or a span that may be null: it is `null` when `a` is, and then the index is not evaluated; otherwise it is the element, as an optional (`int?`, `char?`, `string?`). An array reached through `?.` works the same way (`box?.Items?[0]`).

```nz fragment output="none\n2 3\n"
List<int>? l = null;
int? first = l?[0];
Console.WriteLine(first == null ? "none" : "some");
l = [2, 3];
Console.WriteLine($"{l?[0]} {l?[^1]}");
```

The full rules, including when a check stops being valid, are in [Null Safety](+/Null-Safety.md).

## 1.8 Input and output

This section is not written yet.

Outline:

- Writing to the console
- Reading from the console
- Reading and writing files
- Paths and directories

## 1.9 The `using` keyword

A class or struct that implements the library interface **`IDisposable`** (`void Dispose();`) has a cleanup of its own. NZ runs `Dispose` exactly once, whenever the value is dropped: at the end of its scope (also on `return`, `break`, `continue` and `?`), when it is replaced, when what holds it (an object, a list, a dictionary, a callee that took it) is dropped, and at the end of a `using` block or of the block of a `using var` declaration. `Dispose` runs first, while the fields are still there, and then the fields are dropped. A moved value is disposed by its new owner, never twice; an aborting program runs no `Dispose`. Cleanup follows ownership, and `using` only marks where a value's scope ends. A struct that implements `IDisposable` moves like a class instead of being copied.

```nz output="in the block\nclose a\nclose b\nend of block\ndeferred\nclose e\nend of Main\nclose d\nclose c\n"
package main;

import std.io.Console;
import std.collections.List;

class Log : IDisposable {
    string Name = "";
    Log(string name) { Name = name; }
    void Dispose() { Console.WriteLine($"close {Name}"); }
}

void Keep(ref List<Log> logs, Log log) {
    logs.Add(log);
}

void Main(string[] args) {
    using (var a = new Log("a")) {
        Console.WriteLine("in the block");
    }
    var b = new Log("b");
    b = new Log("c");
    var logs = new List<Log>();
    Keep(logs, new Log("d"));
    {
        using var e = new Log("e");
        defer Console.WriteLine("deferred");
        Console.WriteLine("end of block");
    }
    Console.WriteLine("end of Main");
}
```

`Dispose` runs by itself at the drop, so a program does not call it:

```nz error="calling `Dispose` is not supported yet: it runs by itself when the value is dropped"
package main;

class Log : IDisposable {
    void Dispose() { }
}

void Main(string[] args) {
    var l = new Log();
    l.Dispose();
}
```

`using` has these forms: `using (var r = e)`, `using (T r = e)` (the value converts to `T` as in a declaration), `using (e)` (the value has no name), several declarators `using (var a = e, b = f)` (dropped in reverse order, `b` first), and the declarations `using var r = e;` and `using T r = e;`, which last to the end of their block.

Inside a `try` block, a `using` value is dropped (with its `Dispose`) when the program leaves the block it belongs to, before the `finally` block runs, whichever way it leaves: the end of the block, `return`, `break`, `continue` or `?`, except inside a `Dispose`: there the value waits until the enclosing `Dispose` returns (decision 0038).

A value that is replaced or removed (`x.F = y;`, `= null`, a list or dictionary element, `Clear`, `Remove`) is taken out of its place before it is dropped, so a `Dispose` that reaches the same place again (through a `Shared` or weak back edge, a static) sees the new value. What a `Dispose` releases (a field set to `null` or replaced, a local leaving its scope, `Clear`) is never dropped inside it: a value that has a `Dispose` of its own, a class object or a struct, waits until that `Dispose` returns and the object that ran it has been dropped, then the waiting values are dropped in the order they were released, each completely before the next. This also holds for such a value reached inside what was released (a plain object, a struct field, a list, dictionary, array, optional, union or `Shared` payload). It is the same at every depth, so a chain released link by link (`Next = null;` in `Dispose`) needs a bounded stack however long it is and whatever its links go through. Anything else it releases is dropped at once: a `Shared` handle held directly or in a value without a `Dispose` (a list, dictionary, array, plain object, struct) counts down when it is released, so a `WeakReference` sees the object gone in the next statement; a handle inside a value that has a `Dispose` counts down when that value is dropped.

`using` also accepts a value that does not implement `IDisposable`; it then only drops it at the end of the block. Assigning to a `using` variable disposes the old value at once, and moving it out (`return r;`, passing it by value) hands the cleanup to the new owner.

Previous: [1.5 Operators](01-basic-operators-control-flow.md#15-operators) · [Contents](README.md) · Next: [2. Programming paradigms](02-programming-paradigms.md#2-programming-paradigms)
