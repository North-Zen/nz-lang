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

This section is not written yet.

Previous: [7. Concurrency and parallel programming](07-concurrency-and-parallel-programming.md#7-concurrency-and-parallel-programming) · [Contents](README.md) · Next: [9. Domain-specific languages](09-domain-specific-languages.md#9-domain-specific-languages)
