# What is the kernel?

The kernel is an implementation of Lean's logic in software; a computer program with the minimum amount of machinery required to construct elements of Lean's logical language and check those elements for correctness. The major components are:

+ A sort of names used for addressing.

+ A sort of universe levels.

+ A sort of expressions (lambdas, variables, etc.)

+ A sort of declarations (axioms, definitions, theorems, inductive types, etc.)

+ Environments, which are maps of names to declarations.

+ Functionality for manipulating expressions. For example bound variable substitution and substitution of universe parameters.

+ Core operations used in type checking, including type inference, reduction, and definitional equality checking.

+ Functionality for manipulating and checking inductive type declarations. For example, generating a type's recursors (elimination rules), and checking whether a type's constructors agree with the type's specification.

+ Optional kernel extensions which permit the operations above to be performed on nat and string literals.

The purpose of isolating a small kernel and requiring Lean definitions to be translated to a minimal kernel language is to increase the trustworthiness of the proof system. Lean's design allows users to interact with a full-featured proof assistant which offers nice things like robust metaprogramming, rich editor support, and extensible syntax, while also permitting extraction of constructed proof terms into a form that can be verified without having to trust the correctness of the code that implements the higher level features that makes Lean (the proof assistant) productive and pleasant to use.

In section 1.2.3 of the [_Certified Programming with Dependent Types_](http://adam.chlipala.net/cpdt/), Adam Chlipala defines what is sometimes referred to as the de Bruijn criterion, or de Bruijn principle.

> Proof assistants satisfy the “de Bruijn criterion” when they produce proof terms in small kernel languages, even when they use complicated and extensible procedures to seek out proofs in the first place. These core languages have feature complexity on par with what you find in proposals for formal foundations for mathematics (e.g., ZF set theory). To believe a proof, we can ignore the possibility of bugs during search and just rely on a (relatively small) proof-checking kernel that we apply to the result of the search.

Lean's kernel is small enough that developers can write their own implementation and independently check proofs in Lean by using an exporter[^1]. Lean's export format contains contains enough information about the exported declarations that users can optionally restrict their implementation to certain subsets of the full kernel. For example, users interested in the core functionality of inference, reduction, and definitional equality may opt out of implementing the functionality for checking inductive specifications.

In addition to the list of items above, external type checkers will also need a parser for [Lean's export format](./export_format.md), and a pretty printer, for input and output respectively. The parser and pretty printer are not part of the kernel, but they are important if one wants to have interesting interactions with the kernel.

[^1]: Writing your own type checker is not an afternoon project, but it is well within the realm of what is achievable for citizen scientists.
