# Reduction

Reduction is about nudging expressions toward their [normal form](https://en.wikipedia.org/wiki/Normal_form_(abstract_rewriting)) so we can determine whether expressions are definitionally equal. For example, we need to perform beta reduction to determine that `(fun x => x) Nat.zero` is definitionally equal to `Nat.zero`, and delta reduction to determine that `id List.nil` is definitionally equal to `List.nil`. 

Reduction in Lean's kernel has two properties that introduce concerns which sometimes go unaddressed in basic textbook treatments of the topic. First, reduction in some cases is interleaved with inference. Among other things, this means reduction may need to be performed with open terms, even though the reduction procedures themselves are not creating free variables. Second, `const` expressions which are applied to multiple arguments may need to be considered together with those arguments during reduction (as in iota reduction), so sequences of applications need to be unfolded together at the beginning of reduction. 

## Beta reduction 

Beta reduction is about reducing function application. Concretely, the reduction:

```
(fun x => x) a    ~~>    a
```

An implementation of beta reduction must despine an expression to collect any arguments in `app` expressions, check whether the expression to which they're applied is a lambda, then substitute the appropriate argument for any appearances of the corresponding bound variable in the function body:

```
betaReduce e:
  betaReduceAux e.unfoldApps

betaReduceAux f args:
  match f, args with
  | lambda _ body, arg :: rest => betaReduceAux (inst body arg) rest
  | _, _ => foldApps f args
```

An important performance optimization for the instantiation (substitution) component of beta reduction is what's sometimes referred to as "generalized beta reduction", which involves gathering the arguments that have a corresponding lambda and substituting them in all at once. This optimization means that for `n` sequential lambda expressions with applied arguments, we only perform one traversal of the expression to substitute the appropriate arguments, instead of `n` traversals.

```
betaReduce e:
  betaReduceAux e.unfoldApps []

betaReduceAux f remArgs argsToApply:
  match f, remArgs with
  | lambda _ body, arg :: rest => betaReduceAux body rest (arg :: argsToApply)
  | _, _ => foldApps (inst f argsToApply) remArgs
```

## Zeta reduction (reducing `let` expressions)

Zeta reduction is a fancy name for reduction of `let` expressions. Concretely, reducing 

```
let (x : T) := y; x + 1    ~~>    (y : T) + 1
```

An implementation can be as simple as:

```
reduce Let _ val body:
  instantiate body val
```

## Delta reduction (definition unfolding)

Delta reduction refers to unfolding definitions (and theorems). Delta reduction is done by replacing a `const ..` expr with the referenced declaration's value, after swapping out the declaration's generic universe parameters for the ones that are relevant to the current context. 

If the current environment contains a definition `x` which was declared with universe parameters `u*`and value `v`, then we may delta reduce an expression `Const(x, w*)` by replacing it with `val`, then substituting the universe parameters `u*` for those in `w*`.

```
deltaReduce Const name levels:
  if environment[name] == (d : Declar) && d.val == v then
  substituteLevels (e := v) (ks := d.uparams) (vs := levels)
```

If we had to remove any applied arguments to reach the `const` expression that was delta reduced, those arguments should be reapplied to the reduced definition.

## Projection reduction

A `proj` expression has a natural number index indicating the field to be projected, and another expression which is the structure itself. The actual expression comprising the structure should be a sequence of arguments applied to `const` referencing a constructor.

Keep in mind that for fully elaborated terms, the arguments to the constructor will include any parameters, so an instance of the `Prod` constructor would be e.g. `Prod.mk A B (a : A) (b : B)`.

The natural number indicating the projected field is 0-based, where 0 is the first *non-parameter* argument to the constructor, since a projection cannot be used to access the structure's parameters.

With this in mind, it becomes clear that once we despine the constructor's arguments into `(constructor, [arg0, .., argN])`, we can reduce the projection by simply taking the argument at position `i + num_params`, where `num_params` is what it sounds like, the number of parameters for the structure type.

```
reduce proj fieldIdx structure:
  let (constructorApp, args) := unfoldApps (whnf structure)
  let num_params := environment[constructorApp].num_params
  args[fieldIdx + numParams]

  -- Following our `Prod` example, constructorApp will be `Const(Prod.mk, [u, v])`
  -- args will be `[A, B, a, b]`
```

### Special case for projections: String literals

The kernel extension for string literals introduces one special case in projection reduction, and one in iota reduction. 

Projection reduction for string literals: Because the projection expression's structure might reduce to a string literal (Lean's `String` type is defined as a structure with one field, which is a `List Char`)

If the structure reduces as a `StringLit (s)`, we convert that to `String.mk (.. : List Char)` and proceed as usual for projection reduction.



## Nat literal reduction

The kernel extension for nat literals includes reduction of `Nat.succ` as well as the binary operations of addition, subtraction, multiplication, exponentiation, division, mod, boolean equality, and boolean less than or equal.

If the expression being reduced is `Const(Nat.succ, []) n` where `n` can be reduced to a nat literal `n'`, we reduce to `NatLit(n'+1)`

If the expression being reduced is `Const(Nat.<binop>, []) x y` where `x` and `y` can be reduced to nat literals `x'` and `y'`, we apply the native version of the appropriate `<binop>` to `x'` and `y'`, returning the resulting nat literal.


Examples:
```
Const(Nat.succ, []) NatLit(100) ~> NatLit(100+1)

Const(Nat.add, []) NatLit(2) NatLit(3) ~> NatLit(2+3)

Const(Nat.add, []) (Const Nat.succ [], NatLit(10)) NatLit(3) ~> NatLit(11+3)
```

## Iota reduction (pattern matching)

