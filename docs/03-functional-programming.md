# 3. Functional programming

> NZ is still in development and is not stable. Anything in this documentation can change.

## 3.1 Functions are first-class and can be higher-order

This section is not written yet.

Outline:

- Lambdas
- Closures
- Delegates (`Action`, `Func`)
- Events

## 3.2 Main function

This section is not written yet.

Outline:

- The forms of `Main`
- The exit code

## 3.3 Pure functions

This section is not written yet.

## 3.4 Recursion

This section is not written yet.

Outline:

- Recursive functions
- Recursion depth and the stack

## 3.5 Immutable variables in functions

This section is not written yet.

Outline:

- `in` parameters
- Local constants
- `readonly` fields

## 3.6 Functions with and without a return value

Functions are declared: **the return type goes in front of the name**, and `void` stands for "returns nothing" (it is only valid as a return type). The entry point of a program is `void Main(string[] args)`.

```nz output="Hello, NZ\n3\n"
package main;

import std.io.Console;

int Add(int a, int b) {
    return a + b;
}

void Greet(string name) {
    Console.WriteLine($"Hello, {name}");
}

void Main(string[] args) {
    Greet("NZ");
    Console.WriteLine($"{Add(1, 2)}");
}
```

A non-void function must return a value on **every path**:

```nz error="not all code paths return"
package main;

int Positive(bool b) {
    if (b) {
        return 1;
    }
}

void Main(string[] args) { }
```

## 3.7 Function parameters



### `ref` parameters

**`ref` parameters** are used when a function writes to the caller's variable. The argument must be a variable of exactly the same type:

```nz output="x = 2, y = 1\n"
package main;

import std.io.Console;

void Swap(ref int a, ref int b) {
    var tmp = a;
    a = b;
    b = tmp;
}

void Main(string[] args) {
    int x = 1;
    int y = 2;
    Swap(x, y);
    Console.WriteLine($"x = {x}, y = {y}");
}
```

### Default parameter values

