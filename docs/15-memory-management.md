# 15. Memory management

> NZ is still in development and is not stable. Anything in this documentation can change.

The guides [Ownership](+/Ownership.md) and [Memory Model](+/Memory-Model.md) are the full reference for this chapter.

## 15.1 No garbage collector by default

NZ uses two memory tiers layered on top of one shared core, not global modes:

| Tier | Form | Used for |
|---|---|---|
| 1. No GC | ownership and borrowing | kernels, game loops, hot paths |
| 2. ARC | `Arc<T>`, `Shared<T>`, `WeakReference<T>` | shared structures, graphs |

The explanation and the design reasons are in [Memory Model](+/Memory-Model.md).

## 15.2 How memory is managed: stack, heap, deterministic drop, drop order

The rules are in [Ownership: Freed at scope exit](+/Ownership.md#freed-at-scope-exit) and [The order inside one drop](+/Ownership.md#the-order-inside-one-drop).

- **Freed at scope exit.** Owned objects are freed when the owner leaves scope, is replaced or is moved into a consuming call; `defer`s and drops of a scope run in one reverse registration order. Inside one drop the object is freed first, then its fields in declaration order, each completely before the next (see [Ownership: The order inside one drop](+/Ownership.md#the-order-inside-one-drop)). Strings are reference-counted: every holder keeps a count and the last one frees the bytes (see [Ownership: Strings](+/Ownership.md#strings)).

Outline:

- The stack and the heap
- Deterministic drop
- Drop order

## 15.3 Ownership and moves

The core tier of the memory model is checked by the compiler. The full description, with every rule and error, is in [Ownership](+/Ownership.md). In short:

- **One owner.** A class object, list or array has one owner. Assigning it or passing it without a modifier **moves** it, and using the old variable is an error:

```nz error="moved"
package main;

class Box {
    int N;
}

void Take(Box b) { }

void Main(string[] args) {
    var b = new Box();
    Take(b);
    Take(b);
}
```

- **Moving out of a field or a list element is an error**: the object or list keeps owning it. `var t = l[i];` of a class element is a read-only alias of the element that ends when the collection may change (see Ownership, "Reading an element through an alias").

- **A union moves as a whole.** A union whose cases are all `Copy` is `Copy`, one with a string case (and no owned one) is counted like a struct of strings, and any other is owned; dropping it drops the case it holds. A `switch` or `is` borrows the union it tests, and a binding (`Cat c`) borrows the case, so a case cannot be moved out of a union.
- **Generics are checked once**, with `T` as an owned type. A generic function that takes an element out of `List<T>` is refused even if only `int` uses it.
- Forms the checker cannot prove safe are refused with `not supported yet:` and a way to rewrite them.

## 15.4 Borrowing (`in`, `ref`, loans, aliases)

The rules are in [Ownership: Borrowing](+/Ownership.md#borrowing-in-and-ref) and [One writer or many readers](+/Ownership.md#one-writer-or-many-readers).

- **Borrowing lasts one call.** `in` lends for reading, `ref` lends for writing; there are no local borrows. An `in` parameter is a read-only reference, also for value types: a change to the caller's storage during the call (by an `unsafe` callee writing a static, or through another handle to a shared object) is seen through it, except through an `in string`, which keeps the value the call started with (see [Ownership: Strings](+/Ownership.md#strings)). One `ref` (or moved) argument cannot share storage with another argument of the same call, and a `foreach` freezes the collection it walks.
- **Local functions borrow what they read.** A local function reads the variables around it as a read-only borrow that lasts for each call; it cannot move, assign or lend them by `ref`, and each call counts as a read of every variable it reads ([Ownership](+/Ownership.md)).

Outline:

- `in` and `ref`
- Loans
- Aliases

## 15.5 Reference counting (`Shared<T>`, `Arc<T>`, `WeakReference<T>`)

The full rules are in [Ownership: Shared objects](+/Ownership.md#shared-objects).

Tier 2 lives in `std.memory`:
- `Shared<T>` shares an object on one thread;
- `Arc<T>` shares a read-only object, with atomic counts;
- `WeakReference<T>` reaches a `Shared<T>` object without keeping it alive.

`new Shared<T>(v)` moves `v` in, and copying a handle shares the object and counts it. The object changes through any handle, whose members, indexer and `foreach` reach the payload. Borrows of its owned parts are checked when the program runs, and a change while one is lent stops the program. `w.TryGetTarget()` gives a `Shared<T>?`, and `Shared.StrongCount(s)` reads the count. Cycles of strong handles leak visibly (live count, `nzc test`); a `WeakReference<T>` back edge breaks them. The rules, with examples, are in [Ownership: Shared objects](+/Ownership.md#shared-objects), and `examples/graph.nz` is a whole program.

## 15.6 Unsafe memory

Pointer types are in [1.1.4 Pointer](01-basic.md#114-pointer), mutable statics in [5.16 Static](05-object-oriented-programming.md#516-static), and calling C in [23.1 C interop](23-serialization-and-interoperability.md#231-c-interop).

Inside `unsafe`, NZ has systems features: pointers, `&`, `fixed`, `stackalloc`, manual memory, C layouts, calls into C and functions C can call. Everything else stays checked inside `unsafe`: types, definite assignment, null safety, integer overflow, array and list indexes, and the ownership of managed values (classes, strings, arrays, lists, owned structs). What `unsafe` turns on is listed in [Ownership](+/Ownership.md#pointers-and-fixed), with why managed values stay sound.

### Taking an address

`&` takes the address of a local, a value parameter, a field of a struct local, or a mutable static. A field of a class object, an array element and a `ref` or `in` parameter live in storage an owner can free, so their address is taken with `fixed`:

```nz output="6 9\n"
package main;

import std.io.Console;

struct Point {
    public int X;
    public int Y;
}

void Main(string[] args) {
    int[] values = new int[3];
    int sum = 0;
    unsafe {
        fixed (int* p = values) {
            for (int i = 0; i < 3; i++) {
                p[i] = i + 1;
            }
        }
        Point pt = new Point();
        Point* q = &pt;
        q->Y = 9;
        int* s = &sum;
        *s = values[0] + values[1] + values[2];
        Console.WriteLine($"{sum} {pt.Y}");
    }
}
```

`fixed (int* p = arr)` points at the first element, `fixed (int* p = &obj.X)` at a field, and `fixed (byte* p = "abc"u8)` at a UTF-8 literal. The variable is a pointer, and an expression that is already a pointer is not fixed again. For the block, the root object is **frozen**: moving it, assigning it, dropping it, passing it by `ref`, or calling a method that may replace what is fixed are errors, while reading and writing its fields stays allowed. The `fixed` pointer itself cannot be changed. Two limits: a second `Shared<T>` or `Arc<T>` handle to the same object is not frozen, and `fixed` on a `string` is not supported yet (NZ strings are UTF-8).

`&` of a captured local is refused only inside the local function that captures it; the enclosing function may take it, which is sound because captures are addresses of stack storage that never moves. And `&` takes a mutable static directly.

### `stackalloc`

`byte* buf = stackalloc byte[64];` and `byte* s = stackalloc byte[] { 104, 105, 0 };` allocate on the stack. The buffer lives until the function returns, also when it is allocated in an inner block, and it is zeroed. The length must be a constant, and a `stackalloc` stands only as the initializer of a pointer local. Inside a loop each pass takes more stack, so the compiler warns:

```nz warning="`stackalloc` in a loop"
package main;

void Main(string[] args) {
    unsafe {
        for (int i = 0; i < 3; i++) {
            byte* p = stackalloc byte[16];
            p[0] = 1;
        }
    }
}
```

A buffer too large for the stack ends the program with a crash, with no `nz:` message.

### Manual memory: `NativeMemory`

`import std.interop.NativeMemory;` gives six functions, inside `unsafe`: `Alloc(nuint size)`, `AllocZeroed(nuint count, nuint size)`, `Realloc(void* p, nuint size)`, `Free(void* p)`, `Copy(void* source, void* destination, nuint size)` and `Clear(void* p, nuint size)`. Out of memory stops the program with `nz: out of memory`. The runtime counts the blocks: `NZ_ALLOC_REPORT=1` prints `nz: live native blocks = N` before the owned-objects line, and `nzc test` fails a test that leaves a block, with both counts when it leaks both.

```nz output="104 105 30\n"
package main;

import std.io.Console;
import std.interop.NativeMemory;

void Main(string[] args) {
    unsafe {
        byte* text = stackalloc byte[] { 104, 105, 0 };
        long* numbers = (long*)NativeMemory.Alloc((nuint)(sizeof(long) * 3));
        for (int i = 0; i < 3; i++) {
            numbers[i] = (i + 1) * 5;
        }
        long total = numbers[0] + numbers[1] + numbers[2];
        NativeMemory.Free(numbers);
        Console.WriteLine($"{text[0]} {text[1]} {total}");
    }
}
```

### The `unsafe` keyword

- **`unsafe`** is valid on functions, methods, classes and structs. It allows the use of mutable statics and the systems features of [15.6](#156-unsafe-memory) (pointers, `stackalloc`, `NativeMemory`, calls into C): the ownership and borrowing rules are not suspended inside it.
- **Calling an `unsafe` function needs `unsafe`.** A call of an `unsafe` function or method, or of any method of an `unsafe` class or struct, is written inside an `unsafe` block or inside a function, method or type marked `unsafe`: ``calling unsafe function `Bump` needs an unsafe context; wrap the call in `unsafe { }` ``. So is `new T()` when a field initializer of an `unsafe` type `T` is not a constant expression (it calls something, creates an object with `new`, or writes or reads a static). The rule is lexical: a function that is not `unsafe` stays safe, whoever calls it, and the need does not spread to its callers.
- **The `unsafe` block is the vouch point.** As in Rust, the programmer who writes `unsafe { }` vouches for what is inside it, wherever the block appears. The checker sees one function at a time, so a safe function that contains an `unsafe` block which assigns a static is as dangerous to a caller lending it that static (by `in`, `ref` or as a receiver) as an `unsafe` function is: the lent value is freed under the caller. Writing the block is taking on that responsibility.

## 15.7 Memory problems and how NZ prevents them

This section is not written yet.

Outline:

- Leaks
- Double free
- Use after free
- Cycles

## 15.8 Detecting leaks: sanitizer and trace modes

This section is not written yet.

Outline:

- The leak rule of `nzc test`
- `Diag.LiveObjects()`
- Sanitizers

## 15.9 Memory layout and limits

This section is not written yet.

Outline:

- Layout of classes, structs and unions
- Stack size
- Limits

## 15.10 Memory without a garbage collector

This section is not written yet.

Outline:

- What a garbage collector does
- What NZ does instead

Previous: [14. Preprocessor directives](14-preprocessor-directives.md#14-preprocessor-directives) · [Contents](README.md) · Next: [16. Performance engineering](16-performance-engineering.md#16-performance-engineering)
