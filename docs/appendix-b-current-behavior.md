# Appendix B. Current behavior notes

> NZ is still in development and is not stable. Anything in this documentation can change.

The C backend is young. These are facts about how the compiler behaves today, not settled design; any of them may change.

**`string.Length` counts UTF-8 bytes**, not characters:

```nz output="6\n"
package main;

import std.io.Console;

void Main(string[] args) {
    string s = "héllo";
    Console.WriteLine($"{s.Length}");
}
```

**`foreach` keeps its collection borrowed while the body runs**: the body can read the list it walks, but adding to it, assigning an element or the list, or moving it is an error. Collect the changes and apply them after the loop:

```nz error="collection is borrowed by the loop"
package main;

import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(1);
    foreach (int v in l) {
        if (l.Count < 4) {
            l.Add(v + 1);
        }
        Console.Write($"{v} ");
    }
    Console.WriteLine("");
}
```

**A list element cannot be passed by `ref`**: a `List<T>` element is read and written through the list's indexer, so it is not a variable. An array element is a variable and can be passed by `ref`. To let a function change a list element, copy it into a variable, pass the variable and store it back. The NZ code below swaps an array element with a list element that way:

```nz output="3 2 1\n"
package main;

import std.io.Console;
import std.collections.List;

void Swap(ref int a, ref int b) {
    int t = a;
    a = b;
    b = t;
}

void Main(string[] args) {
    int[] xs = new int[2];
    xs[0] = 1;
    xs[1] = 2;
    List<int> ys = new List<int>();
    ys.Add(3);
    int y = ys[0];
    Swap(xs[0], y);
    ys[0] = y;
    Console.WriteLine($"{xs[0]} {xs[1]} {ys[0]}");
}
```

```nz error="is an element of a list, read through its indexer, not a variable"
package main;

import std.collections.List;

void Bump(ref int x) {
    x++;
}

void Main(string[] args) {
    List<int> ys = new List<int>();
    ys.Add(3);
    Bump(ys[0]);
}
```

**A struct in a `List<T>`**: assigning a field, `l[0].X = v`, writes into the list. Calling a method, `l[0].M()`, depends on the struct. A struct whose fields are all copied freely (numbers, `bool`, `char`, `string`, enums, such structs) is copied, and the method runs on the copy, so what it changes is lost:

```nz output="5\n"
package main;

import std.io.Console;
import std.collections.List;

struct P {
    int X;
    void Bump() { X = X + 1; }
}

void Main(string[] args) {
    List<P> ps = new List<P>();
    ps.Add(new P());
    ps[0].X = 5;
    ps[0].Bump();
    Console.WriteLine($"{ps[0].X}");
}
```

A struct that owns something (a class object, a list, an array, or a struct that does) is move-only: it cannot be copied, so the method runs on the value stored in the list and its changes stay. The same holds for such a struct reached through `x?.M()`, a `foreach` variable or a pattern variable:

```nz output="6\n"
package main;

import std.io.Console;
import std.collections.List;

struct Q {
    int X;
    List<int> Log = new List<int>();
    void Bump() { X = X + 1; }
}

void Main(string[] args) {
    List<Q> qs = new List<Q>();
    qs.Add(new Q());
    qs[0].X = 5;
    qs[0].Bump();
    Console.WriteLine($"{qs[0].X}");
}
```

**Deep recursion in a compiled program crashes** with a stack overflow and no NZ message. When standard output is piped it is fully buffered, so output printed before the crash can be lost too.

**Nesting limits.** Input nested very deeply is refused with a message, never a crash. The parser stops at 1000 levels of parentheses, of prefix operators and casts, or of types (`expression nested too deeply`); the checker stops at 1200 levels of expression (`code is nested too deeply to check`). A third limit is on the C that the compiler writes: every call, parenthesis, cast and block adds to a budget of 2500 units, because clang itself crashes on very deep C. An expression can pass `nzc check` and still be refused by `nzc build` and `nzc run` with `expression nested too deeply to compile`; a cast of a parenthesised value adds the most, so a chain of a few hundred nested casts is over the budget. Split such an expression with local variables. A `defer` statement is written again at every exit of its scope (the end, each `return`, `break` and `continue`), and a `defer` inside a deferred statement multiplies that at every level; a function whose C passes 4 MiB once a deferred statement is about to be written again is refused by `nzc build` and `nzc run` with a message that says so. Move the cleanup into a function or reduce the exits.

