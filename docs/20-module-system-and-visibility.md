# 20. Module system and visibility rules

> NZ is still in development and is not stable. Anything in this documentation can change.

## 20.1 Packages

A **package** is a folder. All the `.nz` files directly inside one folder are the files of one package; a subfolder is another package. A program is the package of the folder you give to `nzc`, plus every package it imports.

```bash
nzc run examples/packages/app
```

The program in `examples/packages/` has two packages:

```text
examples/packages/
  app/main.nz          package app;
  geo/point.nz         package geo;
  geo/distance.nz      package geo;
```

```text
// examples/packages/app/main.nz
package app;

import std.io.Console;
import geo.Point;
import geo.{Origin, Distance2};

void Main(string[] args) {
    Point a = Origin();
    Point b = new Point();
    b.X = 3;
    b.Y = 4;
    Console.WriteLine($"squared distance = {Distance2(a, b)}");
}
```

```text
// examples/packages/geo/point.nz
package geo;

public class Point {
    int X;
    int Y;
}

public Point Origin() {
    return new Point();
}
```

```text
// examples/packages/geo/distance.nz
package geo;

int square(int n) {
    return n * n;
}

public int Distance2(in Point a, in Point b) {
    return square(b.X - a.X) + square(b.Y - a.Y);
}
```

It prints `squared distance = 25`. The `.nz` files of a package are read in order of their file names.

### The `package` line

Every file starts with `package <name>;`. In folder mode the name of a package is its folder: `geo/` is `geo`. A file whose `package` line says something else gets a warning, and the folder name is used:

```text
Warning: package `application` differs from its folder `app`; the folder name is used
```

A folder name must be a plain identifier (ASCII letters, digits and `_`, not starting with a digit), and not a reserved word. A folder called `my-app`, `2app`, `my.app` or `class` cannot be a package, for example ``folder `my-app` is not a valid package name``, and for a reserved word ``folder `class` is not a valid package name: `class` is a reserved word``. The first segment `std` is reserved for the standard library, so no package of yours can be called `std` (``the package name `std` is reserved for the standard library``).

### One namespace per package

The files of a package share one namespace. A function or type declared in `point.nz` is visible in `distance.nz` without an import (`Point` in `Distance2` above), and two files cannot declare the same name: the second is `duplicate declaration`. Visibility follows 2.4, and at the top level **private means visible in the whole package**: `square` above can be called from `point.nz` too, but not from `app`. Only what is public (a function that starts with an uppercase letter, a type written `public`) can be imported by another package.

Imports belong to one file. A file that does not write `import std.io.Console;` cannot use `Console`, even when another file of its package does.

### Running a folder or a file

`nzc check`, `nzc build` and `nzc run` take a `.nz` file or a folder.

- **A folder** is the entry package, and it needs a `Main`: `void Main()`, `void Main(string[] args)`, `int Main()` or `int Main(string[] args)` (an `int` result is the exit status; any other return type, such as a `Result`, is refused). A folder without `Main`, such as a library, fails with `no entry point`, for `check` as well. `nzc build app` writes the executable into the package's `bin` folder, named after the package folder: `app/bin/app` (`app\bin\app.exe` on Windows), wherever you build from, like `dotnet build`. The `bin` folder is created when missing. `-o` names another executable instead. An error in the program is reported before any problem with the output path; if a folder already has the executable's name, the build stops with ``cannot write the executable `app/bin/app`: a folder of that name exists; use -o``. A single file, `nzc build hello.nz`, still leaves `hello` in the current folder.
- **A file** is the entry package on its own. Its `package` line is its name, and no folder name check is made. No other file of its folder is read, unless the program imports the file's own package (`package app;` in `app/main.nz` with `import app.Helper;`): then every other `.nz` file of that folder is read as part of the package, so a broken file there fails the build.

