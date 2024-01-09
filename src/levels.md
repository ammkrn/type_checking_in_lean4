# Universe levels

This section will describe universe levels from an implementation perspective, and will cover what readers need to know when it comes to type checking Lean declarations. More in-depth treatment of their role in Lean's type theory can be found in [TPIL4](https://lean-lang.org/theorem_proving_in_lean4/dependent_type_theory.html#types-as-objects), or section 2 of [Mario Carneiro's thesis](https://github.com/digama0/lean-type-theory)

The syntax for universe levels is as follows:

```
Level ::= Zero | Succ Level | Max Level Level | IMax Level Level | Param Name
```

Properties of the `Level` type that readers should take note of are the existence of a partial order on universe levels, the presence of variables (the `Param` constructor), and the distinction between `Max` and `IMax`. 

`Max` simply constructs a universe level that represents the larger of the left and right arguments. For example, `Max(1, 2)` simplifies to `2`, and `Max(u, u+1)` simplifies to `u+1`. The `IMax` constructor represents the larger of the left and right arguments, *unless* the right argument simplifies to `Zero`, in which case the entire `IMax` resolves to `0`.

The important part about `IMax` is its interaction with the type inference procedure to ensure that, for example, `forall (x y : Sort 3), Nat` is inferred as `Sort 3`, but `forall (x y : Sort 3), True` is inferred as `Prop`.

## Partial order on levels

Lean's `Level` type is equipped with a partial order. The rather nice implementation below comes from Gabriel Ebner's Lean 3 checker [trepplein](https://github.com/gebner/trepplein/tree/master). While there are quite a few cases that need to be covered, the only complex matches are those relying on `cases`, which checks whether `x ≤ y` by examining whether `x ≤ y` holds when a parameter `p` is substituted for `Zero`, and when `p` is substituted for `Succ p`.

```
  leq (x y : Level) (balance : Integer): bool :=
    Zero, _ if balance >= 0 => true
    _, Zero if balance < 0 => false
    Param(i), Param(j) => i == j && balance >= 0
    Param(_), Zero => false
    Zero, Param(_) => balance >= 0
    Succ(l1_), _ => leq l1_ l2 (balance - 1)
    _, Succ(l2_) => leq l1 l2_ (balance + 1)

    -- descend left
    Max(a, b), _ => (leq a l2 balance) && (leq b l2 balance)

    -- descend right
    (Param(_) | Zero), Max(a, b) => (leq l1 a balance) || (leq l1 b balance)

    -- imax
    IMax(a1, b1), IMax(a2, b2) if a1 == a2 && b1 == b2 => true
    IMax(_, p @ Param(_)), _ => cases(p)
    _, IMax(_, p @ Param(_)) => cases(p)
    IMax(a, IMax(b, c)), _ => leq Max(IMax(a, c), IMax(b, c)) l2 balance
    IMax(a, Max(b, c)), _ => leq (simplify Max(IMax(a, b), IMax(a, c))) l2 balance
    _, IMax(a, IMax(b, c)) => leq l1 Max(IMax(a, c), IMax(b, c)) balance
    _, IMax(a, Max(b, c)) => leq l1 (simplify Max(IMax(a, b), IMax(a, c))) balance


  cases l1 l2 p: bool :=
    leq (simplify $ subst l1 p zero) (simplify $ subst l2 p zero)
    ∧
    leq (simplify $ subst l1 p (Succ p)) (simplify $ subst l2 p (Succ p))
```

## Equality for levels

The `Level` type recognizes equality by antisymmetry, meaning two levels `l1` and `l2` are equal if `l1 ≤ l2` and `l2 ≤ l1`.

# Implementation notes

Be aware that the exporter does not export `Zero`, but it is assumed to be the 0th element of `Level`.

For what it's worth, the implementation of `Level` does not have a large impact on performance, so don't feel the need to aggressively optimize here.
