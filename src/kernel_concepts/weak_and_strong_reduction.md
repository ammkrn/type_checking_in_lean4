# Weak and strong reduction

The implementation details of Lean's reduction strategies is discussed in [another chapter](../type_checking/reduction.md); this section is specifically to clarify the difference between the general concepts of weak and strong reduction.

## Weak reduction

Weak reduction refers to reduction that stops at binders which do not have an argument applied to them. By binders, we mean lambda, pi, and let expressions.

For example, weak reduction can reduce `(fun (x y : Nat) => y + x) (0 : Nat)` to `(fun (y : Nat) => y + 0)`, but can do no further reduction.

When we say or 'weak head normal form reduction', or just reduction without specifically identifying it as 'strong', we're talking about weak reduction. Strong reduction just happens as a byproduct of applying weak reduction after we've opened a binder somewhere else. 

## Strong reduction

Strong reduction refers to reduction under open binders; when we run across a binder without an accompanying argument (like a lambda expression with no `app` node applying an argument), we can traverse into the body and potentially do further reduction by creating and substituting in a free variable. Strong reduction is needed for type inference and definitional equality checking. For type inference, we also need the ability to "re-close" open terms, replacing free variables with the correct bound variables after some reduction has been done in the body. This is not as simple as just replacing it with the same bound variable as before, because bound variables may have shifted, invalidating their old deBruijn index relative to the new rebuilt expression.

As with weak reduction, strong reduction can stil reduce `(fun (x y : Nat) => y + x) (0 : Nat)` to `(fun (y : Nat) => y + 0)`, and instead of getting stuck, it can continue by substituting `y` for a free variable, reducing the expression further to `((fVar id, y, Nat) + 0)`, and `(fvar id, y, Nat)`. 

As long as we keep the free variable information around _somewhere_, we can re-combine that information with the reduced `(fVar id, y, Nat)` to recreate `(fun (y : Nat) => bvar(0))`
