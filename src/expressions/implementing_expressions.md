# Implementing Expressions

A few noteworthy points about the expression type for readers interested in writing their own kernel in whole or in part...

## Stored data

Expressions need to store some data inline or cache it somewhere to prevent prohibitively expensive recomputation. For example, creating an expression `app x y` needs to calculate and then store the hash digest of the resulting app expression, and it needs to do so by getting the cached hash digests of `x` and `y` instead of traversing the entire tree of `x` to recursively calculate the digest of `x`, then doing the same for `y`.

The data you will probably want to store inline are the hash digest, the number of loose bound variables in an expression, and whether or not the expression has free variables. The latter two are useful for optimizing instantiation and abstraction respectively.

An example "smart constructor" for `app` expressions would be:

```
def mkApp x y:
  let hash := hash x.storedHash y.storedHash
  let numLooseBvars := max x.numLooseBvars y.numLooseBvars
  let hasFvars := x.hasFvars || y.hasFvars
  .app x y (cachedData := hash numLooseBVars hasFVars)
```

## No deep copies

Expressions should be implemented such that child expressions used to construct some parent expression are not deep copied. Put another way, creating an expression `app x y` should not recursively copy the elements of `x` and `y`, rather it should take some kind of reference, whether it's a pointer, integer index, reference to a garbage collected object, reference counted object, or otherwise (any of these strategies should deliver acceptable performance). If your default strategy for constructing expressions involves deep copies, you will not be able to construct any nontrivial environments without consuming massive amounts of memory.

## Example implementation for number of loose bound variables


```
numLooseBVars e:
    match e with
    | Sort | Const | FVar | StringLit | NatLit => 0
    | Var dbjIdx => dbjIdx + 1,
    | App fun arg => max fun.numLooseBvars arg.numLooseBvars
    | Pi binder body | Lambda binder body => 
    |   max binder.numLooseBVars (body.numLooseBVars - 1)
    | Let binder val body =>
    |   max (max binder.numLooseBvars val.numLooseBvars) (body.numLooseBvars - 1)
    | Proj _ _ structure => structure.numLooseBvars
```

For `Var` expressions, the number of loose bound variables is the deBruijn index plus one, because we're counting the number of binders that would need to be applied for that variable to no longer be loose (the `+1` is because deBruijn indices are 0-based). For the expression `Var(0)`, one binder needs to be placed above the bound variable in order for the variable to no longer be loose. For `Var(3)`, we need four:

```
--  3 2 1 0
fun a b c d => Var(3)
```

When we create a new binder (lambda, pi, or let), we can subtract 1 (using saturating/natural number subtraction) from the number of loose bvars in the body, because the body is now under one additional binder.

