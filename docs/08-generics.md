# 8. Generics

> NZ is still in development and is not stable. Anything in this documentation can change.

## 8.1 Type inference

A generic function call infers the type arguments from its arguments. When nothing can be inferred, write them out: `Empty<int>()`. Here `Empty` has no parameter that mentions `T`, so `Empty()` alone is an error (the message is ``cannot infer the type argument `T` of `Empty` ``).

```nz output="empty\n"
package main;

import std.io.Console;

T? Empty<T>() {
    return null;
}

void Main(string[] args) {
    int? a = Empty<int>();
    if (a == null) {
        Console.WriteLine("empty");
    }
}
```

Explicit type arguments work on a function (`Empty<int>()`) and on a method (`x.First<int>(1)`, `x?.First<int>(1)`). After a name or a member access, a `<` starts a type-argument list only when the matching `>` is followed by one of `(` `)` `]` `}` `:` `;` `,` `.` `?` `==` `!=` `|` `^` `&&` `||`. Otherwise it is the less-than operator. Type arguments must be followed by a call: `Empty<int>` alone is an error. So `a < b > (c)` is a generic call, and `(a) < b > (c)` or `f(x) < b > (c)` are comparisons because the left side is not a plain name.

## 8.2 Generic class

```nz
package main;

class Box<T> {
    T? content;
}

void Main(string[] args) { }
```

## 8.3 Generic function

**Generic functions** infer their type from the arguments:

```nz output="1\nfirst\n"
package main;

import std.io.Console;

T First<T>(T a, T b) {
    return a;
}

void Main(string[] args) {
    int x = First(1, 2);
    string s = First("first", "second");
    Console.WriteLine($"{x}");
    Console.WriteLine(s);
}
```

## 8.4 Covariance

This section is not written yet.

Outline:

- `out` and `in` type parameters
- Variance of read-only views

## 8.5 Constraints

**Constraints** (`where` clauses) say what a type argument must be: `where T : IShape` (it implements the interface), `class` (a reference type: a class, an interface, `string` or a library type such as `List<T>`), `struct` (a value type: a struct, enum, union, tuple, number, `bool` or `char`) and `new()` (a class or struct with a constructor that takes no arguments, so `new T()` works). A type parameter can then call the members of its interfaces, and a call checks each type argument against the constraints (`` `double` does not satisfy `T : IComparable<double>` ``). `class` or `struct` comes first, `new()` last. A generic function is compiled once per type argument, so in `Total<Rect>` the call `a.Area()` calls the struct's method directly, and in `Total<IShape>` it dispatches. A class's own `where` clauses hold in its methods and field initializers.

The library interface **`IComparable<T>`** (`readonly int CompareTo(in T other)`, negative, zero or positive) is implemented by every integer type, `char` and `string` (which compares by bytes), so `where T : IComparable<T>` accepts them; `float` and `double` do not implement it. On these types `CompareTo` gives exactly -1, 0 or 1; a user type may give any negative or positive value, so compare the result with 0. A user type may implement it as any interface.

```nz output="8 12\n7 pear -1\n"
package main;

import std.io.Console;

interface IShape {
    readonly int Area();
}

class Square : IShape {
    int Side = 2;
    int Area() { return Side * Side; }
}

struct Rect : IShape {
    int W;
    int H;
    int Area() { return W * H; }
}

int Total<T>(in T a, in T b) where T : IShape {
    return a.Area() + b.Area();
}

T Make<T>() where T : new() {
    return new T();
}

T Max<T>(T a, T b) where T : IComparable<T> {
    return a.CompareTo(b) >= 0 ? a : b;
}

void Main(string[] args) {
    var s = Make<Square>();
    var r = new Rect { W = 2, H = 3 };
    Console.WriteLine($"{Total(s, s)} {Total(r, r)}");
    Console.WriteLine($"{Max(3, 7)} {Max("pear", "apple")} {'a'.CompareTo('b')}");
}
```

```nz error="`double` does not satisfy `T : IComparable<double>` in `Max`"
package main;

T Max<T>(T a, T b) where T : IComparable<T> {
    return a.CompareTo(b) >= 0 ? a : b;
}

void Main(string[] args) {
    var m = Max(1.5, 2.5);
}
```

Previous: [7. Concurrency and parallel programming](07-concurrency-and-parallel-programming.md#7-concurrency-and-parallel-programming) · [Contents](README.md) · Next: [9. Domain-specific languages](09-domain-specific-languages.md#9-domain-specific-languages)
