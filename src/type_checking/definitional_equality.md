# Definitional equality

Definitional equality is implemented as a function that takes two expressions as input, and returns `true` if the two expressions are definitionally equal within Lean's theory, and `false` if they are not.

Within the kernel, definitional equality is important simply because it's a necessary part of type checking. Definitional equality is still an important concept for Lean users who do not venture into the kernel, because definitional equalities are comparatively nice to work with in Lean's vernacular; for any `a` and `b` that are definitionally equal, Lean doesn't need any prompting or additional input from the user to determine whether two expressions are equal.

There are two big-picture parts of implementing the definitional equality procedure. First, the individual tests that are used to check for different definitional equalities. For readers who are just interested in understanding definitional equality from the perspective of an end user, this is probably what you want to know.

Readers interested in writing a type checker should also understand how the individual checks are composed along with reduction and caching to make the problem tractable; naively running each check and reducing along the way is likely to yield unacceptable performance results.

## Sort equality

Two `Sort` expressions are definitionally equal if the levels they represent are equal by antisymmetry using the partial order on levels. 

```
defEq (Sort x) (Sort y):
  x ≤ y ∧ y ≤ x
```

## Const equality

Two `Const` expressions are definitionally equal if their names are identical, and their levels are equal under antisymmetry.

```
defEq (Const n xs) (Const m ys):
  n == m ∧ forall (x, y) in (xs, ys), antisymmEq x y

  -- also assert that xs and ys have the same length if your `zip` doesn't do so.
```

## Bound Variables