**Default values.** A parameter can have a default value; a call may leave it out. A default is a constant expression (see [1.8 Constants](01-basic.md#constants); `string` constants and the constants of the declaring type count), `null` for an optional parameter, `default`/`default(T)` where the type has a default, or a `float`, `double` or `char` literal (with a sign or casts: `-0.5`, `2F`, `'x'`, `(char)120`). Every parameter after one with a default needs a default too. A `ref`, `this` or `params` parameter has none; an `in` parameter may have one (the caller stores it in a temporary and passes its address). A default is stored with the function, so a call from another package uses the declaring package's constants.

### Named arguments

**Named arguments.** An argument can name its parameter: `Sub(b: 1, a: 5)`. Positional arguments come first; a name may not repeat, nor name a parameter already given by position; every parameter without a default must receive a value.

```nz output="6 4 a, z a, c 1.5 x\n"
package main;

import std.io.Console;

const int Step = 1;

int Sub(int a, int b = Step) {
    return a - b;
}

string Join(string a, string sep = ", ", string b = "z") {
    return a + sep + b;
}

string Show(double x = 1.5, char c = 'x') {
    return $"{x} {c}";
}

void Main(string[] args) {
    Console.WriteLine($"{Sub(7)} {Sub(b: 1, a: 5)} {Join("a")} {Join("a", b: "c")} {Show()}");
}
```

Arguments are evaluated **in the order they are written**, whatever parameters they bind to; moves and borrows follow the same order:

```nz output="ba 4\n"
package main;

import std.io.Console;

int Log(string s, int v) {
    Console.Write(s);
    return v;
}

int Sub(int a, int b) {
    return a - b;
}

void Main(string[] args) {
    Console.WriteLine($" {Sub(b: Log("b", 1), a: Log("a", 5))}");
}
```

```nz error="positional argument after a named argument"
package main;

int Sub(int a, int b) {
    return a - b;
}

void Main(string[] args) {
    int x = Sub(b: 1, 2);
}
```

A named argument in its own position followed by positional ones (`Sub(a: 1, 2)`) is refused too: every positional argument comes before the first named one.

Named arguments on built-in members (`Console.WriteLine(value: x)`, `List.Add`), on `Abort` and `Assert`, and on `new` are not supported yet.

### `params`

`params` parameters wait for spans: ``not supported yet: `params` needs `Span<T>`, which arrives with spans``.

The last parameter can be `params Span<T>` or, `params T[]`. A call then passes either one array, span or collection expression for it, or any number of values of type `T`, which are put in a new array:

```nz output="0 6 9 2\n2 0\n"
package main;
import std.io.Console;

int Sum(params Span<int> xs) {
    int t = 0;
    foreach (int x in xs) {
        t += x;
    }
    return t;
}

int Count(params string[] words) {
    return words.Length;
}

void Main(string[] args) {
    int[] a = [1, 1];
    Console.WriteLine($"{Sum()} {Sum(1, 2, 3)} {Sum([4, 5])} {Sum(a)}");
    Console.WriteLine($"{Count("x", "y")} {Count()}");
}
```

The values are moved into the new array (or counted, for strings and handles). For a `params Span<T>` the array is a temporary, freed with its elements at the end of the statement; a `params T[]` array is given to the function, as any array passed by value. `params` on any other parameter, or on a type that is not a span or an array, is an error.

## 3.8 Main function parameters

This section is not written yet.

Outline:

- `string[] args`
- Passing arguments with `nzc run`

## 3.9 Short expression bodies

A function or method whose body is one `return` can be written `=> expression;`.

```nz output="16 0 8 4 X Green 0 0 True\n"
package main;

import std.io.Console;

enum Color { Red, Green }

class P {
    int X;
    public int Twice() => X * 2;
}

int Square(int n) => n * n;

void Main(string[] args) {
    P p = new P();
    int zero = default;
    long big = default(long);
    string? s = default;
    Console.WriteLine($"{Square(4)} {p.Twice()} {sizeof(long)} {sizeof(Color)} {nameof(p.X)} {nameof(Color.Green)} {zero} {big} {s == null}");
}
```

## 3.10 Higher-order functions

A function can take a function value (`Func<...>`, `Action<...>`, `Predicate<T>`, `Comparison<T>` or `Converter<TInput, TOutput>`) and a lambda or a function name can be passed for it. The collections take them too: `List<T>` has `Sort(Comparison<T>)`, `ForEach`, `Find`, `FindLast`, `FindIndex`, `FindLastIndex`, `FindAll`, `Exists`, `TrueForAll`, `RemoveAll` and `ConvertAll`, `HashSet<T>` has `RemoveWhere`, and `Array` has the same searching, ordering and converting members as static methods (`Array.Sort(a, cmp)`, `Array.Find(a, p)`).

```nz output="25 4 3,1,2 2 c-b-a-\n"
package main;

import std.io.Console;
import std.collections.List;
import std.collections.Array;

void Main(string[] args) {
    List<int> xs = [3, 1, 2, 4];
    int total = 0;
    xs.ForEach(x => total += x * x);
    int evens = xs.RemoveAll(x => x % 2 == 0);
    xs.Sort((a, b) => b - a);
    List<string> names = ["a", "b", "c"];
    string[] words = Array.ConvertAll(names.ToArray(), s => s + "-");
    Array.Sort(words, (a, b) => string.CompareOrdinal(b, a));
    Console.WriteLine($"{total - 5} {evens + 2} {xs[0]},{xs[1]},{xs.Count} {xs.FindIndex(x => x == 1) + 1} {words[0]}{words[1]}{words[2]}");
}
```

A library member calls the function during the call and keeps nothing, so the lambda may change the variables around it (`total` above). The list the member works on is lent for the call: a lambda cannot change the same list (`xs.ForEach(x => xs.Add(x))` is an error); reading it is fine, and for a list inside a `Shared<T>` a change made through another handle during the call stops the program, while the program runs (a read is allowed). A function is lent each element: an owned element (a class object) can be read, not kept or moved. `Find`, `FindLast` and `FindAll` give copies of elements, so they are available for elements that are copied (numbers, `bool`, `char`, enums, structs of them, `string?`); for a `List<string>` or owned elements use `FindIndex` and the indexer, or `Exists`. The order of elements that compare equal after a `Sort` with a comparison is not specified.

## 3.11 Method references

This section is not written yet.

Outline:

- Method groups as delegates

## 3.12 Local functions

A function can be declared inside a block. It is written like any other function, the return type first, and it is visible only in that block. It can be called before or after its declaration, and it can read the variables around it.

```nz output="26\n120\n"
package main;

import std.io.Console;

int Total(int n) {
    int bonus = 10;

    int sum = add(n);

    int add(int x) {
        return x + bonus;
    }

    return sum + add(1);
}

int Fact(int n) {
    int fact(int k) {
        if (k <= 1) {
            return 1;
        }
        return k * fact(k - 1);
    }
    return fact(n);
}

void Main(string[] args) {
    Console.WriteLine($"{Total(5)}");
    Console.WriteLine($"{Fact(5)}");
}
```

`add` is called before its declaration, reads `bonus`, and `fact` calls itself. The rules:

- **Where.** A local function is a statement directly inside a braced block: a function body, the body of an `if` or a loop, a plain block, a `defer` block. It cannot be the unbraced body of an `if`, `else`, `while`, `for`, `foreach` or `using`, a `for` initializer, or follow `defer`:

```nz fragment error="a local function must be declared directly inside a block"
if (args.Length == 0) int g() { return 1; }
```

- **Visible only in its block.** The name is known from the start of the block to its end, and nowhere else:

```nz fragment error="unknown name"
{
    int g() { return 8; }
    Console.WriteLine($"{g()}");
}
int h = g();
```

- **What it can read.** The variables declared before it, in its own block or an enclosing one: locals, parameters, loop variables and `is` bindings. A variable or constant of the block declared later cannot be used inside the function (``cannot use local `x` before it is declared``, the block-scope rule of [Variables](01-basic.md#variables)). In a method it can also read the fields and properties of the object and call methods that do not change it. A local function that calls another local function reads what that one reads, too.
- **Calls before the declaration.** A function that reads no variable can be called anywhere in its block. One that does can be called only where every variable it reads is already declared; the same holds for a call from inside another local function:

```nz fragment error="which is not declared at this point"
int r = early();
int x = 1;
int early() { return x; }
```

- **Read-only.** A local function only reads what it captures. It cannot assign a captured variable (also with `++` or `+=`), pass it by `ref`, move it, write a field or element of it, or call a method on it that changes it. The same holds for `this` in a method:

```nz error="captured variables are read-only"
package main;

void Main(string[] args) {
    int count = 0;
    void bump() {
        count = count + 1;
    }
    bump();
}
```

  To change a variable, give the function a `ref` parameter and pass the variable at the call (as with any function, no keyword at the call):

```nz output="2 1\n7\n"
package main;

import std.io.Console;

void Swap(ref int a, ref int b) {
    int t = a;
    a = b;
    b = t;
}

void Main(string[] args) {
    int x = 1;
    int y = 2;
    void swapThem(ref int p, ref int q) {
        Swap(p, q);
    }
    swapThem(x, y);
    Console.WriteLine($"{x} {y}");
    void addTo(ref int total, int amount) {
        total = total + amount;
    }
    addTo(x, 5);
    Console.WriteLine($"{x}");
}
```

  Assigning a variable **outside** the local function is fine; each call reads the value the variable has at that call:

```nz fragment output="5\n"
int x = 1;
int f() { return x; }
x = 5;
Console.WriteLine($"{f()}");
```

- **Optionals are not narrowed inside.** The body is checked once, so a captured `T?` is not known to be non-null in it, even when every call is inside an `if (x != null)`. Test it inside the function. A call does not undo the caller's own checks, except through a `ref` parameter ([Null Safety](+/Null-Safety.md)):

```nz fragment error="possibly null"
string? s = null;
if (s != null) {
    int f() { return s.Length; }
    int n = f();
}
```

- **In a method.** The function may read fields and call methods that leave the object as it is; writing a field, or calling a method that changes the object, is the same error as for a captured variable:

```nz output="12\n"
package main;

import std.io.Console;

class Counter {
    int Step = 2;
    int Value = 10;

    int Next() {
        int plus() {
            return Value + Step;
        }
        return plus();
    }
}

void Main(string[] args) {
    Counter c = new Counter();
    Console.WriteLine($"{c.Next()}");
}
```

```nz error="a local function only reads `this`"
package main;

class Counter {
    int Value = 10;

    void Grow() {
        void bump() {
            Value = Value + 1;
        }
        bump();
    }
}

void Main(string[] args) { }
```

- **Names.** A local function is written with a lower-case first letter: it is visible only in its block, so the rule "an upper-case name is public" does not apply, and it takes no `public` or `private`. An upper-case name works but gives a warning (an error with `-Werror`):

```nz warning="visible only in its block"
package main;

void Main(string[] args) {
    int Helper(int x) {
        return x;
    }
    int r = Helper(1);
}
```

  Inside its block a local function hides a top-level function, a method or a built-in of the same name. It cannot reuse the name of a visible variable or of another local function, in either order, and its parameters cannot reuse the name of a variable of the enclosing function:

```nz output="-3\n"
package main;

import std.io.Console;

int Abs(int v) { return 100; }

void Main(string[] args) {
    int abs(int v) { return v; }
    Console.WriteLine($"{abs(-3)}");
}
```

```nz fragment error="already declared"
int n = 1;
int n() { return 2; }
```

- **Jumps.** `return` leaves the local function and has its own result type. `break` and `continue` cannot reach a loop of the enclosing function:

```nz fragment error="`break` is only valid inside a loop"
while (true) {
    int f() { break; return 1; }
    int r = f();
}
```

- **A value.** A local function can be used where a function value is expected: `Func<int> g = f;`, or `var g = f;` (the natural type of a function named alone). A local function that captures variables works on them while it is only passed to a call that does not keep it.

```nz output="1 1\n"
package main;

import std.io.Console;

void Main(string[] args) {
    int f() { return 1; }
    Func<int> g = f;
    var h = f;
    Console.WriteLine($"{g()} {h()}");
}
```

- **Not supported yet.** A generic local function (`int f<T>(T x) { ... }`) is reported as "not supported yet". A local function that captures variables and is kept as a value (stored or returned) is refused too; see [Lambdas borrow or copy what they capture](+/Ownership.md#lambdas-borrow-or-copy-what-they-capture). The type parameters of the enclosing function or class can be used inside a local function.

Previous: [2. Programming paradigms](02-programming-paradigms.md#2-programming-paradigms) · [Contents](README.md) · Next: [4. Struct](04-struct.md#4-struct)
