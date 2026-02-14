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

When both `-language:strictEquality` and `-Yexplicit-nulls` are enabled, it is currently impossible to compare a value of type `A | Null` against `null` using either `==` (because there is no `CanEqual[A | Null, Null]` instance) or `eq` (because `A | Null` is not necessarily an `AnyRef` subtype). This proposal introduces special handling for `== null` and `!= null` comparisons under strict equality: instead of requiring a `CanEqual[A, Null]` instance, the compiler should require `Null <:< A`, i.e. that `Null` is a subtype of the compared type. This allows null checks for nullable types like `Int | Null` while correctly rejecting them for non-nullable types like `Int`.

## Motivation

Scala 3 introduced two important compiler flags that improve type safety:

- **`-Yexplicit-nulls`**: Makes `Null` no longer a subtype of all reference types. A value that may be `null` must be explicitly typed as `T | Null`. The compiler performs flow typing so that after a `x == null` / `x != null` check, the type is narrowed accordingly.
- **`-language:strictEquality`** (multiversal equality): Requires a `CanEqual[A, B]` instance for `a == b` to compile, preventing nonsensical comparisons like `1 == "one"`.

Each feature works well in isolation, but they interact badly when combined. Consider:

```scala
//> using options -language:strictEquality -Yexplicit-nulls

def process(x: Int | Null): Int =
  if x == null then 0  // ERROR: Values of types Int | Null and Null cannot be compared with == or !=
  else x
```

This fails to compile because strict equality requires a `CanEqual[Int | Null, Null]` instance, and none exists.

### Workaround 1: Add a `CanEqual` instance

A user might try adding an instance for the specific type they need:

```scala
given CanEqual[Int | Null, Null] = CanEqual.derived
```

However, `CanEqual` is contravariant in both type parameters. This means `CanEqual[Int | Null, Null]` is a subtype of `CanEqual[Int, Null]` (because `Int <: Int | Null` and contravariance flips the relationship). As a result, providing this instance also allows null checks on the non-nullable type `Int`:

```scala
val x: Int = 42
if x == null then ???  // Compiles — but Int is never null!
```

This is already problematic for a single type, but the real issue is that to solve the general case, a user would need to provide a polymorphic instance:

```scala
given [A]: CanEqual[A | Null, Null] = CanEqual.derived
```

Since `A | Null` for an unconstrained `A` is equivalent to `Any`, this is equivalent to `CanEqual[Any, Null]`, which allows null checks for *every* type, completely defeating the purpose of strict equality for null checks.

### Workaround 2: Use `eq`

A user might try using reference equality:

```scala
if x.eq(null) then 0 else x
```

But `eq` is defined on `AnyRef`, and `Int | Null` is not an `AnyRef` subtype (since `Int` extends `AnyVal`), so this also fails to compile.

### The result

Users who want the combined safety of explicit nulls and strict equality are left with no ergonomic way to perform the most fundamental null check. This is a significant gap, especially since `== null` is already treated specially under explicit nulls for flow typing purposes.

The goal of this proposal is to allow `== null` checks for nullable types while preserving the strictness of multiversal equality for all other comparisons. Out of scope is any change to how `== null` behaves without strict equality, or any change to the `eq` method.

## Proposed solution

### High-level overview

When strict equality is enabled and the compiler encounters a comparison of the form `expr == null` or `expr != null`, it should not require a `CanEqual` instance. Instead, it should check whether `Null <:< A` holds, where `A` is the type of `expr`.

This means:

```scala
val x: Int | Null = ???
if x == null then 0 else x + 1   // OK: Null <:< (Int | Null)

val y: Int = 42
if y == null then 0 else y + 1   // ERROR: Null is not a subtype of Int
```

This aligns with the semantics of explicit nulls: if a type admits `null` as a value, you should be able to check for it. If it doesn't, the check is rejected — which is exactly what strict equality should enforce.

Flow typing continues to work as before. After `x == null` is checked, the compiler narrows `x` to `Int` in the else branch.

### Specification

Currently, when strict equality is enabled, the compiler checks for a comparison `a == b` that a `CanEqual[A, B]` or `CanEqual[B, A]` instance exists (where `A` and `B` are the types of `a` and `b` respectively).

This proposal adds a special case: when one side of the comparison is the literal `null` (i.e. has type `Null`), the compiler skips the `CanEqual` lookup and instead checks:

