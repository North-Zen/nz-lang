# 5. Object-oriented programming

> NZ is still in development and is not stable. Anything in this documentation can change.

## 5.1 Object

This section is not written yet.

Outline:

- Making an object
- Owning, moving and lending an object

## 5.2 Class

A `class` is a heap object with a **single owner** (move-only); a `struct` is a value type that is copied. An object is freed when its owner goes out of scope.

```nz output="1\n"
package main;

import std.io.Console;

class Counter {
    int count;
    string label = "counter";

    int Next() {
        count += 1;
        return count;
    }
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.Next();
    Console.WriteLine($"{n}");
}
```

Assigning an object to another variable, or passing it to a parameter without a modifier, **moves** it: the new owner has it, and the old variable cannot be used again until it is given a new value. Passing an object somewhere and then using it again is rejected:

```nz error="moved"
package main;

import std.io.Console;

class Counter {
    int Count;
}

void Show(Counter c) {
    Console.WriteLine($"{c.Count}");
}

void Main(string[] args) {
    var c = new Counter();
    Show(c);
    Show(c);
}
```

To keep the object, lend it instead: `in` lends it for reading and `ref` lends it to be changed. The caller still owns it after the call. `in` is a read-only reference for every type, value types included, not a copy, except `in string` and `in string?`, which hold a copy of their own (see [Ownership: Strings](+/Ownership.md#strings)) ([Ownership](+/Ownership.md#borrowing-in-and-ref)):

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

A field or property needs a value: **an initial value, an optional type, or an assignment in every constructor**. Numbers, `bool` and `char` are zero automatically, and a `string` is the empty string `""`; a class, a struct, an array or a list has no default:

```nz error="needs a value"
package main;

class Label { }

class User {
    Label Name;
}

void Main(string[] args) { }
```

```nz
package main;

class User {
    string Name = "no name";
    string? Note;
    int Age;
}

void Main(string[] args) { }
```

## 5.3 Property and method

### Properties

**Properties** use `get` and `set`; a property without `set` cannot be assigned:

```nz
package main;

class Item {
    int Qty { get; set; }
    string? Note { get; set; }
}

void Main(string[] args) { }
```

An **auto-property** (`{ get; set; }`, `{ get; }`, `{ get; init; }`) stores its value like a field and may have an initializer (`= value;`). A struct auto-property is a variable, so `c.P.X = 1` changes the stored struct; with a computed property NZ refuses it, because the getter returns a copy. A get-only auto-property is set by its initializer or in a constructor of its type; an `init` property also by an object initializer. One accessor may be `private` (`{ get; private set; }`): outside the type the property is then read-only.

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

A **computed property** has accessors with bodies: `get { ... }` returns the value, `set { ... }` (or `init { ... }`) receives it as `value`. It has no storage of its own. `=> expr` is a getter written as one expression, for the whole property (`int Area => W * H;`) or for one accessor (`get => n;`). Accessors are all bodies or all `;`, never mixed.

```nz output="Temp 20 68 21\n"
package main;
import std.io.Console;

class Thermometer {
    double celsius;

    public double Celsius {
        get { return celsius; }
        set {
            if (value < -273.15) {
                return;
            }
            celsius = value;
        }
    }

    public double Fahrenheit => celsius * 9 / 5 + 32;
}

void Main(string[] args) {
    var t = new Thermometer();
    t.Celsius = 20;
    t.Celsius = -300;
    double f = t.Fahrenheit;
    t.Celsius += 1;
    Console.WriteLine($"Temp 20 {f} {t.Celsius}");
}
```

Reading a computed property calls its getter, assigning calls its setter, and `+=`, `++` and the other compound forms call the getter once and the setter once. The receiver and the index of a compound write are evaluated once, first, then the getter, the value and the setter; a value that could change the receiver's variable or field before the setter runs (`c.P += Bump(ref c)`) is refused for now. A computed property is not a variable: it cannot be passed by `ref`, a setter cannot change a struct value that is not a variable (a call result, a list element, another property's value), and a field of a struct value that a getter, an indexer or an operator gave cannot be assigned (`c.P.X = 1`, `g[0].X++`: the change would be lost):

```nz error="not a variable"
package main;

struct Size {
    int w;
    public int W { get { return w; } set { w = value; } }
}

Size Make() { return new Size(); }

void Main(string[] args) {
    Make().W = 3;
}
```

Not supported yet: using the value of an assignment, a compound assignment or `++`/`--` through a computed property inside a larger expression (write it as a statement); a compound assignment whose value may change the receiver's variable or field before the setter runs (`c.P += Bump(c)` where `Bump` takes `ref C`, or any call when the receiver is reached through a `Shared<T>` handle; compute the value into a variable first); a computed property set in an object initializer or marked `required`; a property (computed or auto) inside a property pattern (`c is { P: > 0 }`: property patterns read fields; test `c.P is > 0` instead, which calls the getter once); static properties; accessor bodies in an interface.

### Indexers

An **indexer** lets a class or struct be indexed like an array: `T this[P p] { get { ... } set { ... } }`. Its accessors need bodies (it has no storage), the setter receives `value`, and it is written `public` or `private`. A user indexer does not make a type usable in `foreach`.

```nz output="16 0 n3\n"
package main;
import std.io.Console;

class Grid {
    int[] cells = new int[4];

    public int this[int i] {
        get { return cells[i]; }
        set { cells[i] = value; }
    }
}

class Names {
    public string this[int i] => $"n{i}";
}

void Main(string[] args) {
    var g = new Grid();
    g[1] = 5;
    g[1] += 2;
    g[1]++;
    g[1] *= 2;
    var n = new Names();
    Console.WriteLine($"{g[1]} {g[0]} {n[3]}");
}
```

### Operator overloading

**Operators** can be declared by a class or struct: `public static T operator +(A a, B b)`. The operators that can be declared are `+ - ! ~ ++ --` (unary) and `+ - * / % == != < > <= >= & | ^ << >>` (binary); `==` and `!=`, `<` and `>`, `<=` and `>=` are declared in pairs, at least one parameter has the declaring type, and `++`/`--` take and return it. The operands are lent with `in` and never moved; the result is a new value. `a += b` uses `+`, and `a == null` stays the null test.

```nz output="8 6 -6 False True\n"
package main;
import std.io.Console;

struct Vec {
    public int X;

    public static Vec operator +(Vec a, Vec b) => Make(a.X + b.X);
    public static Vec operator *(Vec a, int k) => Make(a.X * k);
    public static Vec operator -(Vec a) => Make(-a.X);
    public static Vec operator ++(Vec a) => Make(a.X + 1);
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
    a += b;
    a++;
    Console.WriteLine($"{c.X} {a.X} {(-a).X} {a == b} {a != b}");
}
```

Not supported yet: an indexer with several parameters, more than one indexer or more than one operator with the same symbol in a type, an indexer written without `public` or `private`, an indexer or operator in an interface, operators of a generic type, conversion operators (`implicit`, `explicit`) and `true`/`false`, and using the value of a user `++` or `--` (write it as a statement).

## 5.4 Access modifiers

The visibility rules are in [20.3 Access modifiers](20-module-system-and-visibility.md#203-access-modifiers).

## 5.5 Namespace (NZ: packages)

Packages and imports are in [20. Module system and visibility rules](20-module-system-and-visibility.md#20-module-system-and-visibility-rules).

## 5.6 Encapsulation

This section is not written yet.

Outline:

- Private fields
- Properties as the public surface

## 5.7 Constructor

A **constructor** is a member named after its type, with no return type. A type may declare several; `new` picks the one whose parameters take the arguments (named and default arguments work as in calls), and when several do, the one whose parameter types equal the argument types. A type that declares a constructor has only the ones it declares; a type that declares none has the implicit `new T()`. A constructor is as visible as its type unless it is written `private`.

```nz output="Ada 36\nnobody 0\n"
package main;

import std.io.Console;

class Person {
    string Name;
    int Age;

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

`: this(...)` runs another constructor of the same type first. A field without an initial value whose type has no default (`Name` above, a class) must be assigned on **every path** of every constructor that does not chain, before it is read and before `this` is used (a method call, `this` passed as a value). A local function called, or a `defer` that runs, while such a field is unassigned counts as a read. The arguments of `: this(...)` run before any constructor, so they may use the parameters and constants but no member of the object and no `this`. A constructor that chains starts with every field assigned:

```nz error="is not assigned on every path"
package main;

class Label { }

class Token {
    Label Text;

    Token(bool empty) {
        if (!empty) {
            Text = new Label();
        }
    }
}

void Main(string[] args) { }
```

Creating an object runs, in order: the arguments of `new` from left to right, the field initializers (once, in the constructor that does not chain), the constructor bodies (a chained one first), then the object initializer.

A **primary constructor** writes the parameters after the type's name; the field initializers read them. Every other constructor of the type chains to it with `: this(...)`. Using a primary-constructor parameter in a method is not supported yet: store it in a field.

```nz output="sum 3\n"
package main;

import std.io.Console;

class Labelled(string label, int a, int b) {
    string Label = label;
    int Total = a + b;
}

void Main(string[] args) {
    var l = new Labelled("sum", 1, 2);
    Console.WriteLine($"{l.Label} {l.Total}");
}
```

### Object initializers

`new T(args) { A = a, B = b }` sets fields and settable properties after the constructor ran, in the order written; `new T { ... }` uses the parameterless constructor. A member is set once per initializer.

```nz output="3 4\n"
package main;

import std.io.Console;

class Point {
    int X;
    int Y;
}

void Main(string[] args) {
    var p = new Point { X = 3, Y = 4 };
    Console.WriteLine($"{p.X} {p.Y}");
}
```

A `required` member is set by the object initializer of every `new`; it has no initial value. A `readonly` field is assigned only by its initial value and by the constructors of its type (through `F` or `this.F`), and never by an object initializer. The parts of a `readonly` struct field are the field: they are not assigned or passed by `ref` elsewhere either. A method called on a `readonly` struct field outside those constructors runs on a copy, so the field does not change; a struct that owns an object (a class, a list) cannot be copied, so a method that changes it is refused there:

```nz error="must set the required member"
package main;

class User {
    required string Id;
    readonly int Created;

    User() {
        Created = 1;
    }
}

void Main(string[] args) {
    var u = new User();
}
```

## 5.8 Variable shadowing

This section is not written yet.

Outline:

- Locals
- Fields and locals
- Local functions

## 5.9 Inheritance

This section is not written yet.

Outline:

- Base and derived classes
- `virtual` and `override`
- `sealed`
- Interfaces

## 5.10 Abstract

This section is not written yet.

Outline:

- Abstract classes
- Abstract members

## 5.11 Enumerated types

Enums are described in [1.4.6 Enum](01-basic-data-structures.md#146-enum).

## 5.12 Extension methods

**Extension methods.** A top-level function whose first parameter is marked `this` can also be called as if it were a member of that parameter's type: `5.Double()` calls `Double(5)`. The receiver is evaluated first, then the arguments. On a copied type the receiver is a plain `this` parameter (a copy); on an owned type (a class, a list, an array, a struct that owns something, a type parameter) it must say how it borrows the receiver, `this in T` (read) or `this ref T` (write), so calling an extension never moves the receiver. A `this ref` receiver must be a variable, field or element, as any `ref` argument.

```nz output="20 hi!! 5 2 3\n"
package main;

import std.io.Console;
import std.collections.List;

int Double(this int x) {
    return x * 2;
}

string Exclaim(this string s, int times = 1) {
    string r = s;
    for (int i = 0; i < times; i++) {
        r = r + "!";
    }
    return r;
}

int Total(this in List<int> xs) {
    int t = 0;
    foreach (var x in xs) {
        t += x;
    }
    return t;
}

void Bump(this ref int x) {
    x++;
}

void Main(string[] args) {
    List<int> l = new List<int>();
    l.Add(2);
    l.Add(3);
    int n = 1;
    n.Bump();
    n.Bump();
    Console.WriteLine($"{5.Double().Double()} {"hi".Exclaim(times: 2)} {l.Total()} {l.Count} {n}");
}
```

```nz error="must borrow its receiver"
package main;

import std.collections.List;

int Size(this List<int> xs) {
    return xs.Count;
}

void Main(string[] args) { }
```

How an extension is found: a member of the receiver's type always wins over an extension of the same name; otherwise the function must be visible in the file (declared in the package or imported) and its first parameter marked `this`. There is no overloading: one name names one function. The receiver's type must be exactly the extended type once type parameters are inferred: no numeric widening (`5.Twice()` does not reach `this long x`) and no `int` to `int?`; an optional variable narrowed by `!= null` still reaches an extension on its optional type. Through a `Shared<T>` or `Arc<T>` handle an extension on `T` is reached as a member of `T` is, lending the object for the call (`this ref` through an `Arc<T>` is refused: its object never changes). An extension through `?.` and an extension on a type name (`Color.F()`) are not supported yet. An extension can always be called as a plain function too (`Double(5)`).

## 5.13 The `base` keyword

This section is not written yet.

Outline:

- Calling a base constructor
- Calling a base member

## 5.14 Polymorphism

This section is not written yet.

Outline:

- Virtual calls
- Interfaces

## 5.15 Meta-programming

Attributes and reflection are in [24. Reflection and meta-programming](24-reflection-and-meta-programming.md#24-reflection-and-meta-programming).

## 5.16 Static

A class or struct can have member constants, static fields and static methods. A static member belongs to the type, not to an object: it is reached as `Type.Name`, or by its bare name inside the type. A static method has no `this`; inside it, an instance member is an error.

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

A static member is never reached through an object:

```nz error="is static; write `Counter.Twice(...)`"
package main;

class Counter {
    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    int n = new Counter().Twice(1);
}
```

- A **static field** is a static (see [Statics](#static-variables)): the same initializer rules, `unsafe` for a mutable one, and an owned one is dropped at exit. It is not part of the object.
- A **member constant** is static already: `static const` is an error.
- A **static method** may use the static members and constants of its type by bare name, and is called by bare name from the type's other methods and local functions.
- A plain `readonly` field (not `static`) is assigned only by its initial value and the constructors of its type (see [Object initializers](#object-initializers)).
- Static members of generic types, static properties, and static members of interfaces are not supported yet.

### Static classes

`static class` is a class whose members are all static: it groups functions and constants and has no instances. An instance field, property, method, indexer, constructor or operator in it is an error, and so is using it as a type (a variable, parameter, return value, field, array element or type argument) or writing `new` on it. Call its members as `Type.Name`:

```nz output="True\nFalse\n8\n"
package main;

import std.io.Console;

static class Chars {
    public const int Base = 4;
    public static bool IsDigit(char c) { return c >= '0' && c <= '9'; }
    public static int Twice(int x) => x * 2;
}

void Main(string[] args) {
    Console.WriteLine(Chars.IsDigit('5'));
    Console.WriteLine(Chars.IsDigit('x'));
    Console.WriteLine(Chars.Twice(Chars.Base));
}
```

An extension method (`this` on the first parameter) is still declared as a top-level function, not inside a static class; the compiler says so. A static class cannot be a base class or derive from one (inheritance is not supported yet).

### Static variables

A `static` is one variable for the whole program, declared at the top level or as a [static field](#516-static). A mutable static can be read and written only inside `unsafe`: an `unsafe { ... }` block, or an `unsafe` function, method, class or struct. A `static readonly` is never written and can be read anywhere.

```nz output="hi 2\n"
package main;

import std.io.Console;

static int counter;
static readonly string Greeting = "hi";

unsafe void Bump() {
    counter++;
}

void Main(string[] args) {
    unsafe {
        Bump();
        Bump();
        Console.WriteLine($"{Greeting} {counter}");
    }
}
```

```nz error="can only be used inside `unsafe`"
package main;

static int counter = 0;

void Main(string[] args) {
    counter = 1;
}
```

- **Initializers are constant.** A static starts with a constant expression, `null` (optional types), `default`, or nothing when its type has a default value (numbers, `bool`, `char`, `string`, optionals). A value computed when the program starts (`static List<int> xs = new List<int>();`) is not supported yet. So a static of a `char`, `double` or `float` type cannot take a literal initializer: it is zero by default (numbers, `bool`, `char`), or written as an optional. A static of an enum type starts with a member of that enum (`static readonly Color c = Color.Red;`, `static Color? d = Color.Red;`) or is optional (`static Color? e = null;`; enums have no default). A member keeps its enum's type there too: `static int n = Color.Red;` and `const int K = Color.Red;` are errors (``cannot implicitly convert `Color` to `int`; write `(int)Color.Red` ``), and `(int)Color.Red` works. The same holds for a tuple: `(int, int)` needs `(int, int)?`, as a tuple field needs an initializer.

```nz output="Green 1\n"
package main;

import std.io.Console;

enum Color { Red, Green }

static readonly Color Start = Color.Green;
static readonly int Index = (int)Color.Green;

void Main(string[] args) {
    Console.WriteLine($"{Start} {Index}");
}
```

```nz error="cannot implicitly convert `Color` to `int`; write `(int)Color.Green`"
package main;

enum Color { Red, Green }

static readonly int Index = Color.Green;

void Main(string[] args) {
}
```
- **Never narrowed.** Any call can change a static, so `if (s != null) { ... }` does not make `s` non-null inside; test it with a pattern (`if (s is int v)`) or copy it into a local first.

- A static is a place: it can be passed by `ref`, and two `ref` arguments naming one static are an overlapping borrow.

#### Owned statics

A static of an owned type (a class, a list, an array, or a struct or tuple that owns something) is always optional, since such a type has no constant and no default. The static owns its value for the whole program:

- assigning a new value drops the old one;
- moving out of it is an error (pass it with `in`, or assign it a new value);
- the program drops every owned static when `Main` returns, in reverse declaration order (static fields included), before the `NZ_ALLOC_REPORT=1` report. `nzc test` drops them after each test. An `Abort`, a failed `Assert` or another runtime stop drops nothing, as everywhere.

```nz output="0\n"
package main;

import std.io.Console;

class Node {
    int V;
}

static Node? head;

void Main(string[] args) {
    unsafe {
        Node n = new Node();
        n.V = 1;
        head = n;
        head = new Node();
        if (head is Node h) {
            Console.WriteLine($"{h.V}");
        }
    }
}
```

```nz error="cannot move out of static `head`"
package main;

class Node {
    int V;
}

static Node? head;

void Main(string[] args) {
    unsafe {
        Node? n = head;
    }
}
```

**Any call may replace a static.** A function the compiler does not see at this point could assign `head = null`, which drops the node. So while something an owned static holds is in use, a call that may run NZ code is checked. Such a call is any call of a function, method, local function or static method, and `new T()` when a field initializer of `T` calls one. Calls of the standard library (`Console`, `List`, `Diag`, `string`) and `Abort`/`Assert` are not.

- Inside a `foreach` over a collection an owned static holds, such a call is an error.
- A pattern variable that borrows from an owned static (`if (head is Node h)`) can no longer be used after such a call. Test the static again after the call.
- When a call already holds part of an owned static (an `in` argument, the receiver of `head?.M(...)`, a `ref` to a field inside it), a later argument with such a call is an error. A `ref` to the whole static is allowed: it passes the static's address, read when the callee runs.
- The same for an assignment into an object a static holds (`h.V = F();`) and for an index into a list a static holds (`l[F()]`).

These rules also refuse some safe code. The fix is the same each time: compute the value in a statement of its own first.

```nz error="before the value is stored"
package main;

class Node {
    int V;
}

static Node? head;

int Next() {
    return 2;
}

void Main(string[] args) {
    unsafe {
        head = new Node();
        if (head is Node h) {
            h.V = Next();
        }
    }
}
```

```nz output="2\n"
package main;

import std.io.Console;

class Node {
    int V;
}

static Node? head;

int Next() {
    return 2;
}

void Main(string[] args) {
    unsafe {
        head = new Node();
        int v = Next();
        if (head is Node h) {
            h.V = v;
            Console.WriteLine($"{h.V}");
        }
    }
}
```

**What `unsafe` vouches for.** The analysis checks one function at a time. A function that receives a lent static (`Count(head)` with `int Count(in Node? n)`) cannot free it from safe code: a write to the static, or a call of a function that may write one, needs `unsafe` (see above). Inside `unsafe` code the programmer vouches for these, as for Rust's `static mut`:

- an `unsafe` function or method that receives a static by `in` or `ref`, or as its receiver (`head?.M()`), and calls something that assigns that static;
- an `unsafe` method of an object an owned static holds that frees the static (`ee?.M()` where `M` sets `ee = null`);
- an `unsafe` callee given a field inside a static by `ref` (`Rf(h.Next)` where `Rf` frees the static).

The checks above still refuse every such call that is visible where the static is held.

## 5.17 Metadata

Attributes are in [24.1 Attributes](24-reflection-and-meta-programming.md#241-attributes).

## 5.18 Nested types

A `class`, `struct`, `record`, `enum` or `union` may be declared inside a class, struct, record or union. Inside the enclosing type its bare name is enough; from outside it is written `Outer.Inner`, as a type and in expressions (`new Outer.Inner()`, `Outer.Inner.Make()`, `Outer.Mode.Fast`). A nested type is an ordinary type: it has its own members, constructors and static members, it is owned, copied and dropped like a top-level one, and it may be nested further.

```nz output="1 2\n"
package main;

import std.io.Console;

public class Shape {
    public class Point {
        public int X = 1;
        public int Y = 2;
    }

    public Point Origin() { return new Point(); }
}

void Main(string[] args) {
    var p = new Shape().Origin();
    Shape.Point q = p;
    Console.WriteLine($"{q.X} {q.Y}");
}
```

A nested type is **private to its enclosing type** unless it is written `public`: only the enclosing type, and the types declared inside it, can name it. A nested type can also use every private member of the types around it; the enclosing type cannot use the private members of a nested type. A type name written inside the enclosing type finds the nested types before the top-level ones, so a nested `Node` hides a top-level `Node` there. Two enclosing types may each declare a nested type of the same name.

```nz error="nested type `Hidden` is private to `Outer`"
package main;

class Outer {
    class Hidden { }
}

void Main(string[] args) {
    var h = new Outer.Hidden();
}
```

```nz output="9\n"
package main;

import std.io.Console;

class Outer {
    int secret = 9;

    public class Peek {
        public int Read(in Outer o) { return o.secret; }
    }
}

void Main(string[] args) {
    Console.WriteLine($"{new Outer.Peek().Read(new Outer())}");
}
```

The nested type of a **generic** type takes the type parameters of the enclosing type first: `Box<int>.Cell` and `Box<string>.Cell` are different types, and inside `Box<T>` the name `Cell` means `Box<T>.Cell`. A nested type may declare parameters of its own (`Outer<int>.Pair<string>`).

```nz output="12\n"
package main;

import std.io.Console;

public class Box<T> {
    public class Cell {
        public T? Value;
    }

    public Cell C = new Cell();
}

void Main(string[] args) {
    var b = new Box<int>();
    b.C.Value = 12;
    Console.WriteLine($"{b.C.Value}");
}
```

A dotted path written inside the generic type gets the same arguments (`I.K` inside `O<T>` is `O<T>.I.K`), and a nested type is reached through an alias of an instantiated type (`type A = O<int>;`, `A.I`).

```nz output="3 5\n"
package main;

import std.io.Console;

class O<T> {
    public class I {
        public class K { public int V = 3; }
        public T? A;
    }

    public int Get() { var k = new I.K(); return k.V; }
}

type Alias = O<int>;

void Main(string[] args) {
    var i = new Alias.I();
    i.A = 5;
    Console.WriteLine($"{new O<string>().Get()} {i.A}");
}
```

A bare name inside a nested type finds the member of the nearest enclosing type that has that name. A static member (a constant, a static field or method) is used as it stands; an instance member there is an error, because a nested type has no object of its enclosing type of its own. A nested enum may use the constants of its enclosing type by their bare names.

An `import` names a top-level declaration only: import `geo.Shape` and write `Shape.Point`. Importing `geo.Shape.Point` is an error that says so. A nested type cannot have the name of its enclosing type, of a member of it, or of a type parameter of the types around it. A package whose path equals the path of a type that has nested types is an error (the path `a.b.C` would name two things). In the generated C a nested type has its own name, built from its whole path, so it never meets a top-level type such as `Outer_Inner`.

Not supported yet: a type nested in an interface, a `static` nested type (there are no static classes), an enum nested in a generic type, and static members of a type nested in a generic type (a static member of a generic type is not supported at all yet).

Previous: [4. Struct](04-struct.md#4-struct) · [Contents](README.md) · Next: [6. Exceptions and error handling](06-exceptions-and-error-handling.md#6-exceptions-and-error-handling)