For implementations using a substitution-based strategy like locally nameless (if you're following the C++ or lean4lean implementations, this is you), encountering a bound variable is an error; bound variables should have been replaced during weak reduction if they referred to an argument, or they should have been replaced with a free variable via strong reduction as part of a definitional equality check for a pi or lambda expression.

For closure-based implementations, look up the elements corresponding to the bound variables and assert that they are definitionally equal.

## Free Variables

Two free variables are definitionally equal if they have the same identifier (unique ID or deBruijn level). Assertions about the equality of the binder types should have been performed wherever the free variables were constructed (like the definitional equality check for pi/lambda expressions), so it is not necessary to re-check that now.

```
defEqFVar (id1, _) (id2, _):
  id1 == id2
```

## App

Two application expressions are definitionally equal if their function component and argument components are definitionally equal.

```
defEqApp (App f a) (App g b):
  defEq f g && defEq a b
```


## Pi

Two Pi expressions are definitionally equal if their binder types are definitionally equal, and their body types, after substituting in the appropriate free variable, are definitionally equal.

```
defEq (Pi s a) (Pi t b)
  if defEq s.type t.type
  then
    let thisFvar := fvar s
    defEq (inst a thisFvar) (inst b thisFvar)
  else
    false
```

## Lambda

Lambda uses the same test as Pi:

```
defEq (Lambda s a) (Lambda t b)
  if defEq s.type t.type
  then
    let thisFvar := fvar s
    defEq (inst a thisFvar) (inst b thisFvar)
  else
    false
```

## Structural eta

Lean recognizes definitional equality of two elements `x` and `y` if they're both instances of some structure type, and the fields are definitionally equal using the following procedure comparing the constructor arguments of one and the projected fields of the other:

```
defEqEtaStruct x y:
  let (yF, yArgs) := unfoldApps y
  if 
    yF is a constructor for an inductive type `T` 
    && `T` can be a struct
    && yArgs.len == T.numParams + T.numFields
    && defEq (infer x) (infer y)
  then
    forall i in 0..t.numFields, defEq Proj(i+T.numParams, x) yArgs[i+T.numParams]

    -- we add `T.numParams` to the index because we only want 
    -- to test the non-param arguments. we already know the 
    -- parameters are defEq because the inferred types are 
    -- definitionally equal.
```

The more pedestrian case of congruence `T.mk a .. N` = `T.mk x .. M` if `[a, .., N] = [x, .., M]`, is simply handled by the `App` test.

## Unit-like equality

Lean recognizes definitional equality of two elements `x: S p_0 .. p_N` and `y: T p_0 .. p_M` under the following conditions:

+ `S` is an inductive type
+ `S` has no indices
+ `S` has only one constructor which takes no arguments other than the parameters of `S`, `p_0 .. p_N`
+ The types `S p_0 .. p_N` and `T p0 .. p_M` are definitionally equal

Intuitively this definitional equality is fine, because all of the information that elements of these types can convey is captured by their types, and we're requiring those types to be definitionally equal.

## Eta expansion

```
defEqEtaExpansion x y : bool :=
  match x, (whnf $ infer y) with
  | Lambda .., Pi binder _ => defEq x (App (Lambda binder (Var 0)) y)
  | _, _ => false
```

The lambda created on the right, `(fun _ => $0) y` trivially reduces to `y`, but the addition of the lambda binder gives the `x` and `y'` a chance to match with the rest of the definitional equality procedure.

## Proof irrelevant equality

Lean treats proof irrelevant equality as definitional. For example, Lean's definitional equality procedure treats any two proofs of `2 + 2 = 4` as definitionally equal expressions.

If a type `T` infers as `Sort 0`, we know it's a proof, because it is an element of `Prop` (remember that `Prop` is `Sort 0`).

```
defEqByProofIrrelevance p q :
  infer(p) == S ∧ 
  infer(q) == T ∧
  infer(S) == Sort(0) ∧
  infer(T) == Sort(0) ∧
  defEq(S, T)
```

If `p` is a proof of type `A` and `q` is a proof of type `B`, then if `A` is definitionally equal to `B`, `p` and `q` are definitionally equal by proof irrelevance.

## Natural numbers (nat literals)

Two nat literals are definitionally equal if they can be reduced to `Nat.zero`, or they can be reduced as (`Nat.succ x`, `Nat.succ y`), where `x` and `y` are definitionally equal.

```
match X, Y with
| Nat.zero, Nat.zero => true
| NatLit 0, NatLit 0 => true
| Nat.succ x, NatLit (y+1) => defEq x (NatLit y)
| NatLit (x+1), Nat.succ y => defEq (NatLit x) y
| NatLit (x+1), NatLit (y+1) => x == y
| _, _ => false
```

## String literal

`StringLit(s), App(String.mk, a)`

The string literal `s` is converted to an application of `Const(String.mk, [])` to a `List Char`. Because Lean's `Char` type is used to represent unicode scalar values, their integer representation is a 32-bit unsigned integer.

To illustrate, the string literal "ok", which uses two characters corresponding to the 32 bit unsigned integers `111` and `107` is converted to:

>(String.mk (((List.cons Char) (Char.ofNat.[] NatLit(111))) (((List.cons Char) (Char.ofNat NatLit(107))) (List.nil Char))))

## Lazy delta reduction and congruence

The available kernel implementations implement a "lazy delta reduction" procedure as part of the definitional equality check, which unfolds definitions lazily using [reducibility hints](./declarations.md#reducibility-hints) and checks for congruence when things look promising. This is a much more efficient strategy than eagerly reducing both expressions completely before checking for definitional equality.

If we have two expressions `a` and `b`, where `a` is an application of a definition with height 10, and `b` is an application of a definition with height 12, the lazy delta procedure takes the more efficient route of unfolding `b` to try and get closer to `a`, as opposed to unfolding both of them completely, or blindly choosing one side to unfold.

If the lazy delta procedure finds two expressions which are an application of a `const` expression to arguments, and the `const` expressions refer to the same declaration, the expressions are checked for congruence (whether they're the same consts applied to definitionally equal arguments). Congruence failures are cached, and for readers writing their own kernel, caching these failures turns out to be a performance critical optimization, since the congruence check involves a potentially expensive call to `def_eq_args`.

## Syntactic equality (also structural or pointer equality)

Two expressions are definitionally equal if they refer to exactly the same implementing object, as long as the type checker ensures that two objects are equal if and only if they are constructed from the same components (where the relevant constructors are those for Name, Level, and Expr).