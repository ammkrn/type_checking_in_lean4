# Future work and open issues

## File format

There is an open rfc [here](https://github.com/leanprover/lean4export/issues/3) about moving the export file format to json, which would be a major version change.

## Ensuring Nat/String are defined properly.

The Lean community has not yet settled on a particular solution for determining whether the declarations in an export file for `Nat`, `String`, and the operations covered by the relevant kernel extensions match those expected by extensions in a way that does not pull the exporter into the trusted code. 

An approach similar to that taken for `Eq` and the `Quot` declarations (defining them by hand within the type checker, then asserting they're the same) is not feasible due to the complexity of the fully elaborated terms for the supported binary operations on `Nat`.

## Improving Pollack consistency

Lean 4 offers very powerful facilities for defining custom syntax, macros, and pretty printer behaviors, and almost every aspect of Lean 4's internals is available to users. These elements of Lean's design were effective responses to real world feedback from the mathlib community during Lean 3's lifetime.

While these features were important factors in Lean's success as a tool for enabling large formalization efforts, they are also in tension with Lean4's Pollack consistency[^pollack], or lack thereof. Without replicating the macro and syntax extension capabilities in the pretty printer, type checkers cannot consistently read terms back to the user in a form that is recognizable. However, the idea of adding these features to a pretty printer is an unappealing expansion of the trusted code base. An alternative approach is to drop the pretty printer in favor of a trusted parser (ala metamath zero), but Lean's parser can be modified on the fly in userspace with custom syntax declarations.

As Lean matures and adoption increases, there is likely to be a push for progress in the development of techniques and practices that allow users to take advantage of Lean's extensibility while sacrificing the least degree of Pollack consistency.

## Forward reasoning

Existing type checkers implement a form of backward reasoning; an alternate strategy for type checking is to accept and check forward reasoning chains worked out by an external program, potentially allowing for an even simpler type checker.

[^pollack]: Freek Wiedijk. Pollack-inconsistency. Electronic Notes in Theoretical Computer Science, 285:85–100, 2012