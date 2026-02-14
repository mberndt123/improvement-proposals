---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:title.html
  - /sips/:number
stage: pre-sip
status: under-review
presip-thread: https://contributors.scala-lang.org/t/strictequality-with-explicit-nulls-do-not-work-well-together/7050
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

When the compiler encounters `expr == null`, `expr != null`, or `case null`, it first tries to find a `CanEqual` instance as usual. If none is found **and** one side is `null`, it falls back to checking `Null <:< A`:

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

Flow typing is unaffected — after a null check, the compiler still narrows the type.

### Specification

Currently, strict equality rejects `a == b` when no `CanEqual[A, B]` or `CanEqual[B, A]` instance exists.

This proposal adds a fallback for null comparisons:

1. Attempt `CanEqual` resolution as usual.
2. If resolution fails **and** one operand has type `Null` (or a `case null` pattern is involved):
   - Check `Null <:< A` where `A` is the type of the other operand (or the scrutinee).
   - If the subtype check succeeds, allow the comparison.
   - If it fails, report an error.

This preserves all existing `CanEqual`-based behavior. User-defined instances still take priority.

### Compatibility

**Binary / TASTy**: No changes to code generation or TASTy format. The change is purely in the type checker.

**Source**: Strictly more permissive — code that compiled before continues to compile. Code that was previously rejected (null checks on nullable types under strict equality) now compiles. Code that *should* be rejected (null checks on non-nullable types) remains rejected.

### Feature Interactions

- **Flow typing** (`-Yexplicit-nulls`): Works unchanged. The null check enables type narrowing as before.
- **`CanEqual` instances**: User-provided instances take priority. The fallback only activates when no instance is found.
- **`-Yexplicit-nulls` disabled**: When explicit nulls is off, `Null <: AnyRef` holds, so the subtype check passes for all reference types. This matches the existing (pre-explicit-nulls) behavior.
- **Strict equality disabled**: No change; `== null` is always allowed without strict equality.

### Other concerns

The implementation should be a small, localized change in the compiler's equality checking logic (likely in `TypeComparer` or the `checkCanEqual` method in `Typer`).

## Alternatives

### Alternative 1: Provide `CanEqual[A | Null, Null]` in the standard library

The standard library could ship a built-in:

```scala
given [A]: CanEqual[A | Null, Null] = CanEqual.derived
```

However, due to contravariance this is equivalent to `CanEqual[Any, Null]`, which allows `42 == null` — exactly the kind of comparison strict equality should prevent.

## Related work

- [Pre-SIP discussion](https://contributors.scala-lang.org/t/strictequality-with-explicit-nulls-do-not-work-well-together/7050)
- [Scala 3 Reference: Explicit Nulls](https://docs.scala-lang.org/scala3/reference/experimental/explicit-nulls.html)
- [Scala 3 Reference: Multiversal Equality](https://docs.scala-lang.org/scala3/reference/contextual/multiversal-equality.html)
- Kotlin uses a similar approach: its type system tracks nullability (`T?`), and `== null` checks are always allowed on nullable types without special opt-in.

## FAQ

This section will be updated as discussions progress.