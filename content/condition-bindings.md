---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:number
  - /sips/:title.html
stage: design
status: submitted
presip-thread: TODO
title: Bindings in `while` and `if` Conditions
---

**By: Matthias Berndt**

## History

| Date          | Version            |
|---------------|--------------------|
| Oct 14th 2026 | Initial Draft      |

## Summary

Scala 3 already allows the condition of a `while` loop to be an arbitrary block, including
local definitions:

```scala
while
  val line = in.readLine()
  line != null
do
  process(line) // error: Not found: line
```

However, the definitions in that block are not visible in the body of the loop. This
proposal extends their scope: definitions in the condition block of a `while` loop are
visible in the loop body. For consistency, the same applies to `if` expressions, whose
condition block definitions become visible in the `then` and `else` branches. The syntax
is unchanged.

This gives Scala a direct solution to the _loop-and-a-half_ problem (a loop whose exit
test sits in the middle of its body) without `var`s, duplicated code or `break`.
Since the condition is a full block, the head of the construct can contain any number of
definitions of any kind, including local functions, which makes the mechanism more
versatile than the comparable features of most other languages.

## Motivation

### The loop-and-a-half problem

Many loops need to do some work before they can decide whether to continue, and then use
the result of that work in the rest of the iteration. Reading input until a sentinel is
the textbook example:

```
loop
  line = readLine()
  exit if line == null
  process(line)
end
```

The problem is old. Knuth discusses it in _Structured Programming with go to Statements_
(1974), including Ole-Johan Dahl's proposal of a loop with the test in the middle
(`loop S; while B: T; repeat`). Soloway, Bonar and Ehrlich (_Cognitive Strategies and
Looping Constructs_, CACM 1983) found that novice programmers write such loops much more
reliably with a construct that matches this "read, test, process" structure. Roberts
(_Loop Exits and Structured Programming: Reopening the Debate_, SIGCSE 1995) made the same
point for teaching.

With a `while` loop in Scala today, the options are:

**1. Duplicate the first half of the loop.**

```scala
var line = in.readLine()
while line != null do
  process(line)
  line = in.readLine()
```

The read is written twice, and the two copies must be kept in sync. `line` has to be a
`var`, and it remains in scope after the loop.

**2. Assign a `var` in the condition block.**

```scala
var line: String = null
while
  line = in.readLine()
  line != null
do
  process(line)
```

This avoids the duplication, but needs a `var` with a dummy initial value, and again
leaks `line` into the enclosing scope.

**3. Use `boundary`/`break`.**

```scala
import scala.util.boundary, boundary.break

boundary:
  while true do
    val line = in.readLine()
    if line == null then break()
    process(line)
```

The loop condition is now meaningless, and the actual control flow is hidden in the
body.

Of course, the same can be expressed without a `while` loop, for example with iterators or
a tail-recursive function. But Scala has `while` loops, and this proposal is about them:
given that they are part of the language, would they be better with this change?

Interestingly, Scala 3 already has the syntax for a better solution. When `do`-`while`
was dropped, the recommended replacement was to put the body into the condition block:
`while { body; cond } do ()`. With the new control syntax, this generalizes to loops with
a test in the middle:

```scala
while
  val line = in.readLine()
  line != null
do
  process(line)
```

This is exactly Dahl's loop. It reads naturally and needs no `var`. The only thing
missing is that `line` is not in scope in the body.

### Versatility

Since the condition is an ordinary block, the first half of the loop is not limited to a
single value. It can contain any number of definitions of any kind: several `val`s,
pattern definitions, `lazy val`s, local functions, givens or imports. All of them are
available in the body:

```scala
while
  val token = lexer.next()
  val isOperator = operators.contains(token.text)
  def precedence = operators(token.text)
  token.kind != Kind.EOF
do
  if isOperator then pushOperator(token, precedence)
  else pushOperand(token)
```

