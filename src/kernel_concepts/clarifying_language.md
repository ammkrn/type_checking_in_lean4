# Clarifying language


## Type, Sort, Prop

`Prop` refers to `Sort 0`

`Type n` refers to `Sort (n+1)`

`Sort n` is how these are actually represented in the kernel, and can always be used.

The reason why `Type <N>` and `Prop` are sometimes used instead of always adhering to `Sort n` is that elements of `Type <N>` have certain important qualities and behaviors that are not observed by those of `Prop` and vice versa.

Example: elements of `Prop` can be considered for definitional proof irrelevance, while elements of `Type _` can use large elimination without needing to satisfy other tests.


## Level/universe and Sort

The terms "level" and "universe" are basically synonymous; they refer to the same kind of kernel object.

A small distinction that's sometimes made is that "universe parameter" may be implicitly restricted to levels that are variables/parameters. This is because "universe parameters" refers to the set of levels that parameterize a Lean declaration, which can only be identifiers, and are therefore restricted to identifiers. If this doesn't mean anything to you, don't worry about it for now. As an example, a Lean declaration may begin with `def Foo.{u} ..` meaning "a definition parameterized by the universe variable `u`", but it may not begin with `def Foo.{1} ..`, because `1` is an explicit level, and not a parameter.

On the other hand, `Sort _` is an expression that represents a level.

## Value vs. type

Expressions can be either values or types. Readers are probably familiar with the idea that `Nat.zero` is a value, while `Nat` is a type. An expression `e` is a value or "value level expression" if `infer e != Sort _`. An expression `e` is a type or "type level expression" if `infer(e) = Sort _`.


## Parameters vs. indices

The distinction between a parameter and index comes into play when dealing with inductive types. Roughly speaking, elements of a telescope that come before the colon in a declaration are parameters, and elements that come after the colon are indices:

```
      parameter ----v         v---- index
inductive Nat.le (n : Nat) : Nat → Prop
```

The distinction is non-negligible within the kernel, because parameters are fixed within a declaration, while indices are not.
