# Unsafe declarations

Lean's vernacular allows users to write declarations marked as `unsafe`, which are permitted to do things that are normally forbidden. For example, Lean permits the following definition:

```
  unsafe def y : Nat := y
```

Unsafe declarations are not exported[^note1], do not need to be trusted, and (for the record) are not permitted in proofs, even in the vernacular. Permitting unsafe declarations in the vernacular is still beneficial for Lean users, because it gives users more freedom when writing code that is used to produce proofs but doesn't have to be a proof in and of itself.

The aesop library provides us with an excellent real world example. [Aesop](https://github.com/leanprover-community/aesop) is an automation framework; it helps users generate proofs. At some point in development, the authors of aesop felt that the best way to express a certain part of their system was with a mutually defined inductive type, [seen here](https://github.com/leanprover-community/aesop/blob/69404390bdc1de946bf0a2e51b1a69f308e56d7a/Aesop/Tree/Data.lean#L375). It just so happens that this set of inductive type has an invalid occurrence of one of the types being declared within Lean's theory, and would not be permitted by Lean's kernel, so it needs to be marked `unsafe`.

Permitting this definition as an `unsafe` declaration is still a win-win. The Aesop developers were able to use Lean to write their library the way they wanted, in Lean, without having to call out to (and learn) a separate metaprogramming DSL, they didn't have to jump through hoops to satisfy the kernel, and users of aesop can still export and verify the proofs produced *by* aesop without having to verify aesop itself.

[^note1]: There's technically nothing preventing an unsafe declaration from being put in an export file (especially since the exporter is not a trusted component), but checks run by the kernel will prevent unsafe declarations from being added to the environment if they are actually unsafe. A properly implemented type checker would throw an error if it received an export file declaring the aesop library code described above.
