# Syntax Reference, Part 2: Declarations

> **Status: Experimental.** NZ is a young language and its syntax will change. NZ code (C# style) follows the syntax of C# 14 except where a deviation was decided on purpose; a piece of syntax that turns out not to fit the language will be changed.

This page lists how a program is declared: source files and packages, types, members, parameters, generics, interfaces, inheritance, delegates and events, attributes, preprocessor directives, async code, exceptions, reflection, unsafe code, and the ownership rules that exist only in NZ. [Part 1](Syntax-Reference.md) covers the rest: lexical structure, literals, operators and precedence, built-in types, variables, expressions, statements and patterns.

## How to read this page

Every construct has a short entry: the syntax, a sentence or two on what it does, an example, and two lines:

- **Status** says whether today's compiler accepts it.
- **Differs from C#** says in one line how NZ differs from C# 14, or "Same as C#".

| Status | Meaning |
|---|---|
| Available | Works on today's compiler. Every example marked Available is checked and run by the docs test. |
| Partial | Works, but a part is missing; the entry says which. |
| Planned | Decided, and coming. The example shows the C# syntax that NZ will accept; today the compiler refuses it. |
| Later | Wanted, but deferred: no date yet. |
| Not planned | Decided against. |
| NZ only | Exists only in NZ (no C# equivalent). The entry also gives its status. |

Every NZ code block on this page is a whole program and is checked by the docs test; a block that shows an error message is a program the compiler refuses today. Blocks shown as plain text are not checked, because the compiler does not accept them yet.

## Contents

1. [Source files and packages](#1-source-files-and-packages)
2. [Visibility](#2-visibility)
3. [Type declarations](#3-type-declarations)
4. [Members](#4-members)
5. [Constructors and object creation](#5-constructors-and-object-creation)
6. [Parameters and arguments](#6-parameters-and-arguments)
7. [Generics](#7-generics)
8. [Interfaces](#8-interfaces)
9. [Inheritance](#9-inheritance)
10. [Lambdas, delegates and events](#10-lambdas-delegates-and-events)
11. [Attributes](#11-attributes)
12. [Preprocessor directives](#12-preprocessor-directives)
13. [Async code and iterators](#13-async-code-and-iterators)
14. [Exceptions](#14-exceptions)
15. [Reflection](#15-reflection)
16. [Unsafe code and native interop](#16-unsafe-code-and-native-interop)
17. [Ownership and memory (NZ only)](#17-ownership-and-memory-nz-only)
18. [Not planned](#18-not-planned)
19. [Index](#19-index)

---

## 1. Source files and packages

### 1.1 Source file layout

A source file is UTF-8 text with the extension `.nz`. It holds, in this order: one `package` line, the `import` lines, then the declarations.

```nz output="Hello, NZ\n"
package main;

import std.io.Console;

void Main(string[] args) {
    Console.WriteLine("Hello, NZ");
}
```

**Status:** Available.
**Differs from C#:** C# files start with `using` directives and an optional `namespace`; NZ files start with `package` and `import`.

### 1.2 `package`

`package name;` is the first line of every file. A package is a **folder**: all `.nz` files directly inside one folder belong to one package, and a subfolder is another package. A dotted name such as `app.util` stands for the folder `app/util/`. All files of a package share one set of top-level names.

**Status:** Available.
**Differs from C#:** C# uses `namespace`, which is independent of folders. In NZ the folder decides the package, and `namespace` is not accepted (see 1.8).

### 1.3 `import`

An `import` names a package and one or more of its declarations. Four forms exist:

```text
import geo.Point;            // one name
import geo.Point as P;       // one name, under another name
import geo.{Point, Line};    // a group of names
import geo.*;                // every public name of the package
```

```nz output="1 x\n"
package main;

import std.io.Console as Out;
import std.collections.{List, Dictionary};
import std.text.*;

void Main(string[] args) {
    var l = new List<int>();
    l.Add(1);
    var sb = new StringBuilder();
    sb.Append("x");
    Out.WriteLine($"{l.Count} {sb.ToString()}");
}
```

An import belongs to one file. A name must be imported before it is used: `geo.Point p;` without an import is an error. Two packages cannot import each other, directly or through others.

**Status:** Available.
**Differs from C#:** C# writes `using geo;` (a whole namespace) and `using P = geo.Point;`. NZ imports names, has a group form, and a qualified name without an import is an error.

### 1.4 Folders as packages

`nzc run app` runs the folder `app` as the **entry package**. Its `import geo.Point;` loads the sibling folder `geo`. Only `public` declarations can be imported by another package.

```text
proj/
  app/main.nz     package app;  import geo.Point;  void Main(...) { ... }
  geo/point.nz    package geo;  public class Point { public int X = 7; }
```

**Status:** Available.
**Differs from C#:** C# groups files with a project file; NZ has no project file yet, the folders decide.

### 1.5 Entry point `Main`

The entry package declares one top-level `Main`, in one of C#'s four forms: `void Main()`, `void Main(string[] args)`, `int Main()`, `int Main(string[] args)`. An `int` result is the exit status. `args` holds the arguments without the program name.

```nz output="hi\n"
package main;

import std.io.Console;

int Main() {
    Console.WriteLine("hi");
    return 0;
}
```

**Status:** Available.
**Differs from C#:** `Main` is a top-level function, not a `static` method of a class. `async Task Main` is not available (see 13).

### 1.6 Top-level functions

A function may be declared at the top level of a file, outside any type. It is called by its name from anywhere in the package, and from other packages when it is public and imported.

```nz output="12\n"
package main;

import std.io.Console;

int Square(int x) {
    return x * x;
}

int Twice(int x) => x * 2;

void Main(string[] args) {
    Console.WriteLine($"{Twice(Square(2)) + 4}");
}
```

**Status:** Available.
**Differs from C#:** C# has no top-level functions (only methods of types, and local functions); in NZ they are the normal case and take no `static`.

### 1.7 Top-level constants and statics

A constant (`const`) and a static variable (`static`, `static readonly`) may also be declared at the top level.

```nz output="23\n"
package main;

import std.io.Console;

const int Max = 10;
static readonly int Base = 3;

void Main(string[] args) {
    Console.WriteLine($"{Max * 2 + Base}");
}
```

**Status:** Available. A mutable static needs `unsafe` to be read or written (see 4.4).
**Differs from C#:** C# has no top-level fields or constants; they live in a class.

### 1.8 `namespace` and `using` directives

NZ uses `package` and `import` in their place. `namespace` and the `using` directive (`using System;`, `using static`, `global using`) are not accepted. The `using` **statement** exists (see 17.6).

```nz error="expected `package name;` at the start of the file"
namespace App;

void Main(string[] args) { }
```

**Status:** Not planned.
**Differs from C#:** a deliberate deviation: `package` follows the folder and `import` names what it brings in.

### 1.9 Top-level statements

C# 9 allows statements at the top level of one file, without a `Main`. NZ requires `Main`.

```nz error="a function needs a return type"
package main;

import std.io.Console;

Console.WriteLine("top");
```

**Status:** Later.
**Differs from C#:** the C# form is not accepted; write `void Main(string[] args) { ... }`.

---

## 2. Visibility

### 2.1 The first-letter rule for members

For functions, methods, fields, properties, constants and type aliases, the **first letter of the name** decides the visibility when no keyword is written: `Add` is public, `add` and `_add` are private. Private means: inside the declaring type for a member, inside the declaring package for a top-level declaration.

```nz error="is local"
package main;

class Counter {
    int count;
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.count;
}
```

**Status:** Available. NZ only.
**Differs from C#:** C# members are private unless a keyword says otherwise; the letter means nothing in C#.

### 2.2 `public` and `private`

The keywords may be written. An explicit keyword wins over the letter, but `public` on a name that starts with a lowercase letter is an error, and `private` on an uppercase name gives a warning.

```nz error="a `public` name must start with an uppercase letter"
package main;

public class C {
    public int x;
}

void Main(string[] args) { }
```

**Status:** Available.
**Differs from C#:** the keywords mean the same; the naming checks are NZ's.

### 2.3 Visibility of types

A `class`, `struct`, `record`, `interface`, `enum` or `union` without a keyword is visible inside its package; `public` makes it visible to other packages. A constructor is as visible as its type, and cannot be more visible than its type.

```nz error="a constructor cannot be more visible than its type"
package main;

class Point {
    int x;
    public Point(int x) { this.x = x; }
}

void Main(string[] args) { }
```

**Status:** Partial. The default is C#'s `internal` (the package); writing `public` on a constructor of a type that is not public is still refused.
**Differs from C#:** C# accepts `public Point(...)` inside an internal type; NZ will too.

### 2.4 `internal`

`internal` names the default visibility of a type (the package) explicitly.

```text
internal class Cache { }
```

**Status:** Planned.
**Differs from C#:** today the word is not accepted; the meaning will be C#'s, with the package in place of the assembly.

### 2.5 `protected`

A `protected` member is visible in its class and in the classes derived from it. It comes with class inheritance (section 9). The word is reserved today.

```nz error="`protected` is reserved for a future feature"
package main;

class C {
    protected int X;
}

void Main(string[] args) { }
```

**Status:** Planned.
**Differs from C#:** Same as C# once inheritance exists.

### 2.6 `file`, `protected internal`, `private protected`

**Status:** Later.
**Differs from C#:** not accepted.

---

## 3. Type declarations

### 3.1 `class`

A class is a heap object with **one owner**. Assigning it, or passing it to a parameter without a modifier, **moves** it; it is freed when its owner goes out of scope. See [Ownership](Ownership.md).

```nz output="1\n"
package main;

import std.io.Console;

class Counter {
    int count;

    public int Next() {
        count += 1;
        return count;
    }
}

void Main(string[] args) {
    var c = new Counter();
    Console.WriteLine($"{c.Next()}");
}
```

**Status:** Available.
**Differs from C#:** an object has one owner and is moved, not shared by reference; there is no garbage collector.

### 3.2 `struct`

A struct is a value type: it is copied on assignment. A struct that holds an owned value (a class object, a list) is not copied but moved, like a class.

```nz output="5 2\n"
package main;

import std.io.Console;

public struct V {
    public int X;
    public int Y;

    public V(int x, int y) {
        X = x;
        Y = y;
    }

    public int Sum() => X + Y;
}

void Main(string[] args) {
    var a = new V(2, 3);
    var b = a;
    b.X = 0;
    Console.WriteLine($"{a.Sum()} {a.X}");
}
```

**Status:** Available.
**Differs from C#:** a struct that owns an object moves instead of being copied (a copy would give the object two owners).

### 3.3 `record class` and `record struct`

A positional record declares its members in its header: `record class Person(string Name, int Age);`. Every parameter becomes a public member, and `var (a, b) = r;` deconstructs it. `record` alone means `record class`. A body may add methods.

```nz output="Ada Lovelace 1 2\n"
package main;

import std.io.Console;

record class Person(string First, string Last) {
    public string Full() => $"{First} {Last}";
}

record struct Point(int X, int Y);

void Main(string[] args) {
    var p = new Person("Ada", "Lovelace");
    var (x, y) = new Point(1, 2);
    Console.WriteLine($"{p.Full()} {x} {y}");
}
```

**Status:** Partial. Value equality (`==`), the generated `ToString`, and `with` are missing (see 3.4).
**Differs from C#:** a record class has one owner like any class.

### 3.4 Record equality, `ToString` and `with`

C# generates value equality and a `ToString` for records, and `r with { X = 5 }` copies a record with some members changed.

```text
var b = a with { X = 5 };
```

**Status:** Planned. `==` on records will be refused until value equality is built; `with` on a record that owns objects will move the source into the copy.
**Differs from C#:** today `with` is not accepted.

### 3.5 `enum`

An enum is a set of named integer constants, stored as `int` or as the underlying type written after `:`. A member without a value is the previous one plus one. An enum value prints as its member name.

```nz output="High 11\n"
package main;

import std.io.Console;

enum Level : byte { Low = 1, Mid = 10, High }

void Main(string[] args) {
    Level l = Level.High;
    Console.WriteLine($"{l} {(int)l}");
}
```

**Status:** Available.
**Differs from C#:** `[Flags]` and `HasFlag` do not exist yet (see 11.4); `Enum.GetValues` and the other members of `Enum` come with reflection (section 15).

### 3.6 `union`

A union holds exactly one value of one of its case types. The cases are existing types; a value of a case converts to the union, and `switch` or `is` reads it. A union is never `null`; write `Pet?` for "maybe none".

```nz output="cat Tom\ndog of 3\n"
package main;

import std.io.Console;

class Cat { public string Name = "Tom"; }
class Dog { public int Age = 3; }

union Pet(Cat, Dog);

string Describe(in Pet p) {
    return p switch {
        Cat c => $"cat {c.Name}",
        Dog d => $"dog of {d.Age}",
    };
}

void Main(string[] args) {
    Pet p = new Cat();
    Console.WriteLine(Describe(p));
    p = new Dog();
    Console.WriteLine(Describe(p));
}
```

A union may be generic, and may have methods. `Result<T, E>` is a union of the standard library (see [Error Handling](Error-Handling.md)).

**Status:** Available. Properties and constants in a union body, `==`, and a text form are missing.
**Differs from C#:** C# 14 has no unions; NZ follows the shape proposed for C# 15. A value that converts to two cases is refused, where C# 15 would pick one.

### 3.7 `interface`

An interface declares members that a type promises to have. Today an interface can be declared, but no type can implement it yet (section 8).

```nz
package main;

interface IShape {
    int Area();
    string Name();
}

void Main(string[] args) { }
```

**Status:** Partial. Implementing an interface, default members, interface inheritance and generic constraints are Planned (section 8).
**Differs from C#:** see section 8.

### 3.8 `delegate`

`delegate int Op(int a, int b);` declares a named function type. See section 10.

**Status:** Planned.
**Differs from C#:** see 10.3.

### 3.9 Type aliases: `type`

`type Name = Type;` gives a type another name. An alias is transparent: messages show the type it stands for. An alias may have type parameters.

```nz output="7 3\n"
package main;

import std.io.Console;
import std.collections.List;

type Id = int;
type Ids = List<Id>;
type Pair<T> = (T, T);

void Main(string[] args) {
    Ids xs = new List<int>();
    xs.Add(7);
    Pair<int> p = (1, 2);
    Console.WriteLine($"{xs[0]} {p.Item1 + p.Item2}");
}
```

**Status:** Available.
**Differs from C#:** C# writes `using Id = int;` and has no generic aliases; NZ uses `type` because `using` is the resource statement.

### 3.10 `static class`

A static class holds only static members and constants; it has no instances.

```nz output="True\n8\n"
package main;

import std.io.Console;

static class Chars {
    public const int Base = 4;
    public static bool IsDigit(char c) { return c >= '0' && c <= '9'; }
    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    Console.WriteLine(Chars.IsDigit('5'));
    Console.WriteLine(Chars.Twice(Chars.Base));
}
```

**Status:** Available. A static class nested in another type is not supported yet.
**Differs from C#:** extension methods are declared as top-level functions, not inside a static class (see 6.8).

### 3.11 Nested types

A `class`, `struct`, `record`, `enum` or `union` may be declared inside a class, struct, record or union. From outside it is written `Outer.Inner`. A nested type is private to its enclosing type unless it is `public`.

```nz output="1 2 Low\n"
package main;

import std.io.Console;

public class Shape {
    public class Point {
        public int X = 1;
        public int Y = 2;
    }

    public enum Kind { Low, High }
}

void Main(string[] args) {
    var p = new Shape.Point();
    Console.WriteLine($"{p.X} {p.Y} {Shape.Kind.Low}");
}
```

**Status:** Partial. A type nested in an interface, a `static` nested class, an enum nested in a generic type, and static members of a type nested in a generic type are not supported yet.
**Differs from C#:** an `import` names top-level types only: import `Shape`, then write `Shape.Point`.

### 3.12 `partial`

`partial class C` splits one type over several declarations.

**Status:** Later.
**Differs from C#:** not accepted.

### 3.13 `abstract`, `sealed`, `virtual`, `override`, `new` (modifier)

These belong to class inheritance (section 9). The words are reserved today.

```nz error="`abstract` is reserved for a future feature"
package main;

abstract class Shape { }

void Main(string[] args) { }
```

**Status:** Planned.
**Differs from C#:** Same as C# once inheritance exists; classes stay open by default.

### 3.14 `readonly struct` and `ref struct`

**Status:** Later.
**Differs from C#:** not accepted. A `Span<T>` is a borrowed view that already obeys the rules of a `ref struct` (see Part 1).

### 3.15 Generic types

Classes, structs, records, unions, interfaces and aliases take type parameters: `class Box<T> { ... }`. See section 7.

**Status:** Available.
**Differs from C#:** generic code is compiled once per type argument (as C++ templates or Rust generics).

---

## 4. Members

### 4.1 Fields

A field is a variable of an object. It needs a value: **an initializer, an optional type (`T?`), or an assignment in every constructor**. Numbers, `bool` and `char` default to zero.

```nz output="no name 0 True\n"
package main;

import std.io.Console;

class User {
    public string Name = "no name";
    public int Age;
    public string? Note;
}

void Main(string[] args) {
    var u = new User();
    Console.WriteLine($"{u.Name} {u.Age} {u.Note == null}");
}
```

```nz error="needs a value"
package main;

class User {
    string Name;
}

void Main(string[] args) { }
```

**Status:** Available. A non-optional `string` field will default to `""` (Planned).
**Differs from C#:** C# gives every field a default (`null` for a string or class); NZ refuses a field that could be `null` without being optional.

### 4.2 `const`

A constant member is a compile-time value; it is static already (`static const` is an error). A constant holds an integer, `bool` or `string`.

```nz output="10 lim\n"
package main;

import std.io.Console;

class Limits {
    public const int Max = 10;
    public const string Name = "lim";
}

void Main(string[] args) {
    Console.WriteLine($"{Limits.Max} {Limits.Name}");
}
```

```nz error="`double` cannot be a constant type"
package main;

const double Pi = 3.14;

void Main(string[] args) { }
```

**Status:** Partial. `float`, `double` and `char` constants are not supported: use `static readonly`.
**Differs from C#:** C# also allows `float`, `double`, `decimal` and `char` constants.

### 4.3 `readonly` fields

A `readonly` field is assigned only by its initializer and by the constructors of its type.

```nz output="1\n"
package main;

import std.io.Console;

class Order {
    public readonly int Id;

    Order(int id) {
        Id = id;
    }
}

void Main(string[] args) {
    Console.WriteLine($"{new Order(1).Id}");
}
```

**Status:** Available.
**Differs from C#:** Same as C#. `readonly` locals do not exist (as in C#).

### 4.4 `static` fields and methods

A static member belongs to the type: `Type.Name` from outside, the bare name inside. A **mutable** static field is read and written only inside `unsafe`; `const` and `static readonly` need nothing.

```nz output="4 6\n"
package main;

import std.io.Console;

class Counter {
    static int count;
    const int Step = 2;

    public static unsafe void Hit() {
        count += Step;
    }

    public static unsafe int Count() {
        return count;
    }

    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    unsafe {
        Counter.Hit();
        Counter.Hit();
        Console.WriteLine($"{Counter.Count()} {Counter.Twice(3)}");
    }
}
```

**Status:** Partial. Static members of generic types, static properties, and static members of interfaces are not supported yet.
**Differs from C#:** a mutable static needs `unsafe` (NZ only), and a static is never reached through an object.

### 4.5 Methods

A method is a function declared in a type: `ReturnType Name(Parameters) { body }`. Inside it, `this` is the object, and the fields are reached by their bare names or as `this.Name`.

```nz output="4\n"
package main;

import std.io.Console;

class Cell {
    int n;

    Cell(int n) {
        this.n = n;
    }

    public int Get() {
        return n;
    }
}

void Main(string[] args) {
    Console.WriteLine($"{new Cell(4).Get()}");
}
```

**Status:** Available.
**Differs from C#:** a method that changes the object needs a changeable receiver (a variable, or a `ref` borrow); through an `in` borrow it is refused.

### 4.6 Method overloading

Two methods (or top-level functions) with one name and different parameters.

```nz error="duplicate member `F` in `M`"
package main;

class M {
    public int F(int x) => x;
    public string F(string s) => s;
}

void Main(string[] args) { }
```

**Status:** Later. Constructors, indexers and operators can be overloaded today.
**Differs from C#:** give the methods different names for now.

### 4.7 Expression-bodied members

`=> expr;` replaces a body that only returns `expr`. It works for functions, methods, properties, indexers, operators and accessors (`get => n;`).

```nz output="12 6\n"
package main;

import std.io.Console;

class Rect {
    public int W = 3;
    public int H = 4;
    public int Area => W * H;
    public int Half() => Area / 2;
}

void Main(string[] args) {
    var r = new Rect();
    Console.WriteLine($"{r.Area} {r.Half()}");
}
```

**Status:** Available.
**Differs from C#:** Same as C#.

### 4.8 Auto-properties

`T Name { get; set; }` stores its value like a field. The forms are `{ get; set; }`, `{ get; }`, `{ get; init; }`, and one accessor may be `private` (`{ get; private set; }`). An initializer may follow: `{ get; set; } = 2;`.

```nz output="Ann 3 1\n"
package main;

import std.io.Console;

public class Account {
    public string Owner { get; }
    public int Visits { get; private set; } = 2;
    public int Id { get; init; }

    public Account(string owner) {
        Owner = owner;
    }

    public void Visit() {
        Visits++;
    }
}

void Main(string[] args) {
    var a = new Account("Ann") { Id = 1 };
    a.Visit();
    Console.WriteLine($"{a.Owner} {a.Visits} {a.Id}");
}
```

**Status:** Available.
**Differs from C#:** a struct auto-property is a variable, so `c.P.X = 1` changes it, where C# refuses it (CS1612).

### 4.9 Computed properties

A property with accessor bodies has no storage: `get { ... }` returns the value, `set { ... }` (or `init { ... }`) receives `value`.

```nz output="68 21\n"
package main;

import std.io.Console;

class Thermometer {
    double celsius = 20;

    public double Celsius {
        get { return celsius; }
        set { celsius = value; }
    }

    public double Fahrenheit => celsius * 9 / 5 + 32;
}

void Main(string[] args) {
    var t = new Thermometer();
    double f = t.Fahrenheit;
    t.Celsius += 1;
    Console.WriteLine($"{f} {t.Celsius}");
}
```

**Status:** Partial. A computed property in an object initializer, in a property pattern, or marked `required`, the value of an assignment through it inside a larger expression, and accessor bodies in an interface are not supported yet.
**Differs from C#:** Same as C# where supported.

### 4.10 The `field` keyword

C# 14 lets an accessor body use `field`, the hidden storage of the property.

```nz error="unknown name `field`"
package main;

class C {
    public int P { get => field; set => field = value; }
}

void Main(string[] args) { }
```

**Status:** Later.
**Differs from C#:** declare the storage field yourself.

### 4.11 `required` and `init`

A `required` field or property must be set by the object initializer of every `new`. An `init` accessor is set only by an object initializer or a constructor.

```nz output="ada 3\n"
package main;

import std.io.Console;

class User {
    public required string Name;
    public int Age { get; init; }
}

void Main(string[] args) {
    var u = new User { Name = "ada", Age = 3 };
    Console.WriteLine($"{u.Name} {u.Age}");
}
```

```nz error="must set the required member"
package main;

class User {
    public required string Name;
}

void Main(string[] args) {
    var u = new User();
}
```

**Status:** Available.
**Differs from C#:** Same as C# (C# 11).

### 4.12 Indexers

`T this[P p] { get { ... } set { ... } }` lets a type be indexed like an array. The accessors need bodies, and the indexer is written `public` or `private`.

```nz output="16 0\n"
package main;

import std.io.Console;

class Grid {
    int[] cells = new int[4];

    public int this[int i] {
        get { return cells[i]; }
        set { cells[i] = value; }
    }
}

void Main(string[] args) {
    var g = new Grid();
    g[1] = 5;
    g[1] += 3;
    g[1] *= 2;
    Console.WriteLine($"{g[1]} {g[0]}");
}
```

**Status:** Partial. Indexers with several parameters and auto-implemented indexers are not supported yet; a user indexer does not make a type usable in `foreach`.
**Differs from C#:** the visibility keyword is required on an indexer.

### 4.13 Operator overloading

`public static T operator +(A a, B b)` declares an operator. The operators that can be declared are `+ - ! ~ ++ --` (unary) and `+ - * / % == != < > <= >= & | ^ << >>` (binary); `==` and `!=`, `<` and `>`, `<=` and `>=` come in pairs.

```nz output="8 False\n"
package main;

import std.io.Console;

struct Vec {
    public int X;

    public static Vec operator +(Vec a, Vec b) => Make(a.X + b.X);
    public static Vec operator *(Vec a, int k) => Make(a.X * k);
    public static bool operator ==(Vec a, Vec b) => a.X == b.X;
    public static bool operator !=(Vec a, Vec b) => a.X != b.X;

    public static Vec Make(int x) {
        var v = new Vec();
        v.X = x;
        return v;
    }
}

void Main(string[] args) {
    var a = Vec.Make(2);
    var b = Vec.Make(3);
    var c = a + b * 2;
    Console.WriteLine($"{c.X} {a == b}");
}
```

**Status:** Partial. Operators of a generic type, operators in an interface, `true`/`false` operators, and using the value of a user `++` or `--` inside an expression are not supported yet.
**Differs from C#:** the operands are lent with `in` and never moved; the result is a new value.

### 4.14 Conversion operators

`public static implicit operator double(Meters m)` and `explicit operator` declare conversions.

```nz error="operator `double` cannot be overloaded"
package main;

struct Meters {
    public double V;
    public static implicit operator double(Meters m) => m.V;
}

void Main(string[] args) { }
```

**Status:** Planned. The operand will be lent with `in` and the result will be a new value, as for the other operators.
**Differs from C#:** Same as C# once it exists.

### 4.15 Events

`public event EventHandler? Click;` declares an event. See 10.4.

**Status:** Planned.
**Differs from C#:** see 10.4.

### 4.16 Finalizers

A finalizer `~C() { ... }` is not part of NZ. An object is freed at a known point (when its owner goes out of scope), and the cleanup hook is `IDisposable.Dispose`, which runs every time a value is dropped (see 17.7).

```nz error="expected identifier, found Tilde"
package main;

class C {
    ~C() { }
}

void Main(string[] args) { }
```

**Status:** Not planned. A message pointing to `IDisposable` is Planned.
**Differs from C#:** C# runs finalizers from its garbage collector, at an unknown time; NZ has no garbage collector.

---

## 5. Constructors and object creation

### 5.1 Constructors

A constructor is a member named after its type, with no return type. A type may declare several, and `new` picks the one whose parameters take the arguments. A type that declares no constructor has the implicit `new T()`.

```nz output="Ada 36\nnobody 0\n"
package main;

import std.io.Console;

class Person {
    public string Name;
    public int Age;

    Person(string name, int age) {
        Name = name;
        Age = age;
    }

    Person() : this("nobody", 0) { }
}

void Main(string[] args) {
    var a = new Person("Ada", 36);
    var b = new Person();
    Console.WriteLine($"{a.Name} {a.Age}");
    Console.WriteLine($"{b.Name} {b.Age}");
}
```

**Status:** Available.
**Differs from C#:** a field without a default must be assigned on every path of every constructor that does not chain (C# would leave it `null`).

### 5.2 Chaining with `: this(...)`

`: this(...)` runs another constructor of the same type first (see the example in 5.1). Its arguments may use the parameters and constants, but no member of the object.

**Status:** Available.
**Differs from C#:** Same as C#.

### 5.3 Chaining with `: base(...)`

`: base(...)` runs a constructor of the base class. It comes with inheritance (section 9).

```text
class Puppy : Dog {
    public Puppy(string name) : base(name) { }
}
```

**Status:** Planned.
**Differs from C#:** Same as C# once inheritance exists.

### 5.4 Primary constructors

The parameters follow the type's name, and the field initializers read them. Every other constructor of the type chains to it with `: this(...)`.

```nz output="sum 3\n"
package main;

import std.io.Console;

class Labelled(string label, int a, int b) {
    public string Label = label;
    public int Total = a + b;
}

void Main(string[] args) {
    var l = new Labelled("sum", 1, 2);
    Console.WriteLine($"{l.Label} {l.Total}");
}
```

**Status:** Partial. Using a primary-constructor parameter in a method body is not supported yet: store it in a field.
**Differs from C#:** Same as C# 12 where supported.

### 5.5 Object initializers

`new T(args) { A = a, B = b }` sets fields and settable properties after the constructor ran, in the order written.

```nz output="3 4\n"
package main;

import std.io.Console;

class Point {
    public int X;
    public int Y;
}

void Main(string[] args) {
    var p = new Point { X = 3, Y = 4 };
    Console.WriteLine($"{p.X} {p.Y}");
}
```

**Status:** Available.
**Differs from C#:** Same as C#. The order at creation is: arguments, field initializers, constructor bodies, object initializer.

### 5.6 Target-typed `new()`

`Point p = new();` takes the type from the target.

**Status:** Later.
**Differs from C#:** write the type: `var p = new Point();`.

### 5.7 Static constructors

`static C() { ... }` runs once to set up the statics of a type.

```nz error="`static` is not supported here"
package main;

class C {
    static C() { }
}

void Main(string[] args) { }
```

**Status:** Planned. Static constructors and computed static initializers will run **before `Main`**, in dependency order, and owned statics are dropped at exit in reverse order.
**Differs from C#:** C# runs a static constructor lazily, on first use; NZ will run it eagerly, so its side effects happen at a different time.

---

## 6. Parameters and arguments

### 6.1 Value parameters

A parameter without a modifier receives a copy of a value type, and **takes ownership** of an owned value (the caller's variable is moved and cannot be used afterwards).

```nz error="use of moved value `b`"
package main;

class Box {
    public int V;
}

void Take(Box b) { }

void Main(string[] args) {
    var b = new Box();
    Take(b);
    int v = b.V;
}
```

**Status:** Available.
**Differs from C#:** passing an object moves it; in C# both sides keep a reference.

### 6.2 `in` and `ref` parameters

`in` lends a value for reading during the call; `ref` lends it to be changed. The caller keeps ownership. **No keyword is written at the call**: `Bump(c)`, not `Bump(ref c)`.

```nz output="0\n1\n"
package main;

import std.io.Console;

class Counter {
    public int Count;
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

**Status:** Available.
**Differs from C#:** C# writes `ref` (and optionally `in`) at the call; NZ does not accept a keyword there. `in` is NZ's borrow of an owned value (see [Ownership](Ownership.md)).

### 6.3 `out` parameters

```text
bool ok = int.TryParse(s, out int n);
void Split(string s, out string head, out string tail) { ... }
```

**Status:** Planned, together with `out`-style overloads of the standard library (`int.TryParse(s, out int n)`, `d.TryGetValue(k, out var v)`).
**Differs from C#:** today a tuple result or a `T?` result does the job: `int? n = int.TryParse(s);`.

### 6.4 `params`

`params T[] xs` or `params Span<T> xs` as the last parameter takes any number of arguments.

```nz output="6 9\n"
package main;

import std.io.Console;

int Sum(params int[] xs) {
    int s = 0;
    foreach (var x in xs) { s += x; }
    return s;
}

int SumSpan(params Span<int> xs) {
    int s = 0;
    foreach (var x in xs) { s += x; }
    return s;
}

void Main(string[] args) {
    Console.WriteLine($"{Sum(1, 2, 3)} {SumSpan(4, 5)}");
}
```

**Status:** Available.
**Differs from C#:** Same as C# 13. `params Span<T>` uses a temporary array freed at the end of the statement.

### 6.5 Default values and named arguments

A parameter may have a default value; an argument may name its parameter. Arguments are evaluated in the order written.

```nz output="Hello Ada x1\nHello Bob x2\n"
package main;

import std.io.Console;

string Greet(string name, string greeting = "Hello", int times = 1) => $"{greeting} {name} x{times}";

void Main(string[] args) {
    Console.WriteLine(Greet("Ada"));
    Console.WriteLine(Greet(times: 2, name: "Bob"));
}
```

**Status:** Partial. A struct or class parameter has no default yet, and named arguments on standard-library members and on `Abort`/`Assert` are not supported yet.
**Differs from C#:** Same as C# where supported.

### 6.6 `ref readonly` and `scoped` parameters

**Status:** Later.
**Differs from C#:** not accepted. NZ's `in` is already a read-only borrow that cannot escape the call.

### 6.7 `ref` returns and `ref` locals

`ref T Peek()` returns a reference to an element; `ref var x = ref a[i];` names one. They will follow C#, with NZ's loan rule: the reference must end before the collection or its owner can change.

```text
ref Token Peek() => ref tokens[pos];
ref readonly Token Current() => ref tokens[pos];
ref var t = ref tokens[0];
```

**Status:** Planned.
**Differs from C#:** the loan rule is NZ's; today `var t = list[i];` on a class element is already a borrowed, read-only alias.

### 6.8 Extension methods

A top-level function whose first parameter is marked `this` can be called as a method of that type. On an owned type the receiver is `this in T` (read) or `this ref T` (change).

```nz output="10 7 2\n"
package main;

import std.io.Console;
import std.collections.List;

class Counter {
    public int N;
}

int Double(this int x) => x * 2;

int Total(this in List<int> xs) {
    int s = 0;
    foreach (var x in xs) { s += x; }
    return s;
}

void Bump(this ref Counter c) {
    c.N++;
}

void Main(string[] args) {
    var l = new List<int>();
    l.Add(3);
    l.Add(4);
    var c = new Counter();
    c.Bump();
    c.Bump();
    Console.WriteLine($"{5.Double()} {l.Total()} {c.N}");
}
```

**Status:** Partial. `x?.F()` with an extension, extensions on type names, and two extensions with one name are not supported yet.
**Differs from C#:** C# declares extension methods inside a `static class`; NZ declares them at the top level.

### 6.9 Extension blocks

C# 14's `extension(T x) { ... }` block, with extension properties.

**Status:** Later.
**Differs from C#:** use extension methods (6.8).

---

## 7. Generics

### 7.1 Type parameters, generic methods and inference

A type or a function takes type parameters in `<...>`. A generic call infers the type arguments from its arguments; when nothing can be inferred, write them.

```nz output="1.5 5 z y\n"
package main;

import std.io.Console;

struct Box<T> {
    public T V;

    Box(T v) {
        V = v;
    }
}

T Id<T>(T x) => x;

(B, A) Swap<A, B>(A a, B b) => (b, a);

void Main(string[] args) {
    var b = new Box<double>(1.5);
    var s = Swap(1, "y");
    Console.WriteLine($"{b.V} {Id(5)} {Id<string>("z")} {s.Item1}");
}
```

**Status:** Available.
**Differs from C#:** generic code is compiled once per type argument. `F<int>` without a call is an error.

### 7.2 Constraints: `where`

`where T : IComparable<T>`, `where T : class`, `where T : struct`, `where T : new()` limit what a type argument can be, and let generic code call the members the constraint promises.

```text
T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) >= 0 ? a : b;
T Make<T>() where T : new() => new T();
```

**Status:** Planned (soon). Today `where` is reserved, and `default(T)` is refused because it needs a constraint.
**Differs from C#:** Same as C# once it lands.

### 7.3 Variance: `in` and `out` on type parameters

`interface IProducer<out T>` and `delegate void Sink<in T>(T x)`.

**Status:** Planned. Only read-only views will convert (an `IEnumerable<Dog>` to an `IEnumerable<Animal>`); mutable arrays and lists will not.
**Differs from C#:** C# also converts arrays covariantly (with a run-time check); NZ will not.

### 7.4 Static members of generic types

```nz error="not supported yet: a static member of a generic type"
package main;

class Box<T> {
    public static int Count() => 0;
}

void Main(string[] args) { }
```

**Status:** Later.
**Differs from C#:** use a top-level function.

### 7.5 Generic local functions

**Status:** Later.
**Differs from C#:** a local function may use the type parameters of the function around it, but not declare its own.

---

## 8. Interfaces

### 8.1 Declaring an interface

Members of an interface are methods without bodies (and property signatures). An interface may be generic. See the example in 3.7.

**Status:** Available.
**Differs from C#:** Same as C# for what is accepted.

### 8.2 Implementing an interface

```text
interface IShape {
    int Area();
}

class Square : IShape {
    public int Side = 3;
    public int Area() => Side * Side;
}
```

An interface value will own its object (moving it in), and a borrowed interface value (`in IShape`) will lend it. A struct will reach interface members through generic constraints only, never in an interface value.

**Status:** Planned (soon).
**Differs from C#:** the ownership of an interface value is NZ's; `==` on interface values will be refused.

### 8.3 Default interface members

`string Describe() => Speak();` inside an interface gives a body that implementing types inherit.

```nz error="default interface members are not supported yet"
package main;

interface IShape {
    string Name() => "shape";
}

void Main(string[] args) { }
```

**Status:** Planned (soon).
**Differs from C#:** Same as C# once it lands.

### 8.4 Interface inheritance

`interface IB : IA { ... }`.

**Status:** Planned (soon).
**Differs from C#:** Same as C#.

### 8.5 `readonly` interface members

`readonly int Size();` in an interface: only `readonly` members can be called through an `in` interface value, and an implementation must not change the object.

**Status:** Planned (soon). NZ only.
**Differs from C#:** C# has no `readonly` interface members; NZ needs the marker to know which calls a read-only borrow allows.

### 8.6 Explicit interface implementation

`int IShape.Area() => ...;` makes a member reachable only through the interface.

**Status:** Planned.
**Differs from C#:** Same as C#.

### 8.7 Static and static abstract interface members

**Status:** Later.
**Differs from C#:** not accepted.

---

## 9. Inheritance

### 9.1 Base classes

`class Dog : Animal, IPet` declares one base class and any number of interfaces. Structs do not inherit; they implement interfaces.

```text
class Animal {
    public virtual string Speak() => "...";
}

class Dog : Animal {
    public override string Speak() => "Woof";
}
```

**Status:** Planned.
**Differs from C#:** there is no common root class (no `object` base with members for every class); an owner of a `Dog` moves into an owner of an `Animal`.

### 9.2 `virtual`, `override`, `abstract`, `sealed`, `new`

A `virtual` member may be replaced in a derived class with `override`; an `abstract` member has no body and makes its class abstract; `sealed` stops further derivation; `new` hides a base member.

**Status:** Planned.
**Differs from C#:** Same as C#. A missing `new` will be a warning, as in C#.

### 9.3 `base` access

`base.M()` calls the base class's version of a member.

**Status:** Planned.
**Differs from C#:** Same as C#.

### 9.4 Casts between classes and `as`

`animal as Dog` gives a `Dog?`; `(Dog)animal` moves the object into a `Dog` and throws `InvalidCastException` when it is not one.

**Status:** Planned (after inheritance and exceptions).
**Differs from C#:** the operand of a successful cast is moved.

### 9.5 `object`

A minimal `object` that can only be borrowed: `==`, `ReferenceEquals` and `GetType()`; no boxing of values.

```nz error="unknown type `object`"
package main;

void Main(string[] args) {
    object o = 1;
}
```

**Status:** Planned (minimal). Boxing of values is not planned.
**Differs from C#:** an `int` never converts to `object`.

---

## 10. Lambdas, delegates and events

### 10.1 Local functions

A function declared inside a block. It is visible only in that block and may be called before or after its declaration. It reads the variables around it.

```nz output="15\n"
package main;

import std.io.Console;

void Main(string[] args) {
    int k = 3;
    int times(int x) {
        return x * k;
    }
    Console.WriteLine($"{times(5)}");
}
```

**Status:** Available. A local function cannot change a variable it captures yet, and cannot be generic.
**Differs from C#:** an uppercase name gives a warning (a local function is never visible outside its block).

### 10.2 Lambdas, `Func` and `Action`

```text
Func<int, int> square = x => x * x;
Action<string> log = s => Console.WriteLine(s);
int i = list.FindIndex(x => x > 5);
```

A lambda passed to a call borrows what it captures for that call, and may change it. A lambda that is stored or returned **copies or moves** what it captures when it is made, and its captures are read-only.

**Status:** Planned.
**Differs from C#:** a stored lambda copies its captures (C# shares the variable); a program where that would print something different from C# will be refused.

### 10.3 `delegate` types

```text
delegate int Op(int a, int b);
Op add = (a, b) => a + b;
```

**Status:** Planned. Delegate values will be copied freely; `+=`, `-=` and `==` will follow C#.
**Differs from C#:** there will be no `Delegate` or `MulticastDelegate` type to name.

### 10.4 Events

```text
class Button {
    public event EventHandler? Click;
    public void Press() => Click?.Invoke(this, EventArgs.Empty);
}

button.Click += (sender, e) => Console.WriteLine("pressed");
```

**Status:** Planned.
**Differs from C#:** a handler stored on an owned object cannot keep a borrow of it; shared state goes through `Shared<T>` (see 17.3).

### 10.5 Function pointers: `delegate*`

**Status:** Later.
**Differs from C#:** refused with "not supported yet: function pointers".

---

## 11. Attributes

### 11.1 Attribute syntax

Attributes are written in square brackets in front of a declaration: a top-level function, a type, a field, a property, a method or an interface member. Arguments are positional or named (`name: value` for a constructor parameter, `Name = value` for a field or property), and each argument is a literal or a constant expression.

```nz warning="unknown attribute"
package main;

[Route("/users", method: "GET"), Owner("web")]
[Cache(60 * 60)]
class UsersController {
    [Column("id")]
    int Id;
}

void Main(string[] args) { }
```

**Status:** Available. An attribute the compiler does not know gives a warning ("unknown attribute") and has no effect yet.
**Differs from C#:** attributes on parameters, local functions and enum members are not accepted yet.

### 11.2 `[Test]`

`[Test]` marks a top-level `void name()` function as a test; `nzc test` runs each test in its own process.

```nz
package main;

int Add(int a, int b) => a + b;

[Test]
void AddsNumbers() {
    Assert(Add(2, 3) == 5, "2 + 3");
}

void Main(string[] args) { }
```

**Status:** Available. NZ only.
**Differs from C#:** C# has no built-in test attribute (test frameworks bring their own).

### 11.3 Interop attributes

`[DllImport]`, `[UnmanagedCallersOnly]`, `[StructLayout]` and `[FieldOffset]` from `std.interop`. See section 16.

**Status:** Available (`[FieldOffset]` is Later).
**Differs from C#:** see section 16.

### 11.4 `[Flags]`

```nz output="3\n"
package main;

import std.io.Console;

enum Perm { None = 0, Read = 1, Write = 2 }

void Main(string[] args) {
    var p = Perm.Read | Perm.Write;
    Console.WriteLine($"{(int)p}");
}
```

**Status:** Planned. Today `[Flags]` is an unknown attribute: a combined value prints as its number, not as `Read, Write`.
**Differs from C#:** see Status.

### 11.5 `[Obsolete]` and other .NET attributes

`[Obsolete("use G")]` will warn at every use. A fixed list of other .NET attributes (`[Serializable]`, `[DebuggerDisplay]`, `[MethodImpl]` and others) will be accepted and documented as having no effect.

**Status:** Planned. Today they are unknown attributes (a warning).
**Differs from C#:** Same as C# for `[Obsolete]`.

### 11.6 User-defined attributes

```text
[AttributeUsage(AttributeTargets.Class)]
class TagAttribute : Attribute {
    public TagAttribute(string name) { ... }
}

[Tag("model")]
class User { }
```

**Status:** Partial. The use `[Tag("model")]` parses and is kept (with a warning); declaring the attribute class needs inheritance, and reading it needs reflection. Both are Planned, following C# 14 (`[AttributeUsage]` included).
**Differs from C#:** Same as C# once it lands.

### 11.7 Attribute targets: `[return: ...]`, `[assembly: ...]`, `[module: ...]`

```nz error="attribute targets are not supported yet"
package main;

[return: Tag]
string Name() { return "n"; }

void Main(string[] args) { }
```

**Status:** Planned. `assembly:` will mean the program and `module:` the package.
**Differs from C#:** today every target is refused.

---

## 12. Preprocessor directives

### 12.1 `#if`, `#elif`, `#else`, `#endif`, `#define`, `#undef`

Conditional compilation. The predefined symbols will be `DEBUG` and `TRACE` (or `RELEASE` and `TRACE` with `--release`), and `NZ_NO_RUNTIME` with `--no-runtime`.

```nz error="unexpected character"
package main;

#define FAST
#if FAST
void Main(string[] args) { }
#endif
```

**Status:** Planned.
**Differs from C#:** the symbol `NZ_NO_RUNTIME` is NZ only.

### 12.2 `#region` and `#endregion`

**Status:** Planned.
**Differs from C#:** Same as C# (no effect on the program).

### 12.3 `#pragma warning`

`#pragma warning disable CS0618` turns warnings off and on.

**Status:** Planned. It will take NZ's own codes, and a fixed list of the C# codes that NZ mirrors.
**Differs from C#:** only the mirrored C# codes are known.

### 12.4 `#nullable`

**Status:** Planned. `#nullable enable` and `#nullable restore` will be accepted silently (NZ's null safety is always on); `#nullable disable` will be accepted with a warning that it has no effect.
**Differs from C#:** null safety cannot be turned off.

### 12.5 `#error`, `#warning`, `#line`

**Status:** Planned (`#error` and `#warning`); Later (`#line`).
**Differs from C#:** Same as C# once they land.

---

## 13. Async code and iterators

### 13.1 `async`, `await`, `Task` and `Task<T>`

```text
async Task<string> LoadAsync(string path) {
    string text = await File.ReadAllTextAsync(path);
    return text.Trim();
}
```

All async methods will run on one main loop, in order; only `Task.Run` will use worker threads. Tasks still running when `Main` returns will be cancelled, running their drops.

```nz error="`async` is reserved for a future feature"
package main;

async Task<int> F() { return 1; }

void Main(string[] args) { }
```

**Status:** Planned.
**Differs from C#:** the threading model is fixed (one main loop); a value moved to a worker thread must be safe to send, checked by the compiler.

### 13.2 `async void`, `ValueTask`, `await using`

**Status:** Planned. `async void` will be allowed only for event handlers; `ValueTask` will work as `Task`; `await using` with `IAsyncDisposable`.
**Differs from C#:** `async void` elsewhere will be refused.

### 13.3 Iterators: `yield return`

```text
IEnumerable<int> Count(int n) {
    for (int i = 0; i < n; i++) {
        yield return i;
    }
}
```

**Status:** Planned. A `foreach` over an iterator will receive each value moved to the loop variable.
**Differs from C#:** Same as C# once it lands.

### 13.4 Async iterators: `IAsyncEnumerable<T>`, `await foreach`

**Status:** Later.
**Differs from C#:** not accepted.

### 13.5 `lock` and `Mutex<T>`

```text
var counter = new Arc<Mutex<int>>(new Mutex<int>(0));
lock (counter) { ... }
```

**Status:** Planned. NZ only: `Mutex<T>` owns the data it protects, and `lock (m)` gives write access to it inside the block. `lock (this)` on an arbitrary object will be refused.
**Differs from C#:** C# locks any object; NZ locks a `Mutex<T>`.

---

## 14. Exceptions

### 14.1 Exception types

Exception classes with the .NET names (`Exception`, `InvalidOperationException`, `FormatException`, `FileNotFoundException` and others), and user exception classes derived from `Exception`.

**Status:** Planned. User exception classes need inheritance (section 9).
**Differs from C#:** an exception object has one owner: `throw` moves it and `catch` owns it.

### 14.2 `throw`, `try`, `catch`, `finally`

```text
try {
    string text = File.ReadAllText(path);
} catch (FileNotFoundException e) when (e.Message.Length > 0) {
    Console.WriteLine(e.Message);
} finally {
    Console.WriteLine("done");
}
```

The statements are described in [Part 1](Syntax-Reference.md). An exception that nothing catches stops the program with `nz: <Type>: <message>` and exit status 1, without running `finally`, as .NET does.

```nz error="expected Semi, found LBrace"
package main;

void Main(string[] args) {
    try { } finally { }
}
```

**Status:** Planned (the work has started).
**Differs from C#:** run-time checks (index out of range, overflow, a failed `Assert`) will throw the .NET exception; some failures stay stops that cannot be caught: `Abort`, `Environment.Exit`, stack overflow, out of memory.

### 14.3 Failures today

Today a failure stops the program (`Abort`, `Assert`, run-time checks), and an expected failure is a value: `Result<T, E>` with the `?` operator, and the `Try*` members of the standard library. These stay when exceptions arrive. See [Error Handling](Error-Handling.md).

**Status:** Available.
**Differs from C#:** `Result<T, E>` and `?` are NZ only.

---

## 15. Reflection

### 15.1 `nameof`

`nameof(x)` is the last name of `x` as a `string` constant. See Part 1.

```nz output="Dog\n"
package main;

import std.io.Console;

class Dog { }

void Main(string[] args) {
    Console.WriteLine(nameof(Dog));
}
```

**Status:** Available.
**Differs from C#:** Same as C#.

### 15.2 `typeof`, `GetType()` and `Type`

`typeof(Dog)` and `d.GetType()` give a `Type`: its `Name`, `FullName`, and a list of its members (names, types and attributes, no values). `Type.Name` follows .NET's spelling (`Int32`, ``List`1``), and `FullName` uses the package (`zoo.Dog`).

```nz error="`Dog` has no member `GetType`"
package main;

class Dog { }

void Main(string[] args) {
    var d = new Dog();
    string n = d.GetType().Name;
}
```

**Status:** Planned. Only types a program asks about carry the extra data. `GetValue` and `Invoke` are not planned for now.
**Differs from C#:** a smaller `System.Reflection`.

### 15.3 Reading attributes

`GetCustomAttribute<T>()` returns a `T?`, and `GetCustomAttributes()` an `Attribute[]`.

**Status:** Planned.
**Differs from C#:** the typed forms only (no `object[]`).

### 15.4 `dynamic`

```nz error="unknown type `dynamic`"
package main;

void Main(string[] args) {
    dynamic o = 1;
}
```

**Status:** Later.
**Differs from C#:** not accepted.

---

## 16. Unsafe code and native interop

### 16.1 `unsafe`

`unsafe { ... }`, or `unsafe` on a function, method, class or struct, turns on pointers, `stackalloc`, `fixed`, `NativeMemory`, calls to C functions and mutable statics. Types, null safety, overflow checks, bounds checks on arrays and lists, and ownership stay checked inside.

**Status:** Available.
**Differs from C#:** a call to an `extern` function always needs `unsafe`, and so does a mutable static.

### 16.2 Pointers, `&` and `stackalloc`

`T*`, `void*`, `&x`, `*p`, `p->F`, `p[i]` and pointer arithmetic. `stackalloc T[n]` with a constant length initializes a pointer local; the buffer lives until the function returns.

```nz output="7 9\n"
package main;

import std.io.Console;

void Main(string[] args) {
    unsafe {
        int x = 5;
        int* p = &x;
        *p = 7;
        int* buf = stackalloc int[4];
        buf[2] = 9;
        Console.WriteLine($"{x} {buf[2]}");
    }
}
```

**Status:** Available. A variable `stackalloc` length, and `stackalloc` into a `Span<T>`, are not supported yet.
**Differs from C#:** a pointer cannot point to an optional or a tuple; `char` is 4 bytes.

### 16.3 `fixed`

`fixed (int* p = arr) { ... }` holds an array or a field in place for the block.

```nz output="4\n"
package main;

import std.io.Console;

void Main(string[] args) {
    int[] arr = new int[3];
    unsafe {
        fixed (int* p = arr) {
            p[1] = 4;
        }
    }
    Console.WriteLine($"{arr[1]}");
}
```

**Status:** Available. `fixed` on a `string` is not supported yet.
**Differs from C#:** NZ strings are UTF-8, not UTF-16.

### 16.4 `NativeMemory`

`NativeMemory.Alloc`, `AllocZeroed`, `Realloc`, `Free`, `Copy` and `Clear` from `std.interop`. The runtime counts the blocks, and `nzc test` fails a test that leaks one.

```nz output="42\n"
package main;

import std.io.Console;
import std.interop.NativeMemory;

void Main(string[] args) {
    unsafe {
        int* p = (int*)NativeMemory.Alloc(16);
        p[0] = 42;
        Console.WriteLine($"{p[0]}");
        NativeMemory.Free(p);
    }
}
```

**Status:** Available.
**Differs from C#:** Same as C#'s API.

### 16.5 `[StructLayout]`

`[StructLayout(LayoutKind.Sequential, Pack = n)]` on a struct of unmanaged fields gives C's layout.

```nz output="5\n"
package main;

import std.io.Console;
import std.interop.*;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct Header {
    public byte Tag;
    public int Size;
}

void Main(string[] args) {
    unsafe {
        Console.WriteLine($"{sizeof(Header)}");
    }
}
```

**Status:** Partial. `LayoutKind.Explicit`, `[FieldOffset]` and `LayoutKind.Auto` are Later.
**Differs from C#:** `char` cannot be a field of a laid-out struct.

### 16.6 `extern` and `[DllImport]`

`[DllImport("lib")] extern int name(...);` at the top level declares a C function.

```nz output="3\n"
package main;

import std.io.Console;
import std.interop.*;

[DllImport("c")]
extern int abs(int x);

void Main(string[] args) {
    unsafe {
        Console.WriteLine($"{abs(-3)}");
    }
}
```

**Status:** Available. `CallingConvention`, `CharSet`, `[LibraryImport]` and C varargs are not supported yet.
**Differs from C#:** the function is top-level (no `static`), the library is a plain name, and every call needs `unsafe`.

### 16.7 `[UnmanagedCallersOnly]`

`[UnmanagedCallersOnly(EntryPoint = "name")]` on a top-level function exports it to C under that name. NZ code cannot call it.

```nz
package main;

import std.interop.*;

[UnmanagedCallersOnly(EntryPoint = "add_i32")]
int AddI32(int a, int b) {
    return a + b;
}

void Main(string[] args) { }
```

**Status:** Available.
**Differs from C#:** `EntryPoint` is required; no `ref` or `in` parameters.

### 16.8 `--no-runtime`

`nzc build --no-runtime` builds without NZ's runtime: numbers, `bool`, `char`, enums, structs, pointers, statics, `extern`, `NativeMemory` and `stackalloc` remain; strings, classes, arrays and lists are refused.

**Status:** Available. NZ only.
**Differs from C#:** C# has no such mode.

### 16.9 `volatile`

**Status:** Later.
**Differs from C#:** not accepted.

---

## 17. Ownership and memory (NZ only)

The rules of this section have no C# equivalent: NZ has no garbage collector. [Ownership](Ownership.md) and [Memory Model](Memory-Model.md) describe them in full.

### 17.1 One owner, moves

A class object, a list and an array have exactly one owner. Assigning or passing one without a modifier moves it (see 6.1); the old variable cannot be used until it gets a new value.

**Status:** Available. NZ only.
**Differs from C#:** in C# every variable holds a shared reference.

### 17.2 Borrowing: `in` and `ref`

`in` lends an owned value for reading and `ref` lends it to be changed, for the length of the call (see 6.2). A borrow is never stored.

**Status:** Available. NZ only (the keywords are C#'s).
**Differs from C#:** see 6.2.

### 17.3 `Shared<T>` and `WeakReference<T>`

`new Shared<T>(v)` moves `v` into a reference-counted block; copying the handle shares the object, and the last handle frees it. A `WeakReference<T>` does not keep the object alive and breaks cycles.

```nz output="5 2\n5\n"
package main;

import std.io.Console;
import std.memory.Shared;
import std.memory.WeakReference;

class Node {
    public int V = 1;
}

void Main(string[] args) {
    var a = new Shared<Node>(new Node());
    var b = a;
    b.V = 5;
    var w = new WeakReference<Node>(a);
    Console.WriteLine($"{a.V} {Shared.StrongCount(a)}");
    if (w.TryGetTarget() is Shared<Node> s) {
        Console.WriteLine($"{s.V}");
    }
}
```

**Status:** Available. NZ only.
**Differs from C#:** `WeakReference<T>` is .NET's name, but it points to a `Shared<T>` object here.

### 17.4 `Arc<T>`

`new Arc<T>(v)` is a reference-counted, read-only object that can be shared between threads.

```nz output="80 2\n"
package main;

import std.io.Console;
import std.memory.Arc;

class Config {
    public int Port = 80;
}

void Main(string[] args) {
    var a = new Arc<Config>(new Config());
    var b = a;
    Console.WriteLine($"{b.Port} {Arc.StrongCount(a)}");
}
```

**Status:** Available. NZ only.
**Differs from C#:** no equivalent.

### 17.5 Drop order

An owned value is dropped when its owner goes out of scope: locals in reverse order of declaration. When an object is dropped, the object is freed first, then its fields in declaration order. A program never sees a dropped value.

**Status:** Available. NZ only.
**Differs from C#:** C# frees objects when its garbage collector runs, in no fixed order.

### 17.6 `defer` and `using`

`defer statement;` runs the statement when the enclosing block ends. `using (var x = e) { ... }` and `using var x = e;` (see Part 1) end the life of `x` at the end of the block.

```nz output="1\nfirst\nsecond\n"
package main;

import std.io.Console;

class Res {
    public int V = 1;
}

void Main(string[] args) {
    using (var r = new Res()) {
        Console.WriteLine($"{r.V}");
    }
    {
        defer Console.WriteLine("second");
        Console.WriteLine("first");
    }
}
```

**Status:** Available (`defer`, NZ only); Partial (`using`: it accepts any value today; with `IDisposable` it will accept only an `IDisposable` value, as C# does).
**Differs from C#:** `defer` has no C# equivalent. `using (x) { }` on an existing variable is not accepted yet; when it is, the block will own `x`, so `x` cannot be used after the block (C# keeps it usable).

### 17.7 `IDisposable` and `Dispose`

A class that implements `IDisposable` has its `Dispose()` called **every time a value of it is dropped**, so cleanup runs at a known point without a `using`. A program may call `x.Dispose()` itself on a variable it owns; the call consumes the variable.

```text
class File : IDisposable {
    public void Dispose() { /* close the handle */ }
}
```

**Status:** Planned (soon).
**Differs from C#:** C# calls `Dispose` only through `using` or by hand; NZ calls it at every drop, and an explicit call moves the value.

---

## 18. Not planned

| Construct | Why |
|---|---|
| `namespace`, `using` directives | `package` and `import` replace them (1.8). |
| Finalizers `~C()` | `IDisposable` runs at every drop (4.16). |
| Query expressions (LINQ: `from x in xs select x`) | Not needed: loops and methods cover the use. |
| Anonymous types (`new { X = 1 }`) | Tuples cover the use. |
| Boxing values into `object` | No common root type; `object` will be minimal (9.5). |

```nz error="expected identifier"
package main;

void Main(string[] args) {
    var a = new { X = 1 };
}
```

---

## 19. Index

| Construct | Section | Status |
|---|---|---|
| Source file layout | [1.1](#11-source-file-layout) | Available |
| `package` | [1.2](#12-package) | Available |
| `import` (four forms) | [1.3](#13-import) | Available |
| Folders as packages | [1.4](#14-folders-as-packages) | Available |
| Entry point `Main` | [1.5](#15-entry-point-main) | Available |
| Top-level functions | [1.6](#16-top-level-functions) | Available |
| Top-level constants and statics | [1.7](#17-top-level-constants-and-statics) | Available |
| `namespace`, `using` directives | [1.8](#18-namespace-and-using-directives) | Not planned |
| Top-level statements | [1.9](#19-top-level-statements) | Later |
| First-letter visibility rule | [2.1](#21-the-first-letter-rule-for-members) | Available (NZ only) |
| `public`, `private` | [2.2](#22-public-and-private) | Available |
| Visibility of types | [2.3](#23-visibility-of-types) | Partial |
| `internal` | [2.4](#24-internal) | Planned |
| `protected` | [2.5](#25-protected) | Planned |
| `file`, `protected internal`, `private protected` | [2.6](#26-file-protected-internal-private-protected) | Later |
| `class` | [3.1](#31-class) | Available |
| `struct` | [3.2](#32-struct) | Available |
| `record class`, `record struct` | [3.3](#33-record-class-and-record-struct) | Partial |
| Record equality, `ToString`, `with` | [3.4](#34-record-equality-tostring-and-with) | Planned |
| `enum` | [3.5](#35-enum) | Available |
| `union` | [3.6](#36-union) | Available |
| `interface` (declaration) | [3.7](#37-interface) | Partial |
| `delegate` (declaration) | [3.8](#38-delegate) | Planned |
| Type aliases `type` | [3.9](#39-type-aliases-type) | Available |
| `static class` | [3.10](#310-static-class) | Available |
| Nested types | [3.11](#311-nested-types) | Partial |
| `partial` | [3.12](#312-partial) | Later |
| `abstract`, `sealed`, `virtual`, `override`, `new` | [3.13](#313-abstract-sealed-virtual-override-new-modifier) | Planned |
| `readonly struct`, `ref struct` | [3.14](#314-readonly-struct-and-ref-struct) | Later |
| Generic types | [3.15](#315-generic-types) | Available |
| Fields | [4.1](#41-fields) | Available |
| `const` | [4.2](#42-const) | Partial |
| `readonly` fields | [4.3](#43-readonly-fields) | Available |
| `static` fields and methods | [4.4](#44-static-fields-and-methods) | Partial |
| Methods | [4.5](#45-methods) | Available |
| Method overloading | [4.6](#46-method-overloading) | Later |
| Expression-bodied members | [4.7](#47-expression-bodied-members) | Available |
| Auto-properties | [4.8](#48-auto-properties) | Available |
| Computed properties | [4.9](#49-computed-properties) | Partial |
| `field` keyword | [4.10](#410-the-field-keyword) | Later |
| `required`, `init` | [4.11](#411-required-and-init) | Available |
| Indexers | [4.12](#412-indexers) | Partial |
| Operator overloading | [4.13](#413-operator-overloading) | Partial |
| Conversion operators | [4.14](#414-conversion-operators) | Planned |
| Events (member) | [4.15](#415-events) | Planned |
| Finalizers | [4.16](#416-finalizers) | Not planned |
| Constructors | [5.1](#51-constructors) | Available |
| `: this(...)` | [5.2](#52-chaining-with--this) | Available |
| `: base(...)` | [5.3](#53-chaining-with--base) | Planned |
| Primary constructors | [5.4](#54-primary-constructors) | Partial |
| Object initializers | [5.5](#55-object-initializers) | Available |
| Target-typed `new()` | [5.6](#56-target-typed-new) | Later |
| Static constructors | [5.7](#57-static-constructors) | Planned |
| Value parameters (moves) | [6.1](#61-value-parameters) | Available |
| `in`, `ref` parameters | [6.2](#62-in-and-ref-parameters) | Available |
| `out` parameters | [6.3](#63-out-parameters) | Planned |
| `params` | [6.4](#64-params) | Available |
| Default values, named arguments | [6.5](#65-default-values-and-named-arguments) | Partial |
| `ref readonly`, `scoped` parameters | [6.6](#66-ref-readonly-and-scoped-parameters) | Later |
| `ref` returns, `ref` locals | [6.7](#67-ref-returns-and-ref-locals) | Planned |
| Extension methods | [6.8](#68-extension-methods) | Partial |
| Extension blocks | [6.9](#69-extension-blocks) | Later |
| Type parameters, generic methods, inference | [7.1](#71-type-parameters-generic-methods-and-inference) | Available |
| Constraints `where` | [7.2](#72-constraints-where) | Planned |
| Variance `in`/`out` | [7.3](#73-variance-in-and-out-on-type-parameters) | Planned |
| Static members of generic types | [7.4](#74-static-members-of-generic-types) | Later |
| Generic local functions | [7.5](#75-generic-local-functions) | Later |
| Interface declaration | [8.1](#81-declaring-an-interface) | Available |
| Implementing an interface | [8.2](#82-implementing-an-interface) | Planned |
| Default interface members | [8.3](#83-default-interface-members) | Planned |
| Interface inheritance | [8.4](#84-interface-inheritance) | Planned |
| `readonly` interface members | [8.5](#85-readonly-interface-members) | Planned (NZ only) |
| Explicit interface implementation | [8.6](#86-explicit-interface-implementation) | Planned |
| Static interface members | [8.7](#87-static-and-static-abstract-interface-members) | Later |
| Base classes | [9.1](#91-base-classes) | Planned |
| `virtual`, `override`, `abstract`, `sealed`, `new` | [9.2](#92-virtual-override-abstract-sealed-new) | Planned |
| `base` access | [9.3](#93-base-access) | Planned |
| Class casts, `as` | [9.4](#94-casts-between-classes-and-as) | Planned |
| `object` | [9.5](#95-object) | Planned |
| Local functions | [10.1](#101-local-functions) | Available |
| Lambdas, `Func`, `Action` | [10.2](#102-lambdas-func-and-action) | Planned |
| `delegate` types | [10.3](#103-delegate-types) | Planned |
| Events | [10.4](#104-events) | Planned |
| `delegate*` | [10.5](#105-function-pointers-delegate) | Later |
| Attribute syntax | [11.1](#111-attribute-syntax) | Available |
| `[Test]` | [11.2](#112-test) | Available (NZ only) |
| Interop attributes | [11.3](#113-interop-attributes) | Available |
| `[Flags]` | [11.4](#114-flags) | Planned |
| `[Obsolete]` and other .NET attributes | [11.5](#115-obsolete-and-other-net-attributes) | Planned |
| User-defined attributes | [11.6](#116-user-defined-attributes) | Partial |
| Attribute targets | [11.7](#117-attribute-targets-return--assembly--module-) | Planned |
| `#if`, `#elif`, `#else`, `#endif`, `#define`, `#undef` | [12.1](#121-if-elif-else-endif-define-undef) | Planned |
| `#region` | [12.2](#122-region-and-endregion) | Planned |
| `#pragma warning` | [12.3](#123-pragma-warning) | Planned |
| `#nullable` | [12.4](#124-nullable) | Planned |
| `#error`, `#warning` | [12.5](#125-error-warning-line) | Planned |
| `#line` | [12.5](#125-error-warning-line) | Later |
| `async`, `await`, `Task` | [13.1](#131-async-await-task-and-taskt) | Planned |
| `async void`, `ValueTask`, `await using` | [13.2](#132-async-void-valuetask-await-using) | Planned |
| Iterators `yield return` | [13.3](#133-iterators-yield-return) | Planned |
| Async iterators | [13.4](#134-async-iterators-iasyncenumerablet-await-foreach) | Later |
| `lock` and `Mutex<T>` | [13.5](#135-lock-and-mutext) | Planned (NZ only) |
| Exception types | [14.1](#141-exception-types) | Planned |
| `throw`, `try`, `catch`, `finally` | [14.2](#142-throw-try-catch-finally) | Planned |
| `Abort`, `Assert`, `Result` | [14.3](#143-failures-today) | Available |
| `nameof` | [15.1](#151-nameof) | Available |
| `typeof`, `GetType()`, `Type` | [15.2](#152-typeof-gettype-and-type) | Planned |
| Reading attributes | [15.3](#153-reading-attributes) | Planned |
| `dynamic` | [15.4](#154-dynamic) | Later |
| `unsafe` | [16.1](#161-unsafe) | Available |
| Pointers, `&`, `stackalloc` | [16.2](#162-pointers--and-stackalloc) | Available |
| `fixed` | [16.3](#163-fixed) | Available |
| `NativeMemory` | [16.4](#164-nativememory) | Available |
| `[StructLayout]` | [16.5](#165-structlayout) | Partial |
| `extern`, `[DllImport]` | [16.6](#166-extern-and-dllimport) | Available |
| `[UnmanagedCallersOnly]` | [16.7](#167-unmanagedcallersonly) | Available |
| `--no-runtime` | [16.8](#168---no-runtime) | Available (NZ only) |
| `volatile` | [16.9](#169-volatile) | Later |
| One owner, moves | [17.1](#171-one-owner-moves) | Available (NZ only) |
| Borrowing `in`, `ref` | [17.2](#172-borrowing-in-and-ref) | Available (NZ only) |
| `Shared<T>`, `WeakReference<T>` | [17.3](#173-sharedt-and-weakreferencet) | Available (NZ only) |
| `Arc<T>` | [17.4](#174-arct) | Available (NZ only) |
| Drop order | [17.5](#175-drop-order) | Available (NZ only) |
| `defer` | [17.6](#176-defer-and-using) | Available (NZ only) |
| `using` | [17.6](#176-defer-and-using) | Partial |
| `IDisposable`, `Dispose` | [17.7](#177-idisposable-and-dispose) | Planned |
| LINQ query expressions | [18](#18-not-planned) | Not planned |
| Anonymous types | [18](#18-not-planned) | Not planned |
| Boxing into `object` | [18](#18-not-planned) | Not planned |
