# Memory Model

> Status: tier 1 is implemented (stage 4). Ownership, moves, `in`/`ref` borrows and deterministic drops are enforced by the compiler; see [Ownership](Ownership.md) for what it really does. Strings are reference-counted and freed with their last holder; see [Ownership: Strings](Ownership.md#strings). Tier 2 (`Shared<T>`, `Arc<T>`, `WeakReference<T>` in `std.memory`) is implemented; see [Ownership: Shared objects](Ownership.md#shared-objects). Tier 3 does not exist yet.

NZ has no garbage collector by default, yet it stays memory safe. The approach is **two layers** on top of one shared core, so the ecosystem does not split into two coding styles.

## Tier 1: No GC (the language core)

- **Move by default** for types that own a resource. A `class` is a heap object with a single owner.
- `in` borrows for reading, `ref` borrows for changing. There is no `&` symbol.
- The core rule: one owner, and either many read borrowers **or** one writing borrower, never both.
- Lifetimes are inferred. Explicit annotations are needed only in rare cases and use a readable word, not `'a`.
- Deterministic cleanup through `defer` and `using`.
- Explicit allocators, and *arena per request* as the main pattern for backends.

## Tier 2: ARC

Library types for data that really has to be shared: `Arc<T>` (atomic, across threads), `Shared<T>` (non-atomic, one thread), and `WeakReference<T>` to break cycles.

- `new Shared<T>(v)` moves `v` into a new block. Copying a handle shares the object and increments the count; the last strong handle drops the object. The content of a handle is a class, struct, tuple, list or array; `Shared<int>` is rejected.
- A `Shared<T>` object can be changed through any handle. Borrows of a part that owns something are checked while the program runs (always on): changing a part that is currently lent stops the program with `nz: shared object changed while it is lent`.
- An `Arc<T>` object is read-only, and an `Arc<T>` that contains a `Shared` or a `WeakReference` (at any depth) is rejected.
- `w.TryGetTarget()` returns `Shared<T>?`; `Shared.StrongCount(s)`, `Shared.WeakCount(s)` and `Arc.StrongCount(a)` are static.
- A cycle of strong handles leaks visibly (`Diag.LiveObjects()`, `nzc test`); break it with `WeakReference<T>` as the back edge.
- A long chain through a field whose type is the same as its content is dropped in a loop; other shapes (a list of handles, a struct field, alternating types) are dropped recursively and can overflow the C stack if they are too deep.
- A temporary handle lives until the end of its statement.

**Why ARC is not the default:** on a many-core backend, ARC as the default causes atomic contention on shared objects, latency that is hard to predict, leaked cycles, chained frees that hurt p99, and memory overhead per object. So ARC is used only where it is explicit.

## Kernel mode

The `no_runtime` mode forbids tier 2 at the package level. Its first form is `nzc build --no-runtime` (guide section 1.13): the program links the C library only and includes `nz_core.h`, not `nz.h`. Strings, `Console`, classes, arrays, lists and the handles are refused there; numbers, structs, pointers, statics, `extern` calls, `stackalloc` and `NativeMemory` remain, and integer arithmetic stays checked.