Most of the corresponding features in other languages allow only a single declaration,
statement or pattern binding in the head of the construct (see
[Related work](#related-work)).

### `if` expressions

`if` expressions get the same rule as `while` loops: definitions in the condition block
are visible in the rest of the construct. This is useful whenever a value is computed only
in order to be tested and then used in the branches:

```scala
if
  val i = s.indexOf(' ')
  i >= 0
then s.substring(0, i)
else s
```

Other languages provide the same: C++17 added `if (init; cond)`, and Go has had
`if init; cond` from the start. In both, the variables introduced in the head of the `if`
are visible in all of its branches.

Both parts of the proposal also mirror what `for` comprehensions already do: definitions
in the head of a `for`, whether generators or value definitions such as `y = f(x)`, are
visible in its body.

## Proposed solution

### High-level overview

Definitions in the condition block of a `while` loop are visible in the loop body, and are
evaluated anew in every iteration:

```scala
while
  val (key, value) = readEntry()
  key.nonEmpty
do
  map(key) = value
```

Likewise, definitions in the condition block of an `if` expression are visible in both
branches:

```scala
def classify(s: String): String =
  if
    val n = s.toIntOption
    n.isDefined
  then s"number ${n.get}"
  else s"not a number: $s"
```

This includes the conditions of subsequent `else if`s:

```scala
if
  val parts = line.split('=')
  parts.length == 2
then Setting(parts(0), parts(1))
else if parts.length == 1 then Flag(parts(0))
else Invalid(line)
```

The definitions are not visible after the `while` or `if`.

### Specification

**Scoping rule.** If the condition of a `while` loop or an `if` expression is a block
`{ stats; cond }`, then the definitions and imports in `stats` are in scope

- for a `while` loop: in its body;
- for an `if` expression: in its `then` branch and in its `else` branch (including, if the
  `else` branch is itself an `if`, that `if`'s condition and branches).

The statements in the condition block are evaluated once per iteration of the `while`
loop, or once per evaluation of the `if`. Each iteration gets fresh instances of the
definitions, so closures created in the body capture the values of the current iteration.

As for any block, a local type defined in the condition block may not escape into the type
of the `if` expression.

**Legacy syntax.** This proposal does not change the meaning of the parenthesized syntax
inherited from Scala 2, as in `while ({ val x = f(); x > 0 }) g(x)`.

**Grammar.** The grammar is unchanged.

### Compatibility

**Source compatibility.** The change is incompatible in theory: a definition in the
condition block now shadows a definition of the same name from an enclosing scope in the
body or branches.

```scala
var i = 0
while { val i = 5; i > 0 } do
  println(i) // today: the outer `var i`; with this proposal: the inner `val i`
  ...
```

In practice, this is unlikely to matter, because all of the following would have to apply:

- The condition block of a `while` or `if` contains a definition. This is not common in
  the first place, because such definitions are of little use with the current scoping
  rules.
- The body or the branches refer to a definition of the same name from an enclosing scope.
- The type checker does not catch the change. In many cases, the two definitions have
  different types, and the program no longer compiles.

In addition, the compiler will emit a warning during a transitional phase whenever a name
in the body or the branches refers to a definition in the condition block that shadows a
definition from an enclosing scope.

**Binary and TASTy compatibility, macros.** The proposal does not add new kinds of trees,
and the condition of a `while` or `if` remains an ordinary expression. Instead, the
proof-of-concept implementation rewrites the construct into existing ones before type
checking:

```scala
while { stats; cond } do body
  ==>
while { stats; if cond then { body; true } else false } do ()

if { stats; cond } then thenp else elsep
  ==>
{ stats; if cond then thenp else elsep }
```

Binary and TASTy compatibility are therefore not affected. Macros, however, see the
rewritten trees. A macro that inspects a `while` or `if` whose condition block contains
definitions sees a different tree shape than before. Again, the impact is likely limited,
because definitions in condition blocks are of little use today.

### Other concerns

**Tooling.** Syntax-based tools such as Scalameta and scalafmt are not affected, since the
syntax is unchanged. Tools that perform their own name resolution, such as IDEs that don't
use the compiler for this, need to apply the new scoping rule. Tools based on the
presentation compiler (e.g. Metals) get it automatically.

**Implementation.** A proof-of-concept implementation is available as the experimental
language feature `scala.language.experimental.conditionBindings` (TODO: link).

## Related work

Several languages allow variables to be introduced in the condition of a loop or an
`if`, and to be used in the rest of the construct.

**C++.** C++ has allowed a declaration as the condition of `if`, `switch` and `while` since
C++98, as in `while (T x = f()) { ... }`. The declared variable is in scope in the body (and
in the `else` branch of an `if`). In a `while` loop, it is created anew in every iteration.
However, the declaration must itself be the condition. C++17 relaxed this for `if` and
`switch` with an _init-statement_ ([P0305R1](https://wg21.link/P0305R1)):
`if (auto it = m.find(k); it != m.end()) { ... }`. C++20 added an init-statement to the
range-based `for` loop ([P0614R1](https://wg21.link/P0614R1)):
`for (auto v = get(); auto& x : v) { ... }`. Both papers motivate the change with keeping
scopes tight. C++ has no init-statement for `while` (as of C++23). Note that the
init-statement of a classic `for` loop is evaluated only once, unlike the condition block
in this proposal.

**Go.** `if` and `switch` statements can start with a simple statement:
`if v, err := f(); err != nil { ... } else { ... }`. Its variables are in scope in all
branches, including `else if` and `else`. This is the closest analogue to the `if` part of
this proposal. Go's `for` has an init statement, but it is evaluated only once.

**Rust.** `if let` and `while let` bind variables from a pattern in the condition, and
_let chains_ (stable since
[Rust 1.88](https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/), 2024 edition) allow
mixing them with boolean conditions:
`if let Some(x) = a && x > 0 { ... }`. The bindings are visible in later parts of the
condition and in the body. Rust also allows a block as a condition,
`while { let x = f(); x > 0 } { ... }`, but just like in Scala today, `x` is not visible in
the body.

**Swift.** `if let x = f(), x > 0 { ... }`, `while let line = readLine() { ... }` and
`guard let` bind variables in the condition. The bindings are visible in the body, but not
in the `else` branch of an `if`.

**Python.** Assignment expressions ([PEP 572](https://peps.python.org/pep-0572/),
Python 3.8) were motivated to a large part by the loop-and-a-half problem:
`while (chunk := f.read(8192)): process(chunk)`. Since Python variables are function-scoped,
`chunk` remains visible after the loop.

**Perl and Ruby.** Both allow assignments in conditions. In Perl,
`while (my $line = <$fh>) { ... }` scopes `$line` to the loop. In Ruby,
`while line = gets ... end` works because assignment is an expression, but local variables
are method-scoped.

**C#.** C# 7.0 introduced variables declared in expressions, such as
`if (int.TryParse(s, out var n)) { ... }` and patterns like `if (o is string s) { ... }`.
In an `if` statement, these variables are in scope in the _enclosing_ block, not just
in the `if`
([announcement](https://devblogs.microsoft.com/dotnet/new-features-in-c-7-0/)).

**Java.** Pattern matching for `instanceof` (JEP 394, Java 16) binds variables in
conditions, with flow-sensitive scoping: `if (o instanceof String s && !s.isEmpty()) { ... }`.
In January 2024, a
[thread on amber-dev](https://mail.openjdk.org/pipermail/amber-dev/2024-January/008525.html)
discussed the reverse direction: making variables declared in a `do`-`while` body visible in
its condition. The discussion pointed out the same shadowing issue as in the
[Compatibility](#compatibility) section of this proposal.

**Kotlin.** In a `do`-`while` loop, variables declared in the body are visible in the
condition. `when (val x = f()) { ... }` binds the subject of a `when` expression.

**Ada.** `loop ... exit when cond; ... end loop;` places the exit test anywhere in the loop.

**Comparison.** C++ (declaration as condition, init-statement), Go (simple statement) and
Python (assignment expression) allow a single declaration or statement in the head of the
construct. Rust's let chains and Swift's condition lists allow several bindings, but each
one is a `let` binding of a pattern to an expression. With this proposal, the head is a
full block. It can contain any number of definitions of any kind, including local
functions, `lazy val`s, givens and imports, as well as arbitrary statements.

### References

- D. E. Knuth, _Structured Programming with go to Statements_, Computing Surveys 6(4), 1974.
- E. Soloway, J. Bonar, K. Ehrlich, _Cognitive Strategies and Looping Constructs: An
  Empirical Study_, Communications of the ACM 26(11), 1983.
- E. Roberts, _Loop Exits and Structured Programming: Reopening the Debate_,
  SIGCSE 1995.
