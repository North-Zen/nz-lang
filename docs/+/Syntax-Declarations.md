# Syntax Reference, Part 2: Declarations

This page lists how a program is declared: source files and packages, types, members, parameters, generics, interfaces, inheritance, delegates and events, attributes, preprocessor directives, async code, exceptions, reflection, unsafe code, and the ownership rules that exist only in NZ. [Part 1](Syntax-Reference.md) covers the rest: lexical structure, literals, operators and precedence, built-in types, variables, expressions, statements and patterns.

## How to read this page

Every construct has a short entry: the syntax, a sentence or two on what it does, and an example.

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
18. [Index](#18-index)

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

NZ files start with `package` and `import`.

### 1.2 `package`

`package name;` is the first line of every file. A package is a **folder**: all `.nz` files directly inside one folder belong to one package, and a subfolder is another package. A dotted name such as `app.util` stands for the folder `app/util/`. All files of a package share one set of top-level names.

The folder decides the package, and `namespace` is not accepted (see 1.8).

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

NZ imports names, has a group form, and a qualified name without an import is an error.

### 1.4 Folders as packages

`nzc run app` runs the folder `app` as the **entry package**. Its `import geo.Point;` loads the sibling folder `geo`. Only `public` declarations can be imported by another package.

```text
proj/
  app/main.nz     package app;  import geo.Point;  void Main(...) { ... }
  geo/point.nz    package geo;  public class Point { public int X = 7; }
```

NZ has no project file yet; the folders decide.

### 1.5 Entry point `Main`

The entry package declares one top-level `Main`, in one of four forms: `void Main()`, `void Main(string[] args)`, `int Main()`, `int Main(string[] args)`. An `int` result is the exit status. `args` holds the arguments without the program name.

```nz output="hi\n"
package main;

import std.io.Console;

int Main() {
    Console.WriteLine("hi");
    return 0;
}
```

`Main` is a top-level function, not a `static` method of a class. `async Task Main` is not available (see 13).

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

Top-level functions are the normal case and take no `static`.

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

Top-level fields and constants are allowed.

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

A member without a keyword is private; the letter of its name decides nothing.

### 2.2 `public` and `private`

The keywords may be written. An explicit keyword wins over the letter, but `public` on a name that starts with a lowercase letter is an error, and `private` on an uppercase name gives a warning.

```nz error="a `public` name must start with an uppercase letter"
package main;

public class C {
    public int x;
}

void Main(string[] args) { }
```

The naming checks are part of NZ.

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

A constructor cannot be more visible than its type, so the example above is refused.

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

An object has one owner and is moved, not shared by reference; there is no garbage collector.

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

A struct that owns an object moves instead of being copied (a copy would give the object two owners).

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

A record class has one owner like any class.

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

`[Flags]` and `HasFlag` do not exist yet (see 11.4); `Enum.GetValues` and the other members of `Enum` come with reflection (section 15).

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

A value that converts to two cases is refused.

### 3.7 `interface`

An interface declares members that a type promises to have; classes and structs implement it (section 8).

```nz
package main;

interface IShape {
    int Area();
    string Name();
}

void Main(string[] args) { }
```

See section 8.

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

NZ writes `type` for an alias, because `using` is the resource statement. Generic aliases are allowed.

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

Extension methods are declared as top-level functions, not inside a static class (see 6.8).

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

An `import` names top-level types only: import `Shape`, then write `Shape.Point`.

### 3.15 Generic types

Classes, structs, records, unions, interfaces and aliases take type parameters: `class Box<T> { ... }`. See section 7.

Generic code is compiled once per type argument.

---

## 4. Members

### 4.1 Fields

A field is a variable of an object. It needs a value: **an initializer, an optional type (`T?`), or an assignment in every constructor**. Numbers, `bool` and `char` default to zero and a `string` to `""`.

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

class Label { }

class User {
    Label Name;
}

void Main(string[] args) { }
```

A `string` field is `""`, and a class field that could be `null` without being optional is refused.

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

`float`, `double`, `decimal` and `char` are not accepted as constant types.

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

`readonly` locals do not exist.

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

A mutable static needs `unsafe` (NZ only), and a static is never reached through an object.

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

A method that changes the object needs a changeable receiver (a variable, or a `ref` borrow); through an `in` borrow it is refused.

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

A struct auto-property is a variable, so `c.P.X = 1` changes it.

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

The visibility keyword is required on an indexer.

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

The operands are lent with `in` and never moved; the result is a new value.

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

A field without a default must be assigned on every path of every constructor that does not chain.

### 5.2 Chaining with `: this(...)`

`: this(...)` runs another constructor of the same type first (see the example in 5.1). Its arguments may use the parameters and constants, but no member of the object.


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

The order at creation is: arguments, field initializers, constructor bodies, object initializer.

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

Passing an object moves it; the caller no longer holds it.

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

NZ does not accept a keyword at the call. `in` is NZ's borrow of an owned value (see [Ownership](Ownership.md)).

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

`params Span<T>` uses a temporary array freed at the end of the statement.

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

NZ declares extension methods at the top level.

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

Generic code is compiled once per type argument. `F<int>` without a call is an error.

### 7.2 Constraints: `where`

`where T : IComparable<T>`, `where T : class`, `where T : struct`, `where T : new()` limit what a type argument can be, and let generic code call the members the constraint promises.

A type argument is checked against the constraints at every call, `new` and written type, also when it is itself a type parameter (its own constraints must imply them). The library interface `IComparable<T>` is implemented by the integer types, `char` and `string`.

```nz output="7 pear\n"
package main;

import std.io.Console;

T Max<T>(T a, T b) where T : IComparable<T> {
    return a.CompareTo(b) >= 0 ? a : b;
}

void Main(string[] args) {
    Console.WriteLine($"{Max(3, 7)} {Max("pear", "apple")}");
}
```

Generic code is compiled once per type argument, so a call through a constraint on a class or struct is direct. `default(T)` is an error, since `T` may be a class, which has no default value. A struct given for a constraint must declare every interface member the generic code reaches; it does not take a default member of the interface. For a generic type, every method of the type counts, whether the program calls it or not.

---

## 8. Interfaces

### 8.1 Declaring an interface

Members of an interface are public instance methods, with or without a body (a default member). An interface may be generic. See the example in 3.7.

### 8.2 Implementing an interface

```nz output="9\n"
package main;

import std.io.Console;

interface IShape {
    readonly int Area();
}

class Square : IShape {
    public int Side = 3;
    public int Area() => Side * Side;
}

void Main(string[] args) {
    IShape s = new Square();
    Console.WriteLine(s.Area());
}
```

An interface value owns its object (converting a class moves it in), and a borrowed interface value (`in IShape`) lends it. A struct reaches interface members through generic constraints only, never in an interface value. `==` on interface values is refused.

### 8.3 Default interface members

`string Describe() => Speak();` inside an interface gives a body that implementing types inherit.

```nz output="shape\n"
package main;

import std.io.Console;

interface IShape {
    readonly string Name() => "shape";
}

class Dot : IShape { }

void Main(string[] args) {
    IShape d = new Dot();
    Console.WriteLine(d.Name());
}
```

Two different defaults inherited for one member must be resolved by declaring the member in the type.

### 8.4 Interface inheritance

`interface IB : IA { ... }`. A type that implements `IB` also implements `IA`. Redeclaring an inherited member in the derived interface is refused.

### 8.5 `readonly` interface members (NZ only)

`readonly int Size();` in an interface: only `readonly` members can be called through an `in` interface value, and an implementation must not change the object. NZ needs the marker to know which calls a read-only borrow allows.

---

## 9. Inheritance

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

An uppercase name gives a warning (a local function is never visible outside its block).

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

Attributes on parameters, local functions and enum members are not accepted yet.

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

`[Test]` is built in; no test framework is needed.

### 11.3 Interop attributes

`[DllImport]`, `[UnmanagedCallersOnly]`, `[StructLayout]` and `[FieldOffset]` from `std.interop`. See section 16.

See section 16.

### 11.6 User-defined attributes

```text
[AttributeUsage(AttributeTargets.Class)]
class TagAttribute : Attribute {
    public TagAttribute(string name) { ... }
}

[Tag("model")]
class User { }
```

---

## 12. Preprocessor directives

---

## 13. Async code and iterators

---

## 14. Exceptions

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

The statements are described in [Part 1](Syntax-Reference.md). An exception that nothing catches stops the program with `nz: <Type>: <message>` and exit status 1, without running `finally`.

```nz output="done\n"
package main;
import std.io.Console;

void Main(string[] args) {
    try { } finally { Console.WriteLine("done"); }
}
```

Some failures stop the program and cannot be caught: `Abort`, `Environment.Exit`, stack overflow, out of memory.

### 14.3 Failures today

Today a failure stops the program (`Abort`, `Assert`, run-time checks), and an expected failure is a value: `Result<T, E>` with the `?` operator, and the `Try*` members of the standard library. These stay when exceptions arrive. See [Error Handling](Error-Handling.md).

`Result<T, E>` and `?` are NZ only.

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


---

## 16. Unsafe code and native interop

### 16.1 `unsafe`

`unsafe { ... }`, or `unsafe` on a function, method, class or struct, turns on pointers, `stackalloc`, `fixed`, `NativeMemory`, calls to C functions and mutable statics. Types, null safety, overflow checks, bounds checks on arrays and lists, and ownership stay checked inside.

A call to an `extern` function always needs `unsafe`, and so does a mutable static.

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

A pointer cannot point to an optional or a tuple; `char` is 4 bytes.

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

NZ strings are UTF-8.

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

`char` cannot be a field of a laid-out struct.

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

The function is top-level (no `static`), the library is a plain name, and every call needs `unsafe`.

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

`EntryPoint` is required; no `ref` or `in` parameters.

### 16.8 `--no-runtime`

`nzc build --no-runtime` builds without NZ's runtime: numbers, `bool`, `char`, enums, structs, pointers, statics, `extern`, `NativeMemory` and `stackalloc` remain; strings, classes, arrays and lists are refused.

---

## 17. Ownership and memory (NZ only)

NZ has no garbage collector. [Ownership](Ownership.md) and [Memory Model](Memory-Model.md) describe them in full.

### 17.1 One owner, moves

A class object, a list and an array have exactly one owner. Assigning or passing one without a modifier moves it (see 6.1); the old variable cannot be used until it gets a new value.

### 17.2 Borrowing: `in` and `ref`

`in` lends an owned value for reading and `ref` lends it to be changed, for the length of the call (see 6.2). A borrow is never stored.

See 6.2.

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

`WeakReference<T>` points to a `Shared<T>` object.

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

### 17.5 Drop order

An owned value is dropped when its owner goes out of scope: locals in reverse order of declaration. When an object is dropped, the object is freed first, then its fields in declaration order. A program never sees a dropped value.

Objects are freed when their owner is dropped, in the order above.

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

`defer` is NZ only. `using` has all its forms: `using (var x = e)`, `using (T x = e)`, `using (e)`, several declarators, and the `using var x = e;` declaration. It also accepts a value that is not `IDisposable` (it then only drops it). The block owns its value, so `using (x) { }` with an existing local moves `x` into the block, and `x` cannot be used after it.

### 17.7 `IDisposable` and `Dispose`

A class or struct that implements `IDisposable` has its `Dispose()` called **every time a value of it is dropped**, before its fields are dropped, so cleanup runs at a known point without a `using`.

```nz output="use\nclose\nafter\n"
package main;

import std.io.Console;

class File : IDisposable {
    public void Dispose() { Console.WriteLine("close"); }
}

void Main(string[] args) {
    {
        var f = new File();
        Console.WriteLine("use");
    }
    Console.WriteLine("after");
}
```

What a `Dispose` releases is never dropped inside it: a value with a `Dispose` of its own is dropped after that `Dispose` returns and its object has been dropped, in the order released (see [Ownership](Ownership.md#idisposable-cleanup-follows-ownership)). This holds for a struct with a `Dispose` as for a class object; dropped anywhere else, a value runs its `Dispose` where it is dropped.

---

## 18. Index

| Construct | Section |
|---|---|
| Source file layout | [1.1](#11-source-file-layout) |
| `package` | [1.2](#12-package) |
| `import` (four forms) | [1.3](#13-import) |
| Folders as packages | [1.4](#14-folders-as-packages) |
| Entry point `Main` | [1.5](#15-entry-point-main) |
| Top-level functions | [1.6](#16-top-level-functions) |
| Top-level constants and statics | [1.7](#17-top-level-constants-and-statics) |
| First-letter visibility rule | [2.1](#21-the-first-letter-rule-for-members) |
| `public`, `private` | [2.2](#22-public-and-private) |
| Visibility of types | [2.3](#23-visibility-of-types) |
| `class` | [3.1](#31-class) |
| `struct` | [3.2](#32-struct) |
| `record class`, `record struct` | [3.3](#33-record-class-and-record-struct) |
| `enum` | [3.5](#35-enum) |
| `union` | [3.6](#36-union) |
| `interface` (declaration) | [3.7](#37-interface) |
| Type aliases `type` | [3.9](#39-type-aliases-type) |
| `static class` | [3.10](#310-static-class) |
| Nested types | [3.11](#311-nested-types) |
| Generic types | [3.15](#315-generic-types) |
| Fields | [4.1](#41-fields) |
| `const` | [4.2](#42-const) |
| `readonly` fields | [4.3](#43-readonly-fields) |
| `static` fields and methods | [4.4](#44-static-fields-and-methods) |
| Methods | [4.5](#45-methods) |
| Expression-bodied members | [4.7](#47-expression-bodied-members) |
| Auto-properties | [4.8](#48-auto-properties) |
| Computed properties | [4.9](#49-computed-properties) |
| `required`, `init` | [4.11](#411-required-and-init) |
| Indexers | [4.12](#412-indexers) |
| Operator overloading | [4.13](#413-operator-overloading) |
| Constructors | [5.1](#51-constructors) |
| `: this(...)` | [5.2](#52-chaining-with--this) |
| Primary constructors | [5.4](#54-primary-constructors) |
| Object initializers | [5.5](#55-object-initializers) |
| Value parameters (moves) | [6.1](#61-value-parameters) |
| `in`, `ref` parameters | [6.2](#62-in-and-ref-parameters) |
| `params` | [6.4](#64-params) |
| Default values, named arguments | [6.5](#65-default-values-and-named-arguments) |
| Extension methods | [6.8](#68-extension-methods) |
| Type parameters, generic methods, inference | [7.1](#71-type-parameters-generic-methods-and-inference) |
| Constraints `where` | [7.2](#72-constraints-where) |
| Interface declaration | [8.1](#81-declaring-an-interface) |
| Implementing an interface | [8.2](#82-implementing-an-interface) |
| Default interface members | [8.3](#83-default-interface-members) |
| Interface inheritance | [8.4](#84-interface-inheritance) |
| `readonly` interface members | [8.5](#85-readonly-interface-members-nz-only) |
| Local functions | [10.1](#101-local-functions) |
| Attribute syntax | [11.1](#111-attribute-syntax) |
| `[Test]` | [11.2](#112-test) |
| Interop attributes | [11.3](#113-interop-attributes) |
| User-defined attributes | [11.6](#116-user-defined-attributes) |
| `Abort`, `Assert`, `Result` | [14.3](#143-failures-today) |
| `nameof` | [15.1](#151-nameof) |
| `unsafe` | [16.1](#161-unsafe) |
| Pointers, `&`, `stackalloc` | [16.2](#162-pointers--and-stackalloc) |
| `fixed` | [16.3](#163-fixed) |
| `NativeMemory` | [16.4](#164-nativememory) |
| `[StructLayout]` | [16.5](#165-structlayout) |
| `extern`, `[DllImport]` | [16.6](#166-extern-and-dllimport) |
| `[UnmanagedCallersOnly]` | [16.7](#167-unmanagedcallersonly) |
| `--no-runtime` | [16.8](#168---no-runtime) |
| One owner, moves | [17.1](#171-one-owner-moves) |
| Borrowing `in`, `ref` | [17.2](#172-borrowing-in-and-ref) |
| `Shared<T>`, `WeakReference<T>` | [17.3](#173-sharedt-and-weakreferencet) |
| `Arc<T>` | [17.4](#174-arct) |
| Drop order | [17.5](#175-drop-order) |
| `defer` | [17.6](#176-defer-and-using) |
| `using` | [17.6](#176-defer-and-using) |
| `IDisposable`, `Dispose` | [17.7](#177-idisposable-and-dispose) |

## Not available yet

These constructs are not available in NZ:

- Top-level statements
- `internal`
- `protected`
- `file`, `protected internal`, `private protected`
- Record equality, `ToString` and `with`
- `delegate`
- `partial`
- `abstract`, `sealed`, `virtual`, `override`, `new` (modifier)
- `readonly struct` and `ref struct`
- Method overloading
- The `field` keyword
- Conversion operators
- Events
- Chaining with `: base(...)`
- Target-typed `new()`
- Static constructors
- `out` parameters
- `ref readonly` and `scoped` parameters
- `ref` returns and `ref` locals
- Extension blocks
- Variance: `in` and `out` on type parameters
- Static members of generic types
- Generic local functions
- Explicit interface implementation
- Properties, indexers, operators and generic methods in an interface
- A default interface member taken by a struct
- `Shared<I>` and `Arc<I>` of an interface `I`
- Calling `Dispose()` directly
- Static and static abstract interface members
- Base classes
- `virtual`, `override`, `abstract`, `sealed`, `new`
- `base` access
- Casts between classes and `as`
- `object`
- Lambdas, `Func` and `Action`
- `delegate` types
- Events
- Function pointers: `delegate*`
- `[Flags]`
- `[Obsolete]` and other predefined attributes
- Attribute targets: `[return: ...]`, `[assembly: ...]`, `[module: ...]`
- `#if`, `#elif`, `#else`, `#endif`, `#define`, `#undef`
- `#region` and `#endregion`
- `#pragma warning`
- `#nullable`
- `#error`, `#warning`, `#line`
- `async`, `await`, `Task` and `Task<T>`
- `async void`, `ValueTask`, `await using`
- Iterators: `yield return`
- Async iterators: `IAsyncEnumerable<T>`, `await foreach`
- `lock` and `Mutex<T>`
- Exception types
- `typeof`, `GetType()` and `Type`
- Reading attributes
- `dynamic`
- `volatile`

The compiler refuses them:

**Top-level statements**

```nz error="a function needs a return type"
package main;

import std.io.Console;

Console.WriteLine("top");
```

**`protected`**

```nz error="`protected` is reserved for a future feature"
package main;

class C {
    protected int X;
}

void Main(string[] args) { }
```

**`abstract`, `sealed`, `virtual`, `override`, `new` (modifier)**

```nz error="`abstract` is reserved for a future feature"
package main;

abstract class Shape { }

void Main(string[] args) { }
```

**Method overloading**

```nz error="duplicate member `F` in `M`"
package main;

class M {
    public int F(int x) => x;
    public string F(string s) => s;
}

void Main(string[] args) { }
```

**The `field` keyword**

```nz error="unknown name `field`"
package main;

class C {
    public int P { get => field; set => field = value; }
}

void Main(string[] args) { }
```

**Conversion operators**

```nz error="operator `double` cannot be overloaded"
package main;

struct Meters {
    public double V;
    public static implicit operator double(Meters m) => m.V;
}

void Main(string[] args) { }
```

**Static constructors**

```nz error="`static` is not supported here"
package main;

class C {
    static C() { }
}

void Main(string[] args) { }
```

**Static members of generic types**

```nz error="not supported yet: a static member of a generic type"
package main;

class Box<T> {
    public static int Count() => 0;
}

void Main(string[] args) { }
```

**`object`**

```nz error="unknown type `object`"
package main;

void Main(string[] args) {
    object o = 1;
}
```

**`[Flags]`**

```nz output="3\n"
package main;

import std.io.Console;

enum Perm { None = 0, Read = 1, Write = 2 }

void Main(string[] args) {
    var p = Perm.Read | Perm.Write;
    Console.WriteLine($"{(int)p}");
}
```

**Attribute targets: `[return: ...]`, `[assembly: ...]`, `[module: ...]`**

```nz error="attribute targets are not supported yet"
package main;

[return: Tag]
string Name() { return "n"; }

void Main(string[] args) { }
```

**`#if`, `#elif`, `#else`, `#endif`, `#define`, `#undef`**

```nz error="unexpected character"
package main;

#define FAST
#if FAST
void Main(string[] args) { }
#endif
```

**`async`, `await`, `Task` and `Task<T>`**

```nz error="`async` is reserved for a future feature"
package main;

async Task<int> F() { return 1; }

void Main(string[] args) { }
```

**`typeof`, `GetType()` and `Type`**

```nz error="`Dog` has no member `GetType`"
package main;

class Dog { }

void Main(string[] args) {
    var d = new Dog();
    string n = d.GetType().Name;
}
```

**`dynamic`**

```nz error="unknown type `dynamic`"
package main;

void Main(string[] args) {
    dynamic o = 1;
}
```

## Not supported

NZ does not support these constructs:

- `namespace` and `using` directives: `package` and `import` replace them (1.8).
- Finalizers `~C()`: `IDisposable` runs at every drop (17.7).
- Query expressions (`from x in xs select x`): loops and methods cover the use.
- Anonymous types (`new { X = 1 }`): tuples cover the use.
- Boxing values into `object`: there is no common root type, and `object` is not available.

The compiler refuses `namespace`, finalizers and anonymous types:

**`namespace` and `using` directives**

```nz error="expected `package name;` at the start of the file"
namespace App;

void Main(string[] args) { }
```

**Finalizers**

```nz error="expected identifier, found Tilde"
package main;

class C {
    ~C() { }
}

void Main(string[] args) { }
```

**Anonymous types**

```nz error="expected identifier"
package main;

void Main(string[] args) {
    var a = new { X = 1 };
}
```
