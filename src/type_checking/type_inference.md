# Type Inference

Type inference is a procedure for determining the type of a given expression, and is one of the core functionalities of Lean's kernel. Type inference is how we determine that `Nat.zero` is an element of the type `Nat`, or that `(fun (x : Char) => var(0))` is an element of the type `Char -> Char`.

This section begins by examining the simplest complete procedure for type inference, then the more performant but slightly more complex version of each procedure.

We will also look at a number of additional correctness assertions that Lean's kernel makes during type inference.

## Bound variables

If you're following Lean's implementation and using the locally nameless approach, you should not run into bound variables during type inference, because all open binders will be instantiated with the appropriate free variables.

When we come across a binder, we need to traverse into the body to determine the body's type. There are two main approaches one can take to preserve the information about the binder type; the one used by Lean proper is to create a free variable that retains the binder's type information, replace the corresponding bound variables with the free variable using instantiation, and then enter the body. This is nice, because we don't have to keep track of a separate piece of state for a typing context.

For closure-based implementations, you will generally have a separate typing context that keeps track of the open binders; running into a bound variable then means that you will index into your typing context to get the type of that variable.

## Free variables

When a free variable is created, it's given the type information from the binder it represents, so we can just take that type information as the result of inference.

```
infer FVar id binder:
  binder.type
```

## Function application

```
infer App(f, arg):
  match (whnf $ infer f) with
  | Pi binder body => 
    assert! defEq(binder.type, infer arg)
    instantiate(body, arg)
  | _ => error
```

The additional assertion needed here is that the type of `arg` matches the type of `binder`. For example, in the expression

`(fun (n : Nat) => 2 * n) 10`, we would need to assert that `defEq(Nat, infer(10))`.

While existing implementations prefer to perform this check inline, one could potentially store this equality assertion for processing elsewhere.

## Lambda

```
infer Lambda(binder, body):
  assert! infersAsSort(binder.type)
  let binderFvar := fvar(binder)
  let bodyType := infer $ instantiate(body, binderFVar)
  Pi binder (abstract bodyType binderFVar)
```

# Pi

```
infer Pi binder body:
  let l := inferSortOf binder
  let r := inferSortOf $ instantiate body (fvar(binder))
  imax(l, r)

inferSortOf e:
  match (whnf (infer e)) with
  | sort level => level
  | _ => error
```

## Sort

The type of any `Sort n` is just `Sort (n+1)`.

```
infer Sort level:
  Sort (succ level)
```

## Const

`const` expressions are used to refer to other declarations by name, and any other declaration referred to must have been previously declared and had its type checked. Since we therefore already know what the type of the referred to declaration is, we can just look it up in the environment. We do have to substitute in the current declaration's universe levels for the indexed definition's universe parameters however.

```
infer Const name levels:
  let knownType := environment[name].type
  substituteLevels (e := knownType) (ks := knownType.uparams) (vs := levels)
```

## Let

```
infer Let binder val body:
  assert! inferSortOf binder
  assert! defEq(infer(val), binder.type)
  infer (instantiate body val)
```

## Proj

We're trying to infer the type of something like `Proj (projIdx := 0) (structure := Prod.mk A B (a : A) (b : B))`.

Start by inferring the type of the structure offered; from that we can get the structure name and look up the structure and constructor type in the environment.

Traverse the constructor type's telescope, substituting the parameters of `Prod.mk` into the telescope for the constructor type. If we looked up the constructor type `A -> B -> (a : A) -> (b : B) -> Prod A B`, substitute A and B, leaving the telescope `(a : A) -> (b : B) -> Prod A B`.

The remaining parts of the constructor's telescope represent the structure's fields and have the type information in the binder, so we can just examine `telescope[projIdx]` and take the binder type. We do have to take care of one more thing; because later structure fields can depend on earlier structure fields, we need to instantiate the rest of the telescope (the body at each stage) with `proj thisFieldIdx s` where `s` is the original structure in the proj expression we're trying to infer.

```
infer Projection(projIdx, structure):
  let structType := whnf (infer structure)
  let (const structTyName levels) tyArgs := structType.unfoldApps
  let InductiveInfo := env[structTyName]
  -- This inductive should only have the one constructor since it's claiming to be a structure.
  let ConstructorInfo := env[InductiveInfo.constructorNames[0]]

  let mut constructorType := substLevels ConstructorInfo.type (newLevels := levels)

  for tyArg in tyArgs.take constructorType.numParams
    match (whnf constructorType) with
      | pi _ body => inst body tyArg
      | _ => error

  for i in [0:projIdx]
    match (whnf constructorType) with
      | pi _ body => inst body (proj i structure)
      | _ => error

  match (whnf constructorType) with
    | pi binder _=> binder.type
    | _ => error 
```

## Nat literals

Nat literals infer as the constant referring to the declaration `Nat`.

```
infer NatLiteral _:
  Const(Nat, [])
```

## String literals

String literals infer as the constant referring to the declaration `String`.

```
infer StringLiteral _:
  Const(String, [])
```

