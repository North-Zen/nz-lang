# 24. Reflection and meta-programming

> NZ is still in development and is not stable. Anything in this documentation can change.

## 24.1 Attributes

An **attribute** is a note in square brackets in front of a declaration. It is compile-time data. The compiler knows `[Test]` (below) and the interop attributes of [23.1](23-serialization-and-interoperability.md#231-c-interop), `StructLayout`, `FieldOffset`, `DllImport` and `UnmanagedCallersOnly`, which come from `std.interop` and need `import std.interop.*;` or an import of the name. They may also be written with the `Attribute` suffix (`[DllImportAttribute(...)]`) or with the whole name (`[std.interop.DllImport(...)]`, which needs no import). Each built-in attribute is given once per declaration, only where it applies. Any other name is accepted by the parser, kept in the syntax tree (`nzc parse` shows it), warned about, and read by nothing yet.

**Where.** Before a top-level function, `class`, `struct`, `interface` or `enum`, and before a member: a method, a field, a property or an interface member. After the `///` comment of the declaration and before `public` or `private`. Anywhere else is a syntax error: on an enum variant, a parameter, a statement. A local function gets its own message:

```nz fragment error="attributes on local functions are not supported yet"
[Test] int Twice(int x) {
    return x * 2;
}
Console.WriteLine($"{Twice(2)}");
```

**Lists.** One list holds several attributes, `[A, B]`, and several lists can follow each other, `[A] [B]`. An empty list `[]` and a trailing comma in a list are errors. A name may be qualified, `[geo.Tag]`, which is never the built-in `Test`.

**Arguments.** `[Route("/users", method: "GET")]`: arguments are positional or named, and the two may be mixed. A named argument has two forms: `name: value` names a constructor parameter and `Name = value` sets a field or property. The built-in attributes say which is which: `EntryPoint` and `Pack` are fields, so `EntryPoint = "puts"` is right and `EntryPoint: "puts"` is an error; a named argument is given once. Each argument is a literal (a number may have a sign), a constant expression of `int`, `long` and `bool` values such as `60 * 60`, or, for `StructLayout`, a member of `LayoutKind`. A call, a variable, a string built with `+` or a real-number expression is an error.

**Targets.** `[return: Tag]` names what the attribute is for. The target is parsed, then refused: `attribute targets are not supported yet`.

```nz error="attribute targets are not supported yet"
package main;

[return: Tag]
string Name() { return "n"; }

void Main(string[] args) { }
```

**Unknown names.** A name that is not a built-in attribute (`Test`, and the interop attributes `StructLayout`, `FieldOffset`, `DllImport` and `UnmanagedCallersOnly`, which, need their import, `import std.interop.*;`, or their whole name, `[std.interop.StructLayout(...)]`; the `Attribute` suffix may be written) is a warning, `unknown attribute`, and `-Werror` turns it into an error. The warning is there because the attribute has no effect yet; how attributes will be read is still open.

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

## 24.2 Reflection

This section is not written yet.

Outline:

- `typeof` and `GetType()`
- `Type.Name`
- Reading attributes

## 24.3 Meta-programming

This section is not written yet.

Previous: [23. Serialization and interoperability](23-serialization-and-interoperability.md#23-serialization-and-interoperability) · [Contents](README.md) · Next: [25. Ecosystem and frameworks](25-ecosystem-and-frameworks.md#25-ecosystem-and-frameworks)
