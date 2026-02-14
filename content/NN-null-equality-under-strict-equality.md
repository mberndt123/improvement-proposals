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

When both `-language:strictEquality` and `-Yexplicit-nulls` are enabled, it is currently impossible to compare a value of type `A | Null` against `null` using either `==` (because there is no `CanEqual[A | Null, Null]` instance) or `eq` (because `A | Null` is not necessarily an `AnyRef` subtype). Similarly, `case null` clauses in pattern matching are rejected under these combined flags. This proposal introduces special handling for null comparisons and null patterns under strict equality: when no `CanEqual` instance is found and one side of the comparison is `null` (or a `case null` pattern is used), the compiler falls back to checking `Null <:< A` instead of rejecting the code outright. This allows null checks for nullable types like `Int | Null` while correctly rejecting them for non-nullable types like `Int`.

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

The same problem affects pattern matching with `case null`:

```scala
//> using options -language:strictEquality -Yexplicit-nulls

def process(x: String | Null): String = x match
  case null => ""       // ERROR under strict equality
  case s    => s
```

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

The goal of this proposal is to allow `== null` checks and `case null` patterns for nullable types while preserving the strictness of multiversal equality for all other comparisons. Out of scope is any change to how `== null` behaves without strict equality, or any change to the `eq` method.

## Proposed solution

### High-level overview

When strict equality is enabled and the compiler encounters a comparison of the form `expr == null` or `expr != null`, or a `case null` clause in a pattern match, it should first attempt to find a `CanEqual` instance as usual. If no instance is found, the compiler falls back to checking whether `Null <:< A` holds, where `A` is the type of `expr` (or the scrutinee type for pattern matching).

This means:

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

This aligns with the semantics of explicit nulls: if a type admits `null` as a value, you should be able to check for it. If it doesn't, the check is rejected — which is exactly what strict equality should enforce.

Flow typing continues to work as before. After `x == null` is checked, the compiler narrows `x` to `Int` in the else branch.

### Specification

Currently, when strict equality is enabled, the compiler checks for a comparison `a == b` that a `CanEqual[A, B]` or `CanEqual[B, A]` instance exists (where `A` and `B` are the types of `a` and `b` respectively). If no such instance is found, the comparison is rejected.

This proposal modifies the fallback behavior when one side of the comparison is `null` (i.e. has type `Null`), or when a `case null` pattern is used in a match expression:

1. The compiler first attempts to resolve a `CanEqual[A, Null]` or `CanEqual[Null, A]` instance as it does today.
2. If no `CanEqual` instance is found, and one operand has type `Null`, the compiler performs a subtype check: verify that `Null <:< A` holds (where `A` is the type of the other operand).
3. If the subtype check succeeds, the comparison is allowed.
4. If the subtype check also fails, a compilation error is reported.

For pattern matching, the same logic applies to `case null` clauses:

1. The compiler first attempts to resolve a `CanEqual` instance between the scrutinee type and `Null`.
2. If no instance is found, the compiler checks whether `Null <:< S` holds, where `S` is the type of the scrutinee.
3. If the subtype check succeeds, the `case null` clause is allowed.
4. If it fails, the clause is rejected.

This preserves full backward compatibility: any code that already compiles via a `CanEqual` instance continues to use that path. The `Null <:< A` check is only a fallback for cases where no instance exists.

### Compatibility

This change is fully backward compatible in the following sense:

- **Binary and TASTy compatibility**: This is a purely compile-time check. No changes to generated bytecode or TASTy format are required. The runtime semantics of `==` and pattern matching are unchanged.
- **Source compatibility**: Code that currently compiles will continue to compile unchanged. The `CanEqual` lookup is performed first, so existing instances are still used. The change only *relaxes* a restriction: comparisons and patterns that were previously rejected (no `CanEqual` instance) will now be accepted if `Null <:< A` holds. No currently valid program changes its meaning.

### Feature Interactions

- **Flow typing under explicit nulls**: This proposal complements flow typing. After `if x == null`, the compiler already narrows the type. This proposal simply ensures the comparison is allowed in the first place under strict equality. Similarly, after `case null =>`, the scrutinee type is narrowed in subsequent cases.
- **`CanEqual` derivation**: Users who have manually added `CanEqual[..., Null]` instances as workarounds can remove them, but their code will continue to work if they don't — the `CanEqual` lookup happens first, so existing instances are still found and used.
- **`eq` / `ne`**: This proposal does not change `eq`/`ne`. Those remain methods on `AnyRef` and are not affected.
- **`derives CanEqual`**: The proposed change does not affect how `CanEqual` is derived for user-defined types. It only adds a fallback for null comparisons and null patterns.

### Other concerns

The implementation should be localized to the type-checking phase that handles equality comparisons and pattern match exhaustivity under strict equality. It adds a fallback branch: when no `CanEqual` instance is found and one operand is `null` (or a `case null` pattern is encountered), check `Null <:< A` before reporting an error.

### Open questions

1. Should the error message for rejected null comparisons be specialized (e.g., "Cannot compare non-nullable type Int to null") or reuse the existing strict equality error format?

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
