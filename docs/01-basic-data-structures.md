# 1.4 Data Structures

Part of chapter [1. Basic](01-basic.md#1-basic).

## 1.4.1 Array

An array `T[]` has a fixed length, lives on the heap and owns its elements. It is made empty with `new T[n]` (every element is the default value of `T`, so `T` needs one: `new string[3]` holds three empty strings and `new string?[3]` three `null`s, while `new Node[3]` for a class `Node` is an error; use `new Node?[3]`), or from its elements:

```nz output="3 2 4 3\nimport package\n9 10 2\n"
package main;
import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    int[] a = new int[] { 1, 2, 3 };
    long[] b = new[] { 1, 2L };          // the best common type of the elements
    int[] c = { 4, 5 };                  // only as the initializer of a declared array
    int[] d = [6, 7, 8];                 // a collection expression
    Console.WriteLine($"{a[2]} {b[1]} {c[0]} {d.Length}");

    string[] words = ["import", "package"];
    Console.WriteLine($"{words[0]} {words[1]}");

    List<int> l = [9, 10];               // a collection expression makes a list too
    Console.WriteLine($"{l[0]} {l[1]} {l.Count}");
}
```

`new T[n] { ... }` takes a constant `n` equal to the number of elements. `new[] { ... }` needs elements with a best common type (`new[] { 1, "a" }` is an error). `{ ... }` alone initializes only a declared array variable or field (`var a = { 1 };` is an error).

### Collection expressions

A collection expression `[a, b]` takes its type from where it stands: a declared variable or field, an assignment, a `return`, an argument, a `?:` arm (from the target, or else from the other arm: `var q = c ? [2] : b;`), an element of a typed tuple (`(int[], int) t = ([1, 2], 3);`), or the value of `new Shared<int[]>(...)`. Its target is an array or a `List<T>`. It has no type of its own, so `var xs = [1, 2];` is an error:

```nz fragment error="there is no target type for the collection expression"
var xs = [1, 2];
```

**Ownership.** The elements are evaluated left to right, then the array or list is made and each element is moved into it (a class object, a list, an array) or counted (a string, a `Shared<T>` handle), as when it is passed to `Add`. A variable moved into a collection cannot be used afterwards, and an element of another collection cannot be moved out to make a new one:

```nz fragment error="use of moved value `n`"
var n = new List<int>();
List<int>[] all = [n];
int k = n.Count;
```

**Spread.** An element written `..e` copies every element of `e` (an array, a `List<T>` or a span) into the new collection, in order. The source is only read: it keeps its own elements.

```nz output="1 2 3 4 5\n2\n"
package main;
import std.io.Console;
import std.collections.List;

void Main(string[] args) {
    int[] a = [1, 2];
    List<int> l = [3];
    int[] all = [..a, ..l, 4, 5];
    Console.WriteLine($"{all[0]} {all[1]} {all[2]} {all[3]} {all[4]}");
    Console.WriteLine($"{a.Length}");
}
```

Numbers, `bool`, `char` and enums are copied; a string or a `Shared<T>` handle gets a count of its own, as when it is passed by value. Elements that are owned (class objects, arrays, lists) cannot be spread, because each would get a second owner:

```nz fragment error="its elements are owned and would get a second owner"
List<int[]> rows = [[1], [2]];
List<int[]> copy = [..rows];
```

## 1.4.2 List

This section is not written yet.

Outline:

- Making a list
- Adding and removing elements
- Who owns the elements
- `foreach` over a list

## 1.4.3 Dictionary

This section is not written yet.

Outline:

- Making a dictionary
- Adding, reading and removing entries
- Keys and values

## 1.4.4 Struct

Structs have their own chapter: [4. Struct](04-struct.md#4-struct).

## 1.4.5 Interface

An **interface** lists methods that a class or struct promises to have. A type names its interfaces after a colon (`class Dog : IPet`), and an interface may inherit other interfaces (`interface IPet : IAnimal`). Interface members are public methods, so their names start with an uppercase letter; the implementing method has the same name, parameters (with their `in`/`ref` modes) and return type, and is a public instance method. A member may have a body: a **default member**, used by every type that does not declare the member itself. An interface holds methods only: no fields, properties, operators, static members or generic methods.

A class converts to any interface it implements, directly or through inherited interfaces, and the **interface value** owns the object as the class variable did: the conversion moves it, and dropping the interface value drops the object. A call through an interface value runs the method of the object's class (dynamic dispatch). Interface values go wherever class values go: locals, parameters, fields, `I?`, lists, arrays and dictionary values. To share an interface value, share a class that holds it.

```nz output="I say woof\na cat\nRex says woof\nI say woof\nI say woof\n"
package main;

import std.io.Console;
import std.collections.List;

interface IAnimal {
    readonly string Speak();
    readonly string Describe() {
        return $"I say {Speak()}";
    }
}

interface IPet : IAnimal {
    readonly string Name();
}

class Dog : IPet {
    string Speak() { return "woof"; }
    string Name() { return "Rex"; }
}

class Cat : IAnimal {
    string Speak() { return "meow"; }
    string Describe() { return "a cat"; }
}

void Main(string[] args) {
    var zoo = new List<IAnimal>();
    zoo.Add(new Dog());
    zoo.Add(new Cat());
    foreach (var a in zoo) {
        Console.WriteLine(a.Describe());
    }
    IPet pet = new Dog();
    Console.WriteLine($"{pet.Name()} says {pet.Speak()}");
    IAnimal animal = pet;
    Console.WriteLine(animal.Describe());
    var d = new Dog();
    Console.WriteLine(d.Describe());
}
```

**`readonly` members.** A call through an interface cannot see which method runs, so the interface says whether a member changes the object: a member written `readonly` promises not to, and only `readonly` members can be called through an `in` interface parameter (or a `readonly` default member). An implementing method of a `readonly` member must not change the object (it reads fields, and calls only `readonly` members or methods that change nothing). The marker is NZ's own: it is what lets the compiler check borrows through an interface.

```nz error="cannot call `Bump` on `c`, which it changes: `c` is an `in` parameter"
package main;

interface ICounter {
    readonly int Peek();
    void Bump();
}

class Counter : ICounter {
    int n;
    int Peek() { return n; }
    void Bump() { n++; }
}

void Show(in ICounter c) {
    c.Bump();
}

void Main(string[] args) { }
```

**Structs** implement interfaces too, but a struct is never held in an interface-typed variable (that would need boxing): it reaches an interface's members through a generic constraint, which calls the struct's method directly (see [8.5 Constraints](08-generics.md#85-constraints)). Comparing two interface values with `==` or `!=` is refused (there is no reference identity); `x == null` on an `I?` works.

```nz error="a struct cannot be held in an interface-typed variable"
package main;

interface IShape {
    readonly int Area();
}

struct Square : IShape {
    int Side;
    int Area() { return Side * Side; }
}

void Main(string[] args) {
    IShape s = new Square();
}
```

```nz error="cannot compare interface values with `==`"
package main;

interface IShape {
    readonly int Area();
}

class Square : IShape {
    int Area() { return 4; }
}

void Main(string[] args) {
    IShape a = new Square();
    IShape b = new Square();
    var same = a == b;
}
```

A missing member, a member with another signature, a static or non-public implementing method, a cycle of interfaces and listing an interface twice are errors; when a type inherits two different default members for one name (a diamond), it must declare the member itself. A struct must declare every default member that code given that struct can reach: a struct can be the argument of a bound that has default members as long as that code reaches none of them. For a generic function this is what its body calls; for a generic type it is every method of the type, whether the program calls that method or not.

## 1.4.6 Enum

**Enums** are a named set of integer constants:

```nz
package main;

enum Color { Red, Green, Blue }

void Main(string[] args) {
    var c = Color.Red;
}
```

**Enums with an underlying type and values.** An enum is stored as an integer, `int` unless written otherwise: `enum Level : byte { ... }` accepts any of `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long` and `ulong` (not `nint`, `nuint`, `char` or `bool`). A member without a value is the previous one plus one, starting at 0. A value is a constant expression of integers that fits the underlying type. It may name any member of the same enum, before or after it (only a cycle is an error, and the message names the members), and a member of another enum only through a cast, as `(int)Level.High`. Two members may share a value. An enum is copied, never moved. An enum value prints as its member name (the first member with that value), or as the number when no member has it.

```nz output="Green High 11 All 3 A 11 X 2\n"
package main;

import std.io.Console;

enum Color { Red, Green, Blue }
enum Level : byte { Low = 1, Mid = 10, High }
enum Perm : uint { None = 0, Read = 1, Write = 2, All = Read | Write }
enum Fwd { A = B + 1, B = 10 }
enum Other : long { X = (long)Fwd.B * 1000000000 }
enum Mix { P = (int)Level.High, Q = (int)Perm.Write }

void Main(string[] args) {
    Level l = Level.High;
    Perm p = Perm.Read | Perm.Write;
    Console.WriteLine($"{Color.Green} {l} {(int)l} {p} {(uint)p} {Fwd.A} {(int)Fwd.A} {Other.X} {(int)Mix.Q}");
}
```

```nz error="the values of `A` and `B` depend on each other"
package main;

enum E { A = B, B = A }

void Main(string[] args) { }
```

```nz error="the value after `A` does not fit `byte`"
package main;

enum E : byte { A = 255, B }

void Main(string[] args) { }
```

```nz error="cannot implicitly convert `A` to `long`; write `(long)A.Y`"
package main;

enum A { X = 1, Y = 2 }
enum C : long { R = A.Y }

void Main(string[] args) { }
```

**Operators on enums.** `|`, `&`, `^` and `~` on one enum give that enum; `e + n`, `n + e` and `e - n`, with `n` an integer that fits the underlying type, give the enum; `e1 - e2` gives the underlying type; `++`, `--` and the compound `+= -= |= &= ^=` work on an enum variable; `== != < <= > >=` compare two values of one enum. An integer variable added to an enum must convert implicitly to the underlying type: `Day + int` works for an `int` (or `long`) enum, while an `int` variable added to an enum of `sbyte`, `byte`, `short`, `ushort`, `uint` or `ulong` is an error (a constant that fits works, as `Perm.Read + 2` below). Arithmetic is checked in the underlying type, so a result outside it stops the program: with ``nz: integer overflow`` for an `int`, `uint`, `long` or `ulong` enum, and with ``nz: value does not fit in `Perm` `` for the four smaller ones; `unchecked` wraps. Anything else (`+` on two enums, comparing two different enums) is an error. A format such as `{p:X}` needs an integer, so it is refused: cast first.

```nz output="Exec 3 3 3 3 3 Write True True True 255 True\n"
package main;

import std.io.Console;

enum Perm : byte { None = 0, Read = 1, Write = 2, Exec = 4 }

void Main(string[] args) {
    Perm p = Perm.Read | Perm.Write;
    p |= Perm.Exec;
    p &= ~Perm.Write;
    p ^= Perm.Read;
    Perm q = Perm.Read + 2;
    Perm r = 2 + Perm.Read;
    Perm s = Perm.Exec - 1;
    byte diff = Perm.Exec - Perm.Read;
    var d2 = Perm.Exec - Perm.Read;
    Perm t = Perm.None;
    t++;
    t++;
    Console.WriteLine($"{p} {q} {r} {s} {diff} {d2} {t} {Perm.Read < Perm.Write} {Perm.Read == Perm.Read} {Perm.Exec >= Perm.Write} {~Perm.None} {(Perm.Read | Perm.Write) == (Perm)3}");
}
```

```nz error="cannot apply `+` to `A` and `A`"
package main;

enum A { X }

void Main(string[] args) {
    A a = A.X;
    var r = a + a;
}
```

**Enum casts.** An enum casts to and from any integer type, to and from `char`, `float` and `double`, and to another enum, always through the underlying type. A cast into an enum checks the underlying range, whatever the value means: `(Level)7` is valid even though no member is 7, and prints `7`; a real is truncated first. A constant that does not fit is a compile-time error, a variable stops the program naming the enum, and `unchecked` keeps the low bits of an integer (a real stays checked). Without a cast an enum does not convert to or from an integer: `int i = Level.High;` is an error.

```nz output="200 1 66 200 3 Q\n"
package main;

import std.io.Console;

enum A : byte { X = 1, Y = 200 }
enum B : int { P = 1, Q = 2 }

void Main(string[] args) {
    A a = A.Y;
    B b = (B)a;
    char c = (char)A.X;
    A fromC = (A)'B';
    double d = (double)A.Y;
    A fromD = (A)3.7;
    Console.WriteLine($"{b} {(int)c} {fromC} {d} {fromD} {(B)2.9}");
}
```

```nz error="constant `300` does not fit `A`"
package main;

enum A : byte { X = 1, Y = 200 }

void Main(string[] args) {
    A a = (A)300;
}
```

A `switch` on an enum compares by value, so two arms for members that share a value are an error (``pattern `E.B` is already handled by an earlier arm``), and an enum member is not a pattern for an integer scrutinee.

## 1.4.7 Object

This section is not written yet.

## 1.4.8 Tuple

A **tuple** groups two or more values of any types into one value: `(int, string)`. It is written as a type between parentheses, and as an expression with commas, `(7, "seven")`. A tuple has at least two elements and no upper limit; `(int)` is not a tuple and `()` is not a type. An element can itself be a tuple.

**Element access.** The elements are `Item1`, `Item2`, and so on, counted from 1. A tuple type may also name its elements, `(int Lo, int Hi)`, and then `r.Lo` and `r.Item1` are the same element. The names belong to the written type only, never to the value: `(int x, int y)`, `(int a, int b)` and `(int, int)` are **one type**, so they are assigned to each other freely, and `var p = r;` keeps the names of `r` only because `var` copies the type. An element can be assigned, `t.Item1 = 5;`.

**Conversion.** A tuple converts to another tuple of the same length when each element converts (the widening rules of 1.1.1: `(int, int)` to `(long, double)`). It never narrows, and a tuple of another length is a different type. How the arms of `?:` and `switch` convert when they are tuples is not settled yet; the compiler widens element by element there too, and this may change.

**Deconstruction.** `var (a, b) = e;` declares one variable per element of the tuple `e`, each with the type of its element. A `_` in place of a name discards that element. There must be one name per element, and at least two names.

```nz output="7 seven\n7 seven seven\n2 9 9\n1 2 4 10\n"
package main;

import std.io.Console;

(int, string) Pair() {
    return (7, "seven");
}

(int Lo, int Hi) Range(int a, int b) {
    return (a, b);
}

void Main(string[] args) {
    var p = Pair();
    Console.WriteLine($"{p.Item1} {p.Item2}");
    var (n, s) = Pair();
    var (_, text) = Pair();
    Console.WriteLine($"{n} {s} {text}");
    var r = Range(2, 9);
    Console.WriteLine($"{r.Lo} {r.Hi} {r.Item2}");
    (long, double) wide = (1, 2);
    (int, int) same = (3, 4);
    (int x, int y) named = same;
    named.x = 10;
    Console.WriteLine($"{wide.Item1} {wide.Item2} {named.y} {named.Item1}");
}
```

Deconstruction also works in a `for` initializer. It declares new variables only: `(a, b) = (b, a);` (assignment to existing variables), a nested `var ((a, b), c) = e;` and a typed form `(int, string) (a, b) = e;` are not supported, and neither is deconstruction in `foreach`.

```nz fragment error="cannot deconstruct a tuple of 3 elements into 2 names"
var (a, b) = (1, 2, 3);
```

```nz fragment error="only a tuple or a positional record struct can be deconstructed"
var (a, b) = 5;
```

**Rules for the names.** Two elements cannot share a name, and the name `ItemN` is allowed only on position N (`(int Item2, int y)` is an error; `(int Item1, int y)` is fine). Reading an element that does not exist is an error naming the tuple type.

```nz fragment error="has no element `Item3`"
var t = (1, 2);
int z = t.Item3;
```

**What a tuple cannot do (yet).**

- **No `==` or `!=`.** Compare the elements.
- **No formatting.** A tuple in an interpolation hole is refused; format its elements.
- **No `null` element without a type.** `var t = (1, null);` cannot infer the element type; write the type, `(int, string?) t = (1, null);`. A conditional with a tuple on one side and `null` on the other, `b ? (1, 2) : null`, is refused too.
- **No `foreach` with a tuple type.** `foreach ((int, int) p in xs)` is refused as `not supported yet: tuple types`. (The type written in a `foreach` is not checked against the elements of the collection at all yet; a wrong type there is also accepted. This is an old gap that is not settled.)
- **No default value.** A tuple type has no default, so a field of tuple type needs an initializer, or an optional type.

```nz fragment error="cannot apply `==`"
var a = (1, 2);
var b = (1, 2);
bool same = a == b;
```

```nz fragment error="cannot format a tuple in an interpolated string"
var t = (1, 2);
Console.WriteLine($"{t}");
```

**Ownership.** A tuple is copied when all its elements are, and moves as a whole when one of them is owned (a class object, a list, an array); see [Ownership](+/Ownership.md). Moving one element out of an owned tuple is refused, but `in` lends it and deconstruction takes every element at once.

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

**Tuples and local functions.** A tuple can be a parameter or a result of a local function, and a local function can read a tuple it captures. `(int, int) swap((int, int) p) { ... }` written in a block is a local function, like any other type followed by a name and `(`.

**Limits of the compiler.** A tuple type nests at most 256 levels, and parentheses in an expression at most 10000.

## 1.4.9 Anonymous object

Not supported.

## 1.4.10 Blocking queue (BlockingCollection, Channel)

This section is not written yet.

Outline:

- `BlockingCollection<T>`
- Channels

## 1.4.11 Others (HashSet, Stack, Queue, Span, ranges)

The members of `HashSet<T>`, `Stack<T>` and `Queue<T>` are listed in [Appendix A. Standard library reference](appendix-a-standard-library.md#appendix-a-standard-library-reference).

### Indexes from the end and ranges

`^n` is an `Index` counted from the end (`^1` is the last element) and `a..b` is a `Range` from `a` up to, not including, `b`; either bound may be left out (`1..`, `..^1`, `..`). `Index` and `Range` are values like numbers: they can be stored, passed and returned, and an `int` converts to an `Index`. An array, a `List<T>` and a `string` take an `Index`; a `string` also takes a `Range` and gives a new string, as `Substring` does:

```nz output="4 2 o\nhel ell lo\n1..^1\n"
package main;
import std.io.Console;

void Main(string[] args) {
    int[] a = [1, 2, 3, 4];
    string s = "hello";
    Index last = ^1;
    Console.WriteLine($"{a[last]} {a[^3]} {s[^1]}");
    Range r = 1..^1;
    Console.WriteLine($"{s[..3]} {s[1..4]} {s[^2..]}");
    Console.WriteLine($"{r}");
}
```

`^0` is one past the end, so `a[^0]` stops the program with `index out of range`; a negative value after `^` stops with `ArgumentOutOfRangeException`, and so does a range outside the string. A range binds tighter than `+` and `-`: `1..n-1` is `(1..n) - 1`, an error; write `1..(n - 1)`. A `List<T>` cannot be sliced, and slicing an array is not supported yet.

### Spans

A `Span<T>` is a view of elements another value owns: a pointer and a length. It is a parameter type: `Span<T>` lets the function read and write the elements, `in Span<T>` only read them. An array converts to a span of its elements, and so does a collection expression (`Sum([1, 2])`); a span parameter can be indexed (also with `^`), measured with `.Length`, iterated with `foreach`, sliced with a range (giving a span over part of the same storage) and passed on to another span parameter:

```nz output="10 5 7\n1 0 0 4\n"
package main;
import std.io.Console;

int Sum(in Span<int> s) {
    int t = 0;
    foreach (int x in s) {
        t += x;
    }
    return t;
}

int Inner(in Span<int> s) {
    return Sum(s[1..^1]);
}

void Clear(Span<int> s) {
    for (int i = 0; i < s.Length; i++) {
        s[i] = 0;
    }
}

void ClearMiddle(Span<int> s) {
    Clear(s[1..^1]);
}

void Main(string[] args) {
    int[] a = [1, 2, 3, 4];
    Console.WriteLine($"{Sum(a)} {Inner(a)} {Sum([3, 4])}");
    ClearMiddle(a);
    Console.WriteLine($"{a[0]} {a[1]} {a[2]} {a[3]}");
}
```

A span lives only as long as the call it is passed to: it cannot be stored in a local variable, a field, a static, an array or a list, returned, used as a type argument, or captured by a lambda that is kept (stored, returned, or passed to a function that keeps it); a lambda called while the span lives may use it. In NZ `in` is read-only, as for every `in` parameter. `ReadOnlySpan<T>` is not supported yet. How a span borrows its source is in [Ownership](+/Ownership.md#spans-borrow-for-one-call).

## 1.4.12 Union

A union holds exactly one value of one of its case types. It is declared with its cases:

```nz output="cat Tom\nbird with 2 wings\n2\nTrue dog of 3\n"
package main;

import std.io.Console;

class Cat { public string Name = "Tom"; }
class Dog { public int Age = 3; }
struct Bird { public int Wings = 2; }

union Pet(Cat, Dog, Bird);

string Describe(in Pet p) {
    return p switch {
        Cat c => $"cat {c.Name}",
        Dog d => $"dog of {d.Age}",
        Bird b => $"bird with {b.Wings} wings",
    };
}

void Main(string[] args) {
    Pet p = new Cat();
    Console.WriteLine(Describe(p));
    p = new Bird();
    Console.WriteLine(Describe(p));
    if (p is Bird b) {
        Console.WriteLine($"{b.Wings}");
    }
    Pet q = (Pet)new Dog();
    Console.WriteLine($"{q is Dog} {Describe(q)}");
}
```

**Cases.** A case is any type that is not optional: a built-in type, a class, struct, enum, tuple, another union, or a generic instance (`union Maybe<T>(Some<T>, None);`). A union needs at least two cases, two cases may not be the same type (or become the same type for some type arguments, as `Box<T>` and `Box<int>` can), and a union is not a case of itself. A bare type parameter (`union U<T>(T, Dog)`) is not a case, because it could be the same type as another case. Arrays and interfaces as cases are not supported yet. A union has no fields of its own: the data lives in its case types.

**Conversion.** A value of a case type converts to the union implicitly, as above, and an explicit cast `(Pet)new Dog()` works. When the value's type is not itself a case, the case it converts to is chosen; when it converts to more than one case, NZ refuses it and asks for a cast to the case that is meant:

```nz error="converts to more than one case"
package main;

union Number(long, double);

void Main(string[] args) {
    Number n = 5;
}
```

`Number n = (long)5;` and `Number m = 2.5;` are accepted. The arms of `?:` and of a `switch` expression take their natural type first, when the arms have one (`int` and `long` give `long`); only when they have none is each arm converted to the target type, so `Pet p = c ? new Cat() : new Dog();` is a `Pet`.

**No null, no default.** A union always holds one of its cases. `Pet p = null;` is an error (``a union is never null; declare `Pet?` ``), and so is `default` for a union (`` `Pet` has no default value ``). Write `Pet?` where "maybe none" is meant.

**Storage.** A union is a value: a tag and the case it holds, stored in place with no allocation. It is `Copy` when every case is `Copy`; when a case holds a string (and none is owned) it is counted like a struct of strings, copied freely with one more count on the string it holds; otherwise it is owned: an owned union moves as a whole. Dropping a union drops only the case it holds (see [Ownership](+/Ownership.md)).

**Methods.** A union body may hold methods. `this` is the union, borrowed, and `this switch { ... }` is the usual way to read it:

```nz output="2 4\n"
package main;

import std.io.Console;

class Cat { }
class Dog { }
struct Bird { public int Wings = 2; }

union Pet(Cat, Dog, Bird) {
    public int Legs() {
        return this switch {
            Bird b => b.Wings,
            _ => 4,
        };
    }
}

void Main(string[] args) {
    Pet p = new Bird();
    Pet q = new Dog();
    Console.WriteLine($"{p.Legs()} {q.Legs()}");
}
```

Properties and constants in a union body are not supported yet. A union has no text form and no `==` yet: format the case it holds, and test the case with `is` or `switch` (see "Not yet" in [Appendix B](appendix-b-current-behavior.md#appendix-b-current-behavior-notes)).

Previous: [1. Basic](01-basic.md#1-basic) · [Contents](README.md) · Next: [1.5 Operators](01-basic-operators-control-flow.md#15-operators)
