
# Trust

A big part of Lean's value proposition is the ability to construct mathematical proofs, including proofs about program correctness. A common question from users is how much trust, and in what exactly, is involved in trusting Lean.

An answer to this question has two parts: what users need to trust in order to trust proofs in Lean, and what users need to trust in order to trust executable programs obtained by compiling a Lean program. 

Concretely, the distinction is that proofs (which includes statements about programs) and uncompiled programs can be expressed directly in Lean's kernel language and checked by an implementation of the kernel. They do not need to be compiled to an executable, therefore the trust is limited to whatever implementation of the kernel they're being checked with, and the Lean compiler does not become part of the trusted code base.

Trusting the correctness of compiled Lean programs requires trust in Lean's compiler, which is separate from the kernel and is not part of Lean's core logic. There is a distinction between trusting _statements about programs_ in Lean, and trusting _programs produced by the Lean compiler_. Statements about Lean programs are proofs, and fall into the category that only requires trust in the kernel. Trusting that proofs about a program _extend to the behavior of a compiled program_ brings the compiler into the trusted code base.

**NOTE**: Tactics and other metaprograms, even tactics that are compiled, do *not* need to be trusted _at all_; they are untrusted code which is used to produce kernel terms for use by something else. A proposition `P` can be proved in Lean using an arbitrarily complex compiled metaprogram without expanding the trusted code base beyond the kernel, because the metaprogram is required to produce a proof expressed in Lean's kernel language.

+ These statements hold for proofs that are [exported](./export_format.md). To satisfy more ~~pedantic~~ vigilant readers, this does necessarily entail some degree of trust in, for example, the operating system on the computer used to run the exporter and verifier, the hardware, etc.

+ For proofs that are not exported, users are additionally trusting the elements of Lean outside the kernel (the elaborator, parser, etc.).

## An more itemized list

A more itemized description of the trust involved in Lean 4 comes from a post by Mario Carneiro on the Lean Zulip. 

> In general:
> 
> 1. You trust that the lean logic is sound (author's note: this would include any kernel extensions, like those for Nat and String)
> 
> 2. If you didn't prove the program correct, you trust that the elaborator has converted your input into the lean expression denoting the program you expect. 
> 
> 3. If you did prove the program correct, you trust that the proofs about the program have been checked (use external checkers to eliminate this)
> 
> 4. You trust that the hardware / firmware / OS software running all of these things didn't break or lie to you
> 
> 5. (When running the program) You trust that the hardware / firmware / OS software faithfully executes the program according to spec and there are no debuggers or magnets on the hard drive or cosmic rays messing with your output
>
> For compiled executables:
>
> 6. You trust that any compiler overrides (extern / implemented_by) do not violate the lean logic (i.e. the model matches the implementation)
>
> 7. You trust the lean compiler (which lowered the lean code to C) to preserve the semantics of the program
>
> 8. You trust clang / LLVM to convert the C program into an executable with the same semantics

The first set of points applies to both proofs and compiled executables, while the second set applies specifically to compiled executable programs.

## Trust for external checkers

1. You're still trusting Lean's logic is sound.

2. You're trusting that the developers of the external checker properly implemented the program.

3. You're trusting the implementing language's compiler or interpreter. If you run multiple external checkers, you can think of them as circles in a venn diagram; you're trusting that the part where the circles intersect is free of soundness issues.

4. For the Nat and String kernel extensions, you're probably trusting a bignum library and the UTF-8 string type of the implementing language.

The advantages of using external checkers are:

+ Users can check their results with something that is completely disjoint from the Lean ecosystem, and is not dependent on any parts of Lean's code base.

+ External checkers can be written to take advantage of mature compilers or interpreters.

+ For kernel extensions, users can cross-check the results of multiple bignum/string implementations.

+ Using the export feature is the only way to get out of trusting the parts of Lean outside the kernel, so there's a benefit to doing this even if the export file is checked by something like [lean4lean](https://github.com/digama0/lean4lean/tree/master). Users worried about fallout from misuse of Lean's metaprogramming features are therefore encouraged to use the export feature.
