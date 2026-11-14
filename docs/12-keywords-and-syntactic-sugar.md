# 12. Keywords and syntactic sugar

> NZ is still in development and is not stable. Anything in this documentation can change.

This chapter is not written yet.

Outline:

- Keywords ([0.3 Characteristics, Rules and Naming](00-introduction.md#03-characteristics-rules-and-naming))
- `=>` expression bodies ([3.9 Short expression bodies](03-functional-programming.md#39-short-expression-bodies))
- `sizeof` ([The `sizeof` operator](01-basic-operators-control-flow.md#the-sizeof-operator))
- `nameof` ([`nameof`](#nameof))
- `default` ([Default values (`default`)](01-basic.md#default-values-default))
- `var` ([1.3 Data items (variables, `var`, constants)](01-basic.md#13-data-items-variables-var-constants))
- `??` and `??=` ([1.5 Operators](01-basic-operators-control-flow.md#15-operators))
- `defer`
- `using` ([1.9 The `using` keyword](01-basic-null-safety-io.md#19-the-using-keyword))
- Collection expressions ([1.4.1 Array](01-basic-data-structures.md#141-array))
- Object initializers ([5.7 Constructor](05-object-oriented-programming.md#57-constructor))

## `nameof`

`nameof(a.b)` is the last name as a `string` constant; every part must exist and be visible, and naming something is not a use (a moved variable or a mutable static can be named).

In a constant, `nameof` can name only constants.

```nz output="count X 3\n"
package main;

import std.io.Console;

class P {
    int X;
}

void Main(string[] args) {
    int count = 3;
    P p = new P();
    Console.WriteLine($"{nameof(count)} {nameof(p.X)} {count}");
}
```

Previous: [11. Testing and debugging](11-testing-and-debugging.md#11-testing-and-debugging) · [Contents](README.md) · Next: [13. Regular expressions](13-regular-expressions.md#13-regular-expressions)
