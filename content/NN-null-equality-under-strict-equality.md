---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:title.html
  - /sips/:number
stage: pre-sip
status: under-review
presip-thread: https://contributors.scala-lang.org/t/TODO
title: Null equality checks under strict equality
---

**By: mberndt123**

## History

| Date          | Version       |
|---------------|---------------|
| Feb 14th 2026 | Initial Draft |

## Summary

When both `-language:strictEquality` and `-Yexplicit-nulls` are enabled, comparing `A | Null` against `null` via `==` fails (no `CanEqual` instance) and `eq` also fails (`A | Null` is not necessarily `AnyRef`). The same applies to `case null` in pattern matching. This proposal adds a fallback: when no `CanEqual` instance is found for a null comparison, the compiler checks `Null <:< A` instead. This allows null checks for nullable types while rejecting them for non-nullable types.

## Motivation

Scala 3 offers two complementary type-safety flags:

- **`-Yexplicit-nulls`**: `Null` is no longer a subtype of reference types; nullable values must be typed `T | Null`. Flow typing narrows the type after null checks.
- **`-language:strictEquality`**: `a == b` requires a `CanEqual[A, B]` instance.

They interact badly when combined:

```scala
//> using options -language:strictEquality -Yexplicit-nulls

def process(x: Int | Null): Int =
  if x == null then 0  // ERROR: no CanEqual[Int | Null, Null]
  else x


def process2(x: String | Null): String = x match
  case null => ""       // ERROR: same reason
  case s    => s
```

### Workaround 1: Add a `CanEqual` instance

```scala
given CanEqual[Int | Null, Null] = CanEqual.derived
```

`CanEqual` is contravariant, so this also satisfies `CanEqual[Int, Null]`, allowing `42 == null`. A polymorphic instance `given [A]: CanEqual[A | Null, Null]` is equivalent to `CanEqual[Any, Null]`, defeating strict equality entirely.

### Workaround 2: Use `eq`

```scala
if x.eq(null) then 0 else x
```

`eq` is on `AnyRef`; `Int | Null` is not an `AnyRef` subtype, so this fails too.

### The result

There is no ergonomic way to null-check a nullable value when both flags are enabled. This proposal fills that gap for `== null`, `!= null`, and `case null`, without weakening strict equality for other comparisons.

## Proposed solution

### High-level overview

When the compiler encounters `expr == null`, `expr != null`, or `case null` under strict equality, it first looks for a `CanEqual` instance as usual. If none is found, it falls back to checking `Null <:< A` (where `A` is the type of `expr` or the scrutinee). If that holds, the comparison/pattern is allowed; otherwise it is rejected.

```scala
val x: Int | Null = ???
if x == null then 0 else x + 1   // OK: Null <:< (Int | Null)

val y: Int = 42
if y == null then 0 else y + 1   // ERROR: Null is not a subtype of Int

val z: String | Null = ???
z match
  case null => ""                 // OK: Null <:< (String | Null)
  case s    => s
```

Flow typing continues to narrow types after the check as before.

### Specification

Under strict equality today, `a == b` requires `CanEqual[A, B]` or `CanEqual[B, A]`. This proposal adds a fallback when one operand is `null` (or a `case null` pattern is used):

1. Attempt `CanEqual` resolution as today.
2. If that fails and one side has type `Null`, check `Null <:< A` (where `A` is the other operand's type, or the scrutinee type for `case null`).
3. Accept if the subtype check passes; reject otherwise.

### Compatibility

- **Binary / TASTy**: Purely compile-time; no bytecode or TASTy changes.
- **Source**: Strictly relaxes restrictions — previously rejected code may now compile. No existing valid program changes meaning. Existing `CanEqual` instances are still found first.

### Feature Interactions

- **Flow typing**: Complementary — this proposal ensures the null check compiles; flow typing then narrows the type.
- **Existing `CanEqual` instances**: Still resolved first; the `Null <:< A` check is only a fallback.
- **`eq` / `ne`**: Unchanged.
- **`derives CanEqual`**: Unchanged.

### Other concerns

Implementation is localized to the equality type-checking phase: add a fallback branch when `CanEqual` resolution fails and one operand is `null`.

### Open questions

1. Should the error message for rejected null comparisons be specialized (e.g., "Cannot compare non-nullable type Int to null") or reuse the existing format?

## Alternatives

### Alternative 1: Stdlib `CanEqual[A | Null, Null]`

A polymorphic `given [A]: CanEqual[A | Null, Null]` is equivalent to `CanEqual[Any, Null]` — too permissive. Monomorphic instances leak via contravariance.

### Alternative 2: New `NullCheckable[A]` type class

More principled but adds stdlib complexity, and `==` integration still requires compiler changes.

### Alternative 3: Do nothing

Users can use `x.asInstanceOf[AnyRef].eq(null)` — unsafe and unergonomic.

## Related work

- [Explicit Nulls](https://docs.scala-lang.org/scala3/reference/experimental/explicit-nulls.html)
- [Multiversal Equality](https://docs.scala-lang.org/scala3/reference/contextual/multiversal-equality.html)
- Kotlin and Swift allow null/nil checks on nullable/optional types without additional ceremony.
- Pre-SIP discussion: TODO

## FAQ

N/A so far.