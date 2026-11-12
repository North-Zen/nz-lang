# Ownership and Borrowing

This page describes what the stage 4 compiler enforces today. Every block below is checked by `cargo test`; blocks with an `output` are built and run.

## One owner

A `class` object, a `List<T>` and an array each have exactly one owner: a local variable, a parameter, a field, or a list element. Assigning one to another place, or passing it to a parameter with no modifier, **moves** it. The old owner cannot be used again until it is given a new value:

```nz error="moved"
package main;

import std.io.Console;

class Box {
    int N;
}

void Main(string[] args) {
    var a = new Box();
    var b = a;
    Console.WriteLine($"{a.N}");
}
```

The message points at the use and at the move (`a moved here`). A move in only one branch makes the variable "maybe moved" afterwards, and a move inside a loop body is an error unless the variable gets a new value before the next iteration.

Numbers, `bool`, `char`, enums and structs that own nothing are copied, not moved.

## Moving out of a field or an element is refused

An object keeps owning its fields and a list keeps owning its elements, so they cannot be moved out. The checker says so and points at the owner:

```nz error="cannot move out of field"
package main;

class Inner {
    int N;
}

class Outer {
    Inner I = new Inner();
}

void Main(string[] args) {
    Outer o = new Outer();
    Inner i = o.I;
}
```

Lend the field with `in`, copy the values you need, or take the object out by replacing it, as `Take` does in `examples/stack.nz` (a `ref` parameter may be moved out of when it is filled again before the function returns).

## Tuples

A tuple is copied when all its elements are copied (numbers, `bool`, `char`, `string`, enums, structs that own nothing, and tuples of those). A tuple that holds an owned element, such as `(int, C)` with a class `C`, is owned too: it **moves and is freed as a whole**. Building it moves the owned elements in, `var u = t;` moves it, and when its owner leaves scope every owned element is freed once.

An element cannot be moved out of an owned tuple, as with a field: the tuple keeps owning it. Lend it with `in`, read its numbers, or deconstruct the whole tuple with `var (a, b) = t;`, which moves the tuple and gives each name its element. A discarded owned element (`var (n, _) = t;`) is freed at the end of that statement. `t.Item1` and a named `t.x` are the same place, so they cannot be lent twice. Assigning an element, `t.Item2 = new C();`, moves the new value in and frees the old one. A function that captures a tuple (a local function) only reads it.

```nz error="cannot move out of tuple element"
package main;

class C {
    int N;
}

void Main(string[] args) {
    var t = (1, new C());
    C c = t.Item2;
}
```

```nz output="0 1\n"
package main;

import std.io.Console;
import std.diag.Diag;

class C {
    int N;
}

void Main(string[] args) {
    long start = Diag.LiveObjects();
    {
        var (n, _) = (1, new C());
        Console.Write($"{Diag.LiveObjects() - start} ");
        var t = (2, new C());
        Console.WriteLine($"{t.Item1 - n}");
    }
}
```

## Unions