Iota reduction is about applying reduction strategies that are specific to, and derived from, a given inductive declaration. What we're talking about is application of an inductive declaration's recursor (or the special case of `Quot` which we'll see later).

Each recursor has a set of "recursor rules", one recursor rule for each constructor. In contrast to the recursor, which presents as a type, these recursor rules are value level expressions showing how to eliminate an element of type `T` created with constructor `T.c`. For example, `Nat.rec` has a recursor rule for `Nat.zero`, and another for `Nat.succ`.

For an inductive declaration `T`, one of the elements demanded by `T`'s recursor is an actual `(t : T)`, which is the thing we're eliminating. This `(t : T)` argument is known as the "major premise". Iota reduction performs pattern matching by taking apart the major premise to see what constructor was used to make `t`, then retrieving and applying the corresponding recursor rule from the environment.

Because the recursor's type signature also demands the parameters, motives, and minor premises required, we don't need to change the arguments to the recursor to perform reduction on e.g. `Nat.zero` as opposed to `Nat.succ`.

In practice, it's sometimes necessary to do some initial manipulation to expose the constructor used to create the major premise, since it may not be found as a direct application of a constructor. For example, a `NatLit(n)` expression will need to be transformed into either `Nat.zero`, or `App Const(Nat.succ, []) ..`. For structures, we may also perform structural eta expansion, transforming an element `(t : T)` into `T.mk t.1 .. t.N`, thereby exposing the application of the `mk` constructor, permitting iota reduction to proceed (if we can't figure out what constructor was used to create the major premise, reduction fails).

## List.rec type

```
forall 
  {α : Type.{u}} 
  {motive : (List.{u} α) -> Sort.{u_1}}, 
  (motive (List.nil.{u} α)) -> 
  (forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) -> (forall (t : List.{u} α), motive t)
```

## List.nil rec rule

```
fun 
  (α : Type.{u}) 
  (motive : (List.{u} α) -> Sort.{u_1}) 
  (nilCase : motive (List.nil.{u} α)) 
  (consCase : forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) => 
  nilCase
```

## List.cons rec rule

```
fun 
  (α : Type.{u}) 
  (motive : (List.{u} α) -> Sort.{u_1}) 
  (nilCase : motive (List.nil.{u} α)) 
  (consCase : forall (head : α) (tail : List.{u} α), (motive tail) -> (motive (List.cons.{u} α head tail))) 
  (head : α) 
  (tail : List.{u} α) => 
  consCase head tail (List.rec.{u_1, u} α motive nilCase consCase tail)
```


### k-like reduction

For some inductive types, known as "subsingleton eliminators", we can proceed with iota reduction even when the major premise's constructor is not directly exposed, as long as we know its type. This may be the case when, for example, the major premise appears as a free variable. This is known as k-like reduction, and is permitted because all elements of a subsingleton eliminator are identical. 

To be a subsingleton eliminator, an inductive declaration must be an inductive prop, must not be a mutual or nested inductive, must have exactly one constructor, and the sole constructor must take only the type's parameters as arguments (it cannot "hide" any information that isn't fully captured in its type signature).

For example, the value of any element of the type `Eq Char 'x'` is fully determined just by its type, because all elements of this type are identical.

If iota reduction finds a major premise which is a subsingleton eliminator, it is permissible to substitute the major premise for an application of the type's constructor, because that is the only element the free variable could actually be. For example, a major premise which is a free variable of type `Eq Char 'a'` may be substituted for an explicitly constructed `Eq.refl Char 'a'`.

Getting to the nuts and bolts, if we neglected to look for and apply k-like reduction, free variables that are subsingleton eliminators would fail to identify the corresponding recursor rule, iota reduction would fail, and certain conversions expected to succeed would no longer succeed.

### `Quot` reduction; `Quot.ind` and `Quot.lift`

`Quot` introduces two special cases which need to be handled by the kernel, one for `Quot.ind`, and one for `Quot.lift`.

Both `Quot.ind` and `Quot.lift` deal with application of a function `f` to an argument `(a : α)`, where the `a` is a component of some `Quot r`, formed with `Quot.mk r a`. 

To execute the reduction, we need to pull out the argument that is the `f` element and the argument that is the `Quot` where we can find `(a : α)`, then apply the function `f` to `a`. Finally, we reapply any arguments that were part of some outer expression not related to the invocation of `Quot.ind` or `Quot.lift`.

Since this is only a reduction step, we rely on the type checking phases done elsewhere to provide assurances that the expression as a whole is well typed.

The type signatures for `Quot.ind` and `Quot.mk` are recreated below, mapping the elements of the telescope to what we should find as the arguments. The elements with a `*` are the ones we're interested in for reduction.

```
Quotient primitive Quot.ind.{u} : ∀ {α : Sort u} {r : α → α → Prop} 
  {β : Quot r → Prop}, (∀ (a : α), β (Quot.mk r a)) → ∀ (q : Quot r), β q

  0  |-> {α : Sort u} 
  1  |-> {r : α → α → Prop} 
  2  |-> {β : Quot r → Prop}
  3* |-> (∀ (a : α), β (Quot.mk r a)) 
  4* |-> (q : Quot r)
  ...
```

```
Quotient primitive Quot.lift.{u, v} : {α : Sort u} →
  {r : α → α → Prop} → {β : Sort v} → (f : α → β) → 
  (∀ (a b : α), r a b → f a = f b) → Quot r → β

  0  |-> {α : Sort u}
  1  |-> {r : α → α → Prop} 
  2  |-> {β : Sort v} 
  3* |-> (f : α → β) 
  4  |-> (∀ (a b : α), r a b → f a = f b)
  5* |-> Quot r
  ...
```