**A temporary handle lives until the end of its statement.** A `Shared<T>` made inside a statement and not kept is released when the statement ends, not earlier, so a `WeakReference<T>` made from it can still reach the object within the same statement. Keep the result of `TryGetTarget()` to keep the object alive; after the statement, a weak handle to an object nobody holds gives `null`:

```nz output="True\nTrue\n"
package main;

import std.io.Console;
import std.memory.Shared;
import std.memory.WeakReference;

class Node { int N = 7; }

Shared<Node> Make() { return new Shared<Node>(new Node()); }

void Main(string[] args) {
    var t = new WeakReference<Node>(Make()).TryGetTarget();
    Console.WriteLine($"{t != null}");
    var w = new WeakReference<Node>(Make());
    Console.WriteLine($"{w.TryGetTarget() == null}");
}
```

**Dropping a deep structure needs only a bounded stack.** Owned and shared structures of any shape (a chain, a list of children, a field of a struct, objects of alternating types, a tree deep on either side) are dropped through a worklist in the runtime, not by recursion, so their depth is limited by memory. Fields drop in declaration order, so the owning fields declared after a chain's link (a field of the object's own type, the next handle of a `Shared<T>` or `Arc<T>` chain, or any other field that leads on) wait until the rest of the chain has dropped. A chain whose link is the last field that owns something leaves nothing waiting. Otherwise, while the worklist is short (a tree, a short chain), each waiting field (an object, collection, handle, string with heap text, or owning part of a struct, tuple or union) takes one worklist entry of 32 bytes. Once the worklist holds 1024 entries, an object's waiting fields are copied instead into one record, and the records of a chain of one type are linked together under a single entry. For a class object the record is a block as large as the object, allocated right after the object is freed, so the allocator can give back that same block and a long chain takes about the memory it held while it drops. For the struct payload of a `Shared<T>` or `Arc<T>` chain the record holds only the waiting fields, since the handle's block stays while a `WeakReference<T>` to it lives. A chain through objects of alternating types keeps one entry and one record per object instead of one entry per field. Records cost one allocation per object: in our measurements dropping such a chain was 13 to 17% slower than pushing entries, and a tree dropped while the worklist was already that long about 20% slower (each node of its left spines makes a record). This holds with `IDisposable` too: what a `Dispose` releases (a class object or a struct with a `Dispose` of its own, also inside what it releases) waits until that `Dispose` returns, so no `Dispose` runs inside another and a chain released link by link from `Dispose` needs a bounded stack whatever its shape (see [1.9](01-basic-null-safety-io.md#19-the-using-keyword)). Deep recursion in your own code still can overflow the stack (see above).

**Not yet** (each of these is refused or missing today, and none is a crash):

- `[Flags]`: an enum value that combines members prints as its number (`Read | Write` is `3`), not as `Read, Write`, and there is no `HasFlag`.
- The constant graph does not read enum members: `const int K = (int)E.A;` is refused with ``E.A is not a constant``. Enum values themselves may use constants.
- Members on enums: `E.A.ToString()` is an error. Print an enum with `{e}` in an interpolation.
- `sizeof(nint)` and `sizeof(nuint)`: the size depends on the target, so a constant cannot hold it. `sizeof` of the other eight integer types and of an enum works.
- Unions as text and with `==`: `$"{p}"` on a union is refused (`not supported yet: a union as text`), and so is `p == q`; format or compare the case it holds after `is` or `switch`. Properties and constants in a union body, arrays and interfaces as cases, and type patterns between classes are not supported yet. Patterns: list patterns (`[a, .., z]`), extended property patterns (`{ A.X: 0 }`), property patterns on built-in types and handles, a user `Deconstruct`, and a pattern variable used after the `if` that tests `is not` are not supported yet. A union is not the payload of a handle: `Shared<Pet>` and `Arc<Pet>` are refused (`` `Shared<Pet>` is not supported; wrap the value in a class ``); put the union in a field of a class and share that class.
- `switch` patterns above the `long` range: on a `ulong` value, a literal pattern above `long.MaxValue` (`9223372036854775808`) is refused, and so is a suffixed literal pattern (`18446744073709551615UL`). A named constant works: `const ulong Big = 18446744073709551615UL;` and then `u switch { Big => ..., _ => ... }`.

Previous: [Appendix A. Standard library reference](appendix-a-standard-library.md#appendix-a-standard-library-reference) · [Contents](README.md)