A union (README section 2.9) is `Copy` when every case is `Copy`, counted when a case holds a string and none is owned (copied freely, the copy holding its own count on the active case's strings, as a struct of strings is), and owned otherwise. An owned union is moved as a whole, like a struct that owns something: assigning it or passing it without a modifier moves it, and dropping it drops only the case it holds (through the same worklist as every other drop).

A `switch` or an `is` borrows the union it tests, and a binding (`Cat c`) borrows the case (owner decision E4): it can be read and changed, and the change is seen through the union, but it cannot be moved out:

```nz output="1 3\n"
package main;

import std.io.Console;
import std.diag.Diag;
import std.collections.List;

class Cat { public List<int> Toys = new List<int>(); }
class Dog { }

union Pet(Cat, Dog);

void Main(string[] args) {
    Pet p = new Cat();
    if (p is Cat c) {
        c.Toys.Add(1);
    }
    Pet q = p;
    int n = q switch {
        Cat c => c.Toys.Count,
        Dog => 0,
    };
    Console.WriteLine($"{n} {Diag.LiveObjects()}");
}
```

(The three live objects are `args`, the `Cat` and its list.)

```nz error="a pattern variable borrows the value it tested"
package main;

class Cat { }
class Dog { }

union Pet(Cat, Dog);

Cat Take(Pet p) {
    return p switch {
        Cat c => c,
        _ => new Cat(),
    };
}

void Main(string[] args) {
    Cat c = Take(new Cat());
}
```

A value made on the spot that a `switch` or `is` tests lives until the end of its statement (see "Freed at scope exit"). A method of a union receives `this` borrowed, as a struct's method does; a method that changes a case through `this` cannot be called on an `in` parameter. Assigning a pattern variable that reads a struct case which owns something, in place in its union, is not supported yet.

An owned payload leaves a `Result` (README section 1.7) only through a borrow today: the `switch` binding reads `ok.Value` but cannot take it. The `?` operator (batch B4d) will move the payload out of a `Result` the function owns.

## Borrowing: `in` and `ref`

A parameter can borrow instead of owning. `in` lends the value for reading, `ref` lends it for reading and writing. The caller keeps ownership, and the borrow lasts **only for that one call**: there are no local borrows, so a borrow cannot be stored in a variable, a field or a return value.

```nz output="0\n1\n"
package main;

import std.io.Console;

class Counter {
    int Count;
}

void Show(in Counter c) {
    Console.WriteLine($"{c.Count}");
}

void Bump(ref Counter c) {
    c.Count += 1;
}

void Main(string[] args) {
    var c = new Counter();
    Show(c);
    Bump(c);
    Show(c);
}
```

Writing through an `in` parameter is an error. As in C#, an `in` parameter is a read-only **reference**, also for a value type such as `int` or a struct: the callee reads the caller's storage, not a copy, so a change made to that storage during the call by other means is seen through the parameter (except for `in string` and `in string?`, which hold a copy of their own; see [Strings](#strings)). In tier-1 code the borrow rules leave only one such means, an `unsafe` callee writing a static it also received by `in`: with `static int counter = 1;` and `unsafe void Bump(in int n) { counter = 2; Console.WriteLine($"{n} {counter}"); }`, `unsafe { Bump(counter); }` prints `2 2`. A [shared object](#shared-objects) can be changed through another handle:

```nz output="2\n"
package main;

import std.io.Console;
import std.memory.Shared;

class Box {
    int N = 1;
}

void Show(in int n, Shared<Box> other) {
    other.N = 2;
    Console.WriteLine($"{n}");
}

void Main(string[] args) {
    var b = new Shared<Box>(new Box());
    Show(b.N, b);
}
```

An optional value-type `in` parameter (`in int? n`) is still passed as a copy, and an `in string` holds a copy of its own (see [Strings](#strings)).

Because borrows live for one call and cannot be kept, a borrow can never outlive its owner, and no lifetime annotations exist.

## One writer or many readers

Within one call, an argument passed by `ref`, a moved argument, or a receiver that a method changes cannot share storage with any other argument or receiver of that call. Several `in` borrows of the same value are fine. Passing the same variable twice, once by `ref`, is rejected:

```nz error="overlapping"
package main;

class Box {
    int N;
}

void Two(ref Box a, in Box b) {
    a.N = b.N;
}

void Main(string[] args) {
    Box x = new Box();
    Two(x, x);
}
```

The same rule covers an array element passed by `ref`: while the call runs, the array itself or a second element of it cannot be passed to the same call. A `List<T>` element is not a variable (it is read through the list's indexer) and cannot be passed by `ref` at all, as in C#.

## `foreach` freezes its collection

The collection a `foreach` walks is borrowed by the loop. Its body can read it, but adding to it, assigning to an element, replacing it or moving it is an error. Collect the changes and apply them after the loop:

```nz error="collection is borrowed by the loop"
package main;

import std.collections.List;

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(1);
    foreach (int v in l) {
        l.Add(v);
    }
}
```

The loop variable borrows an element: it can be read and passed with `in`, but not assigned, passed by `ref` or moved.

A `foreach` over a `Shared<T>` or `Arc<T>` handle walks the list or array the handle holds, under the same rule; see [`foreach` through a handle](#foreach-through-a-handle).

## Local functions borrow what they capture

A local function (guide section 1.5) reads variables of the function around it. That read is a borrow like `in`: it is read-only, and it lasts for each call of the local function. The local function never owns what it reads, so it never frees it; the owner stays the enclosing function.

Inside the local function a captured variable follows the rules of an `in` parameter: it cannot be assigned, passed by `ref`, moved, or have a field or element written, and a method that changes it cannot be called on it. The error names the local function:

```nz error="cannot move out of `c`"
package main;

class Counter {
    int Count;
}

void Take(Counter c) { }

void Main(string[] args) {
    Counter c = new Counter();
    void f() {
        Take(c);
    }
    f();
}
```

A call of a local function is a read of every variable it captures (also those that local functions it calls capture), evaluated after the arguments. So a variable that was moved earlier is a use after move at the call, with a label at the move and one at the use inside the function:

```nz error="use of moved value"
package main;

class Counter {
    int Count;
}

void Take(Counter c) { }

void Main(string[] args) {
    Counter c = new Counter();
    int f() {
        return c.Count;
    }
    Take(c);
    int r = f();
}
```

Each call also lends every captured variable for the call, like an extra `in` argument. An argument passed by `ref`, or a moved argument, cannot share storage with a variable the function captures (the call has no `ref` keyword: `f(c)` passes `c` by `ref` when `f` declares a `ref` parameter):

```nz error="overlapping borrow"
package main;

class Counter {
    int Count;
}

void Main(string[] args) {
    Counter c = new Counter();
    int f(ref Counter d) {
        return d.Count + c.Count;
    }
    int r = f(c);
}
```

An `in` argument is fine:

```nz output="0\n"
package main;

import std.io.Console;

class Counter {
    int Count;
}

void Main(string[] args) {
    Counter c = new Counter();
    int f(in Counter d) {
        return d.Count + c.Count;
    }
    int r = f(c);
    Console.WriteLine($"{r}");
}
```

In a method, `this` is captured the same way when the local function uses a field, a property or a method of the object: the function cannot write a field and cannot call a method that changes the object (`change the object in the method itself`).

Because the borrow ends with the call, a captured object is freed once, by its owner, when the owner leaves its scope. Calling the local function does not change when:

```nz output="1\n0\n"
package main;

import std.io.Console;
import std.diag.Diag;

class Counter {
    int Count;
}

void Main(string[] args) {
    long start = Diag.LiveObjects();
    {
        Counter c = new Counter();
        int f() {
            return c.Count;
        }
        int r = f();
        Console.WriteLine($"{Diag.LiveObjects() - start}");
    }
    Console.WriteLine($"{Diag.LiveObjects() - start}");
}
```

Returning a captured owned value (a class object, a list, an array, or a variable of a generic type `T`) is a move out of the captured variable and is refused; give the value to the function as a parameter instead.

## Freed at scope exit

An owned object is freed deterministically: when its owner leaves scope, when the owner is replaced by an assignment, when it is moved into a `defer`red call that consumes it, or when the object that owns it is freed. `Diag.LiveObjects()` counts live objects, which is how this program shows the drops (the counts are relative to the start, because `args` is an object too).

All the `defer`s and drops of one scope run in **one reverse order of registration**. Here `b` is declared after the `defer`, so it is freed before the `defer` runs, and `a`, declared before it, is still alive:

```nz output="before the block ends: 3\nin the defer: 2\nafter the block: 1\nafter replacing: 1\n"
package main;

import std.io.Console;
import std.diag.Diag;

class Box {
    int N;
}

long Live(long start) {
    return Diag.LiveObjects() - start;
}

void Main(string[] args) {
    long start = Diag.LiveObjects();
    Box keep = new Box();
    {
        Box a = new Box();
        defer Console.WriteLine($"in the defer: {Live(start)}");
        Box b = new Box();
        b.N = 1;
        Console.WriteLine($"before the block ends: {Live(start)}");
    }
    Console.WriteLine($"after the block: {Live(start)}");
    keep = new Box();
    Console.WriteLine($"after replacing: {Live(start)}");
}
```

Assigning a new object to `keep` frees the old one immediately. `return`, `break` and `continue` drop every live local they leave, in the same order. `using (...)` makes its value visible in the block and frees it when the block ends; it has no other cleanup action.

A value made on the spot that a pattern variable tests (`if (Make() is Node n) { ... }`) lives until the end of the statement that holds the pattern: for an `if`, the whole `if`, both branches included; for a loop condition, the loop (each new value replaces the one before); for a declaration or an expression statement, that statement. Every way out of the statement (its end, `return`, `break`, `continue`) frees it once. The same holds for the value a `switch` tests.

Dropping a container drops what it owns: a list frees its elements, an object frees its fields. A structure of any shape and depth is dropped with a bounded C stack: the generated drop code does not recurse, it defers what it reaches to a worklist in the runtime and works through it before the drop finishes, so a chain through a field of the class's own type, a list of children, a field of a struct, objects of alternating types or a tree deep on either side can be as long as memory allows (chains of 1,000,000 links of each shape are tested).

### The order inside one drop

Dropping a value frees the **object first, then its fields in declaration order**, each field completely (with everything it owns) before the next field. The order is the same for every shape:

- a class object is freed, then its fields drop from the first declared to the last; a field holding another object drops that object the same way before the next field;
- a struct, tuple or positional `record struct` is not freed itself (it lives in its variable, field or element); its fields drop in declaration order in its place;
- a union drops the case it holds, in its place;
- an optional drops its value when it has one;
- a `List<T>`, `Dictionary<K, V>` or `HashSet<T>` is freed first, then its elements drop in order (a dictionary's entries in insertion order, the key before the value), then the storage that held them;
- an array's elements drop in order, then its storage: the elements live in the array's own block, so it goes after the last one;
- a `Shared<T>` or `Arc<T>` whose last strong handle goes releases its block first (the block is freed unless a `WeakReference<T>` to it is left), then drops the object it holds;
- a string field is released in its place among the other fields.

```nz
package main;

import std.io.Console;

class Leaf {
    int N;
}

class Node {
    Leaf First = new Leaf();
    Node? Next;
    Leaf Last = new Leaf();
}

void Main(string[] args) {
    var n = new Node();
    n.Next = new Node();
    Console.WriteLine("built");
    // Freed in this order: n, n.First, n.Next, n.Next.First, n.Next.Last, n.Last.
}
```

Nothing a program can do today observes the order inside one drop: there are no destructors yet, and `Diag.LiveObjects()` and the `NZ_ALLOC_REPORT=1` report are read between statements. The order is fixed now because it becomes the order in which destructors run once they exist. Real C# has no deterministic destructor order for managed objects (a finalizer runs when the garbage collector decides, `IDisposable.Dispose` when the program calls it), so this rule is NZ's own (owner decision, 2026-11-12); it is the parent-before-members order of Rust and C++. Each field's value is read before its object is freed, so no drop reads freed memory, and nothing is freed while a borrow of it lives (the borrow rules above). The order has a memory cost in long chains: the fields declared after a chain's link drop only once the rest of the chain has, so while the drop runs the runtime's worklist grows by one entry (32 bytes) per owning field declared after the link, per node (see README, "Dropping a deep structure needs only a bounded stack").

## Strings

A `string` is an immutable value that is copied freely, as in C#: assigning it, passing it by value, returning it, and storing it in a field, an element or a tuple make a copy, and every copy stays usable. A struct or tuple whose only members that hold something are strings is copied freely too. Nothing about strings is ever refused by the borrow rules.

Underneath, a computed string (made by `+`, an interpolation or a library call) is one block: a reference count followed by the bytes. Every holder (a variable, a field, an element, a by-value parameter, a temporary of a statement) keeps one count, and the last one to go frees the block, at the same points where owned values are dropped. A string literal is static and is never counted or freed. `a + b + c` makes one string, with one allocation. `NZ_ALLOC_REPORT=1` prints the computed strings still alive at exit on a second line, `nz: live strings = M`, and `Diag.LiveStrings()` returns that count while the program runs. Strings reached from an `Arc<T>` payload are counted with atomic operations, so they may be released on any thread. (They are marked when the `Arc` is made, which is enough because an `Arc` payload cannot change; when a later batch adds `Mutex<T>`, every string stored into a place reachable from another thread will have to be marked at the store.)

An `in string` (or `in string?`) parameter holds a counted copy of the argument, made when the call starts. **This differs from C#**, where `in` is a read-only reference: a callee that replaces the original through another path (a shared object, a `ref` of the same variable, a static) still sees the value the call started with. The same program in C# prints `new` twice.

```nz output="old\nnew\n"
package main;

import std.io.Console;
import std.memory.Shared;

class Box {
    string Label = "old";
}

void Show(in string s, Shared<Box> other) {
    other.Label = "new";
    Console.WriteLine(s);
    Console.WriteLine(other.Label);
}

void Main(string[] args) {
    var b = new Shared<Box>(new Box());
    Show(b.Label, b);
}
```

## Generics

A generic function body is checked **once**, with `T` treated as an owned type, whatever the callers use. So the usual rules apply to values of type `T` inside it, and `Swap<T>` works for numbers and for objects alike:

```nz output="2 1 20 10\n"
package main;

import std.io.Console;

class Box {
    int N;
}

void Swap<T>(ref T a, ref T b) {
    T t = a;
    a = b;
    b = t;
}

void Main(string[] args) {
    int x = 1;
    int y = 2;
    Swap(x, y);
    Box p = new Box();
    p.N = 10;
    Box q = new Box();
    q.N = 20;
    Swap(p, q);
    Console.WriteLine($"{x} {y} {p.N} {q.N}");
}
```

The consequence: a generic function that takes an element out of a `List<T>` is refused, even if only `int` ever uses it, because for an owned `T` that would move out of the list:

```nz error="cannot move out of a collection element"
package main;

import std.collections.List;

T First<T>(ref List<T> xs) {
    return xs[0];
}

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(1);
    int n = First(l);
}
```

## Statics

A static is a place with one storage for the whole program, like a variable that never goes out of scope: it can be passed by `ref`, and two `ref` arguments naming it overlap. Rules are not suspended inside `unsafe`: `unsafe` allows the use of mutable statics and the pointer features of [Pointers and `fixed`](#pointers-and-fixed), nothing more.

An **owned static** (guide section 1.9) owns its value for the whole program. Nothing moves out of it, as out of a field; assigning drops the old value once the new one is computed; the program drops it at normal exit in reverse declaration order.

Any call may replace a static, and the analysis sees one function at a time. So while a `foreach` iterates over something an owned static holds, while a pattern variable borrows from one, while an argument or receiver already holds part of one, or while an assignment or index has fixed an object inside one, a call that may run NZ code is refused or ends the borrow. A `ref` to the whole static is the exception: it is an address, read when the callee runs.

**Calling an `unsafe` function needs `unsafe`** (owner decision D-STATIC-UNSAFE, answer (b)): a call of an `unsafe` function or method, or of any method of an `unsafe` type, and `new T()` when a field initializer of an `unsafe` type is not a constant expression (a call, a `new`, a write or a read of a static), are written inside `unsafe`. The rule is lexical: it is checked where the call is written and does not spread to callers.

**The `unsafe` block is the vouch point.** The callee of the call that lends the static is not checked across functions. When that callee assigns the static from `unsafe` code, whether it is an `unsafe` function or method receiving the static by `in`, `ref` or as its receiver (or a field inside it by `ref`), or a safe function that does the write inside an `unsafe { }` block, the caller reads freed memory. A safe function containing such a block is as dangerous to the caller that lends it the static as an `unsafe` function is. As in Rust, the vouch sits with the `unsafe` block wherever it appears: the programmer who writes it vouches that nothing lent is freed by it, as with `static mut`.

## Shared objects

Ownership with one owner covers most data. When a value really has several owners, such as a node reached from two parents, `std.memory` gives three handle types (tier 2 of the [Memory Model](Memory-Model.md)):

- `Shared<T>` shares one object on one thread; its counts are not atomic.
- `Arc<T>` shares a read-only object; its counts are atomic.
- `WeakReference<T>` reaches a `Shared<T>` object without keeping it alive.

`new Shared<T>(v)` moves `v` into a new block. Copying a handle does not copy the object: it shares it and counts one more holder, and every holder releases it once, in the same places an owned value is dropped. The last strong handle drops the object. The payload is a class, a struct, a tuple, a list or an array; a handle of a number or another scalar is refused, also where a generic call makes `T` a scalar:

```nz error="`Shared<int>` is not supported; wrap the value in a class"
package main;

import std.memory.Shared;

void Main(string[] args) {
    var n = new Shared<int>(1);
}
```

The object's members, indexer and methods are reached through the handle. A `Shared<T>` object can change through any handle: an `in` parameter holding the handle keeps the handle from being replaced, not the object from changing. `Shared.StrongCount(s)` and `Shared.WeakCount(s)` read the counts. `w.TryGetTarget()` gives a `Shared<T>?` that holds the object, or null once the object is gone:

```nz output="2 2\n2\n-1\n"
package main;

import std.io.Console;
import std.memory.Shared;
import std.memory.WeakReference;

class Counter {
    int N;
}

void Bump(in Shared<Counter> c) {
    c.N += 1;
}

void Main(string[] args) {
    var a = new Shared<Counter>(new Counter());
    var b = a;
    Bump(a);
    Bump(b);
    Console.WriteLine($"{a.N} {Shared.StrongCount(a)}");
    var w = new WeakReference<Counter>(a);
    b = new Shared<Counter>(new Counter());
    Console.WriteLine($"{w.TryGetTarget()?.N ?? -1}");
    a = b;
    Console.WriteLine($"{w.TryGetTarget()?.N ?? -1}");
}
```

`==` on two handles compares identity, not content. A temporary handle, such as the one `TryGetTarget()` returns above, lives until the end of its statement.

### Lends are checked when the program runs

Any holder can change a shared object, so the compiler cannot see every borrow of it. Borrows of the parts of a shared object that own something (a list field, an object field) take a **lend** that the program checks when it runs, always. Replacing or changing such a part while it is lent stops the program with `nz: shared object changed while it is lent`. Lending a part for writing (`ref`, or a method that changes it) while the object is lent in any other way stops it with `nz: shared object lent for writing while a part of it is lent for reading`; this is also what `RefRef(s.Items, t.Items)` prints when `s` and `t` reach the same object, because every lend is first taken for reading where its argument is evaluated. For example, iterating over `node.Children` with `foreach` while the loop body adds to the same list, through another handle to the same node, stops the program instead of reading freed memory. Lends are counted per object, so a `ref` of one part and an `in` of another part of the same object also stop the program. Parts that own nothing, such as an `int` field, take no lend.

### `foreach` through a handle

A `foreach` over a `Shared<T>` or an `Arc<T>` whose payload is a list or an array walks the payload, as iterating a reference to a `List<int>` does in C#. The loop variable borrows each element, as over a plain list, and the handle the loop walks is frozen like any collection: adding to the list through it in the body, assigning one of its elements or replacing the handle is an error.

```nz output="6 3\n"
package main;

import std.io.Console;
import std.collections.List;
import std.memory.Shared;

void Main(string[] args) {
    var l = new List<int>();
    l.Add(1);
    l.Add(2);
    l.Add(3);
    var a = new Shared<List<int>>(l);
    var b = a;
    int sum = 0;
    foreach (var x in a) {
        sum += x;
    }
    Console.WriteLine($"{sum} {b.Count}");
}
```

Another handle to the same object is out of the compiler's sight, so the loop lends a `Shared` object for reading until it is left by any path (the end, `break`, `return`, a labeled jump). Adding to the list or replacing it through another handle, or lending it for writing, stops the program with the messages above instead of reading freed memory. The loop variable can change its element, as in C#: replace a part of it that owns something (`c.Items = new List<int>()`), add to such a part, or call a method that changes it. Such a change is checked when the program runs as well, without counting the loop's own lend: it runs while nothing else borrows the object, and stops the program with `nz: shared object changed while it is lent` while another loop or call, through another handle, may be using a part of it. Lends are counted per object, so a second loop over the same object, open around the change, stops it too. A method that changes the element, called through the loop variable, holds the object for writing while it runs, so if its body lends the object again (a loop over it through another handle, say) the program stops with `nz: shared object lent for reading while a part of it is lent for writing`. The same holds for a loop over a member (`foreach (var c in s.Items)`), for a loop over a part of the loop variable, and for a pattern variable bound under it. An element that owns nothing, such as an `int`, can be written through another handle; the loop sees the new value.

```nz output="2 3\n"
package main;

import std.io.Console;
import std.collections.List;
import std.memory.Shared;

class Group {
    List<int> Items = new List<int>();
}

void Main(string[] args) {
    var l = new List<Group>();
    l.Add(new Group());
    l.Add(new Group());
    var groups = new Shared<List<Group>>(l);
    foreach (var g in groups) {
        g.Items = new List<int>();
        g.Items.Add(3);
    }
    Console.WriteLine($"{groups.Count} {groups[1].Items[0]}");
}
```

An `Arc<T>` object is read-only, so the loop variable cannot change the objects in its list either:

```nz error="cannot change an object through `Arc<List<Item>>`"
package main;

import std.collections.List;
import std.memory.Arc;

class Item {
    int N;
}

void Main(string[] args) {
    var l = new List<Item>();
    l.Add(new Item());
    var items = new Arc<List<Item>>(l);
    foreach (var it in items) {
        it.N = 1;
    }
}
```

A `WeakReference<T>` is not iterated: upgrade it with `TryGetTarget()`, check the result against `null` and iterate that.

### `Arc<T>` is read-only

An `Arc<T>` object is shared and never changed. `Arc.StrongCount(a)` reads its count:

```nz output="api:8080 2\n"
package main;

import std.io.Console;
import std.memory.Arc;

class Config {
    string Name = "";
    int Port;
}

void Main(string[] args) {
    Config c = new Config();
    c.Name = "api";
    c.Port = 8080;
    var cfg = new Arc<Config>(c);
    var copy = cfg;
    Console.WriteLine($"{copy.Name}:{copy.Port} {Arc.StrongCount(cfg)}");
}
```

Writing through it is refused:

```nz error="cannot change an object through `Arc<Config>`"
package main;

import std.memory.Arc;

class Config {
    int Port;
}

void Main(string[] args) {
    var cfg = new Arc<Config>(new Config());
    cfg.Port = 1;
}
```

The same holds for every other way to name a part of the object: a loop variable or a pattern variable (`if (cfg.Inner is Inner i)`) that borrows from it, and a part chosen with `?:`, `??` or `switch` (`(c ? cfg.Inner : mine).Port = 1`).

An `Arc<T>` cannot hold, at any depth, a `Shared<T>` or a `WeakReference<T>`, whose counts are not atomic:

```nz error="cannot hold `Shared<Node>`, whose count is not atomic"
package main;

import std.memory.Arc;
import std.memory.Shared;

class Node {
    int N;
}

class Holder {
    Shared<Node>? Inner;
}

void Main(string[] args) {
    var h = new Arc<Holder>(new Holder());
}
```

A `WeakReference` to an `Arc<T>` is not supported yet.

### Cycles leak; break them with a weak reference

Two shared objects that hold strong handles to each other keep each other alive, so neither is ever freed. Such a leak is not silent: the objects stay in `Diag.LiveObjects()` and `nzc test` reports them. The fix is the usual one: make the back edge, such as a child's link to its parent, a `WeakReference<T>`. `examples/graph.nz` builds a small graph this way and ends with no live objects.

Dropping shared objects needs a bounded C stack whatever their shape, as for owned classes: a chain through a field of the payload's own type (`Shared<Node>? Next` in `Node`), a list of handles on each step, alternating types or `Arc<T>` links can be as long as memory allows.

## Constructors and object initializers

`new T(args)` passes each argument to the chosen [constructor](../README.md#constructors) by the mode of the parameter it binds to: an argument without a modifier is **moved** into the constructor, an `in` or `ref` argument is lent for the call. A parameter the constructor stores in a field belongs to the new object from then on; one it does not store is dropped when the constructor returns, as in any method (also on an early `return`). The same variable passed twice by value is a use after a move:

```nz error="use of moved value"
package main;

class Node {
    int V;
}

class Pair {
    Node A;
    Node B;

    Pair(Node a, Node b) {
        A = a;
        B = b;
    }
}

void Main(string[] args) {
    var n = new Node();
    var p = new Pair(n, n);
}
```

The object exists before its constructor runs, with every field zeroed. A field whose type has no default (a `string`, a class, a list, a union) is never read while it is zeroed: in a constructor that does not chain, every path assigns it before it is read, before `this` is used (a method call, `this` as a value), and before a local function or a `defer` that reads it can run; the arguments of `: this(...)` do not use the object at all; a `required` member is not read in a constructor, since the object initializer sets it afterwards. A field whose type has a default (a number, `bool`, `char`, an optional) may be read while zeroed: it reads as its default. A primary constructor's parameters are locals of the field initializers: moving one into two fields is a use after a move.

The values of an object initializer move into the new object's fields, in the order written; the value a field held before (its initial value, or what the constructor stored) is dropped once the new value is computed. Below, the first `Node` is freed: what stays alive is the crate and its new node (the program's `args` array is one more live object):

```nz output="3\n"
package main;

import std.io.Console;
import std.diag.Diag;

class Node {
    int V = 1;
}

class Crate {
    Node N = new Node();
}

void Main(string[] args) {
    var c = new Crate { N = new Node() };
    Console.WriteLine($"{Diag.LiveObjects()}");
}
```

## Named arguments and extension receivers

The rules below apply to NZ code (C# style) as written in section 1.12 of the README.

Arguments are moved and borrowed **in the order they are written**, whatever parameters they bind to by name; each one is moved or lent with the mode of the parameter it binds to, and a parameter left to its default takes nothing. So `Pair(second: a, first: a)` moves `a` twice and is refused at the second, as written.

The receiver of an [extension method](../README.md#112-parameters-and-arguments) is its first parameter. It is lent by that parameter's mode, `this in` for reading and `this ref` for writing, and never moved (a plain `this` is only allowed on copied types, where it is a copy). A `this ref` receiver counts as a receiver the call changes, so it cannot share storage with an argument of the same call, and it cannot change a collection a `foreach` iterates:

```nz error="overlapping borrow"
package main;

import std.collections.List;

void Put(this ref List<int> xs, in List<int> ys) { }

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Put(l);
}
```

Through a `Shared<T>` handle, the extension's receiver is the shared object, lent for the call with the run-time lend of [shared objects](#shared-objects): with `this in` exactly as for the plain call `Get(s)`; with `this ref` the extension writes the shared object under a write lend, so no other lend of it may be open during the call (through an `Arc<T>`, `this ref` is refused: its object never changes). A plain function with a `ref C` parameter does not take a `Shared<C>` at all.

## Pointers and `fixed`

Inside `unsafe` a program has pointers (guide section 1.13). This is what `unsafe` turns **on**, and nothing else: pointer types and operators, `sizeof` of unmanaged types, `stackalloc`, `fixed`, `NativeMemory`, calls to `extern` functions and to functions with pointers in their signature, and mutable statics. What **stays checked** inside `unsafe`: types, definite assignment, null safety, integer overflow and checked casts, array and list indexes, and the ownership and borrowing of managed values (classes, strings, arrays, lists, owned structs, tuples). Only pointer arithmetic and `p[i]` are unchecked.

Why managed values stay sound:

- **A pointer never points at a managed value.** Its pointee must be unmanaged, so `C*`, `string*` and `List<int>*` do not exist. Drop glue never runs through a pointer, and a pointer never owns.
- **What an owner can free needs `fixed`.** `&` of a class field, an array element, or a `ref` or `in` parameter is refused; `fixed` takes it and **freezes the root** for the block: the root cannot be moved, assigned, dropped, passed by `ref`, or changed by a method that may replace the fixed storage, while its fields can be read and written. The pointer is declared by `fixed`, cannot be changed, and dies at the block's end; temporaries a `fixed` holds are freed on every way out of it (`return`, `break`, `continue`, labeled jumps). Limit: a second `Shared<T>` or `Arc<T>` handle to the same object is not frozen.
- **A pointer cannot leave `unsafe`.** A pointer type can be written, and a pointer value can exist, only in an unsafe context, so a pointer cannot flow into safe code by assignment, return, field, capture, type argument, tuple or optional. The only way out is a cast to `nint` or `nuint`; turning that number back into a pointer needs `unsafe` again.
- **Pointers are copied and never moved.** For the checker a pointer is a `Copy` value with no drop glue; no loan is recorded for `&x`, whose target is unmanaged. A struct with pointer fields is `Copy` when its other fields are; a class with pointer fields stays an owner, and its drop frees the object, never what the pointers point at.

What `unsafe` vouches for, as in C# and Rust: a pointer to a local that has gone out of scope (`return &x;` from an `unsafe` function is accepted, as in C#), a write past the end of a buffer, a misaligned read through a pointer cast, a write through a pointer into an object lent by `in`, a double `Free`. The compiler does not see these; the sanitizers find them in tests. A pointer write can also corrupt memory next to its target, including a managed object's fields: that is undefined behaviour, which the program that writes `unsafe` takes on.

A static cannot be a `fixed` root yet, because a static starts with a constant and owns no array or object to pin. When statics with computed initializers arrive, `fixed` over one must follow the rule of [Statics](#statics): any call may replace a static, so a call that may run NZ code ends the freeze.

## Not supported yet

A few forms that would need more of the checker are refused with a message that begins `not supported yet:` and says how to rewrite them, for example calling a method through `?.` on a struct that owns something and is not stored in a variable. They are errors, never silent wrong code.
