# 11. Testing and debugging

> NZ is still in development and is not stable. Anything in this documentation can change.

## 11.1 Unit testing

`[Test]` marks a function as a test. The function must be a top-level `void name()`: no parameters, no type parameters, no result, and not `Main`. `[Test]` takes no arguments, can be written once per function, and cannot be put on a method, a type or any other declaration. Each of these is an error.

```nz error="a test must be"
package main;

[Test]
int Count() { return 1; }

void Main(string[] args) { }
```

A test is an ordinary function. It passes when it returns, and fails when it stops through `Assert` or `Abort` (see [6.1](06-exceptions-and-error-handling.md#61-creating-errors)). It calls the code under test and checks the result with `Assert`. This program checks with `nzc check` as it is, and its test also runs with `nzc test`:

```nz
package main;

int Add(int a, int b) {
    return a + b;
}

[Test]
void AddsNumbers() {
    Assert(Add(2, 3) == 5, "2 + 3");
}

[Test]
void AddsNegatives() {
    Assert(Add(-2, -3) == -5, "-2 + -3");
}

void Main(string[] args) { }
```

`nzc check`, `nzc build` and `nzc run` check the tests like any code, so an error in a test body is reported, but the tests are left out of the executable: a normal build contains neither a `[Test]` function nor a top-level function that only tests call, unless other code calls it.

**`nzc test <file.nz|folder> [--release] [-Werror]`** checks the program, builds an executable that holds only the tests, and runs each test in its own process, one after the other, in the order they are written. `Main` is not needed (and not run). Only the tests of the entry package are run; a `[Test]` in an imported package is checked but not run. A test has no name filter and no time limit, and tests do not run in parallel.

One process per test is deliberate: `Assert` and `Abort` end the process without cleanup, so a failing test ends only itself and the next one still runs. The report goes to standard output, one line per test (`test AddsNumbers ... ok`, `test AddsNegatives ... FAILED`), then a block for each failure with the exit code and what the test printed, then a last line `test result: 1 passed, 1 failed`. The output of a passing test is not shown. A failed `Assert` has exit code 1 and the message `nz: assertion failed: <message>`. A crash is shown by its exit code too; on Windows it is in hex, such as `exit code 0xC0000005`. The exit code of `nzc test` is 0 when every test passed, and 1 when one failed or the program has errors. With no `[Test]` at all it prints `nzc: no tests found` on standard error and exits with 0. `-Werror` works as for `check`.

**The leak rule.** The runner starts each test with `NZ_ALLOC_REPORT=1`. A test that returns normally but leaves owned objects or computed strings alive is a failure: `test Name ... FAILED (leaked 1 owned object(s))`, `(leaked 2 string(s))`, or both. Owned values are dropped when their scope ends and strings are freed with their last holder, so a correct program does not trip this rule; it is a safety net for the compiler itself (a cycle of `Shared<T>` handles is the one way to leak, see [Ownership](+/Ownership.md#shared-objects)). String literals never count. The test's own `defer` statements and drops run before the report, so they count as cleanup.

Not available yet: a test name filter, `[Test]` on methods, set-up and tear-down hooks, a time limit for a test, and an `Assert` that lets the test go on.

## 11.2 Debugging

This section is not written yet.

Outline:

- Debug and release builds
- Using a debugger
- Sanitizers

Previous: [10. Package manager](10-package-manager.md#10-package-manager) · [Contents](README.md) · Next: [12. Keywords and syntactic sugar](12-keywords-and-syntactic-sugar.md#12-keywords-and-syntactic-sugar)