Imported packages are found from a **source root**. By default it is the folder above the entry folder: `import geo.Point;` loads the folder `geo` next to `app`, and `import a.b.C;` loads `<root>/a/b/`. When the `package` line of the entry package is dotted and the folder path ends with that name (the folder `src/app/cli` with `package app.cli;`), the source root is the path before the name (`src`), the package is `app.cli`, and no warning is given. For a file, the same rule is applied to the folder that holds it. There is no option to set the root.

A package that cannot be found (``package `geo` not found: no folder `<root>/geo` ``, where the message shows the whole path) or a folder with no `.nz` files is an error at the import that asked for it.

## 20.2 Imports

### Import forms

| Form | Brings in |
|---|---|
| `import geo.Point;` | `Point` |
| `import geo.Point as P;` | `Point`, under the name `P` |
| `import geo.{Point, Origin};` | each name of the group |
| `import geo.*;` | every public name of `geo` (a glob) |

An import names a package and a declaration (`import geo;` is an error). The standard library is a set of built-in packages (`std.io`, `std.collections`, see [Appendix A](appendix-a-standard-library.md#appendix-a-standard-library-reference)) and takes the same four forms. An alias works on one name only: `import geo.* as G;` is an error.

```nz output="3\n"
package main;

import std.io.Console;
import std.collections.List as Seq;

void Main(string[] args) {
    Seq<int> numbers = new Seq<int>();
    numbers.Add(1);
    numbers.Add(2);
    numbers.Add(3);
    Console.WriteLine($"{numbers.Count}");
}
```

A name used with its package (`geo.Point p;`) without importing it is an error that suggests the import: ``unknown type `geo.Point`; import it with `import geo.Point;` ``.

**Globs.** A name has a rank in each file. Declarations of the file's own package (in any file), explicit imports and aliases are the highest rank, and they win over a glob: `import geo.*;` never hides your own `Point`, and `import geo.Point;` beats a glob that also holds a `Point`. A glob comes next. When two globs bring in different declarations with one name, there is no error at the imports; the error appears where the name is used, and it names the packages:

```text
`Thing` is ambiguous: it is declared in `p` and `q`, both imported with `.*`; import it by name or with an alias
```

An alias or an import by name fixes it (`import p.Thing;`). Importing the same package with `.*` twice, or importing the same name twice, is an error.

An import of the package of the file itself has no effect, since its names are already visible; it is a warning (`this import names the package of this file and has no effect`), and a glob of the own package brings in nothing. An alias that differs from the name does work (`import app.Helper as H;`), as a second name.

```nz warning="has no effect"
package main;

import main.Helper;

class Helper { }

void Main(string[] args) { }
```

An import of a name that the file's own package also declares is an error at the import (``the import of `geo.Point` clashes with `Point` declared in package `app` ``).

### Import cycles

Packages cannot import each other, directly or through other packages. The compiler reports one error per group of packages that import each other in a circle, naming the shortest cycle:

```text
Error: import cycle: `p` imports `q`, which imports `p`; packages cannot import each other
```

Importing a package of the standard library takes no part in a cycle.

## 20.3 Access modifiers

The **first letter of the name** decides the visibility of functions, methods, fields and properties:

| Spelling | Meaning |
|---|---|
| `Add` | visible from outside |
| `add` or `_add` | local |

The keywords `public` and `private` may be written as well. An explicit keyword wins over the letter, except that `public` on a name that starts with a lowercase letter is an error.

A `class`, `struct`, `interface` or `enum` is private (visible inside its package) unless it is written `public`. Local variables are not affected.

```nz error="must start with an uppercase letter"
package main;

public int add() { return 1; }

void Main(string[] args) { }
```

```nz error="local"
package main;

class Counter {
    int count;
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.count;
}
```

```nz
package main;

class Counter {
    int Count;
}

void Main(string[] args) {
    var c = new Counter();
    int n = c.Count;
}
```

Previous: [19. Design patterns](19-design-patterns.md#19-design-patterns) · [Contents](README.md) · Next: [21. Networking fundamentals](21-networking-fundamentals.md#21-networking-fundamentals)