- For `a == null` or `a != null`: verify that `Null <:< A` holds.
- For `null == a` or `null != a`: verify that `Null <:< A` holds.

If the subtype check succeeds, the comparison is allowed. If it fails, a compilation error is reported, e.g.:  
```
Values of types Int and Null cannot be compared with == or !=
because Null is not a subtype of Int
```

This check should be performed only when both strict equality and explicit nulls are enabled. When explicit nulls is not enabled, `Null` is a subtype of all reference types by default, so the check would always pass and is not useful.

When only strict equality is enabled (without explicit nulls), the existing behavior is preserved: `CanEqual` instances govern equality checks as before.

### Compatibility

This change is fully backward compatible in the following sense:

- **Binary and TASTy compatibility**: This is a purely compile-time check. No changes to generated bytecode or TASTy format are required. The runtime semantics of `==` are unchanged.
- **Source compatibility**: Code that currently compiles will continue to compile. The change only *relaxes* a restriction: comparisons that were previously rejected (no `CanEqual` instance) will now be accepted if `Null <:< A` holds. No currently valid program changes its meaning.

### Feature Interactions

- **Flow typing under explicit nulls**: This proposal complements flow typing. After `if x == null`, the compiler already narrows the type. This proposal simply ensures the comparison is allowed in the first place under strict equality.
- **`CanEqual` derivation**: Users who have manually added `CanEqual[..., Null]` instances as workarounds can remove them. Their code will continue to compile because the new subtype check will accept the comparison.
- **`eq` / `ne`**: This proposal does not change `eq`/`ne`. Those remain methods on `AnyRef` and are not affected.
- **`derives CanEqual`**: The proposed change does not affect how `CanEqual` is derived for user-defined types. It only adds a special case for null literal comparisons.

### Other concerns

The implementation should be localized to the type-checking phase that handles equality comparisons under strict equality. It adds a single branch: when one operand is `null`, check `Null <:< A` instead of looking up `CanEqual`.

### Open questions

1. Should the `Null <:< A` check also apply when only strict equality is enabled (without explicit nulls)? Without explicit nulls, `Null <: AnyRef`, so the check would allow `refValue == null` but reject `intValue == null`. This could be useful but changes existing strict-equality-only behavior.

2. Should the error message for rejected null comparisons be specialized (e.g., "Cannot compare non-nullable type Int to null") or reuse the existing strict equality error format?

## Alternatives

### Alternative 1: Add a polymorphic `CanEqual[A | Null, Null]` to the standard library

One approach is to provide a built-in polymorphic instance:

```scala
given [A]: CanEqual[A | Null, Null] = CanEqual.derived
```

However, for an unconstrained type parameter `A`, `A | Null` is equivalent to `Any`, making this equivalent to `CanEqual[Any, Null]`. This allows null checks even for non-nullable types, defeating the purpose. Even a monomorphic instance like `CanEqual[Int | Null, Null]` is problematic because contravariance means it also satisfies `CanEqual[Int, Null]`, allowing `42 == null`.

**Pros**: Simple, no compiler changes needed.
**Cons**: Too permissive. Cannot prevent null checks on non-nullable types.

### Alternative 2: Use a type class specifically for null checks

Instead of special-casing `== null` in the compiler, a new type class `NullCheckable[A]` could be introduced, with instances only for types where `Null <:< A`.

**Pros**: More principled from a type class perspective.
**Cons**: Adds complexity (a new type class in the standard library), and the interaction with `==` would still require compiler support.

### Alternative 3: Do nothing, recommend `asInstanceOf` or `@unchecked`

Users can work around the issue with casts:

```scala
if x.asInstanceOf[AnyRef].eq(null) then 0 else x
```

**Pros**: No language changes needed.
**Cons**: Extremely unergonomic, unsafe, and defeats the purpose of the type safety features.

## Related work

- [Scala 3 Reference: Explicit Nulls](https://docs.scala-lang.org/scala3/reference/experimental/explicit-nulls.html)
- [Scala 3 Reference: Multiversal Equality](https://docs.scala-lang.org/scala3/reference/contextual/multiversal-equality.html)
- Kotlin takes a different approach: its type system distinguishes nullable and non-nullable types (`Int?` vs `Int`), and `== null` is always allowed for nullable types without any special type class mechanism.
- Swift similarly allows `== nil` checks for `Optional` types without additional ceremony.
- The Pre-SIP discussion that led to this proposal: TODO (link to contributors.scala-lang.org thread)

## FAQ

N/A so far.
