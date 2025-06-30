# Expressions

## Complete syntax

Expressions will be explained in more detail below, but just to get it out in the open, the complete syntax for expressions, including the string and nat literal extensions, is as follows:

```
Expr ::= 
  | boundVariable 
  | freeVariable 
  | const 
  | sort 
  | app 
  | lambda 
  | forall 
  | let 
  | proj 
  | literal

BinderInfo ::= Default | Implicit | InstanceImplicit | StrictImplicit

const ::= Name, Level*
sort ::= Level
app ::= Expr Expr
-- a deBruijn index
boundVariable ::= Nat
lambda ::= Name, (binderType : Expr), BinderInfo, (body : Expr)
forall ::= Name, (binderType : Expr), BinderInfo, (body : Expr)
let ::= Name, (binderType : Expr), (val : Expr) (body : Expr)
proj ::= Name Nat Expr
literal ::= natLit | stringLit

-- Arbitrary precision nat/unsigned integer
natLit ::= Nat
-- UTF-8 string
stringLit ::= String

-- fvarId can be implemented by unique names or deBruijn levels; 
-- unique names are more versatile, deBruijn levels have better
-- cache behavior
freeVariable ::= Name, Expr, BinderInfo, fvarId
```

Some notes:

+ The `Nat` used by nat literals should be an arbitrary precision natural/bignum.

+ The expressions that have binders (lambda, pi, let, free variable) can just as easily bundle the three arguments (binder_name, binder_type, binder_style) as one argument `Binder`, where a binder is `Binder ::= Name BinderInfo Expr`. In the pseudocode that appears elsewhere I will usually treat them as though they have that property, because it's easier to read.

+ Free variable identifiers can be either unique identifiers, or they can be deBruijn levels.

+ The expression type used in Lean proper also has an `mdata` constructor, which declares an expression with attached metadata. This metadata does not effect the expression's behavior in the kernel, so we do not include this constructor.


## Binder information

Expressions constructed with the lambda, pi, let, and free variable constructors contain binder information, in the form of a name, a binder "style", and the binder's type. The binder's name and style are only for use by the pretty printer, and do not alter the core procedures of inference, reduction, or equality checking. In the pretty printer however, the binder style may alter the output depending on the pretty printer options. For example, the user may or may not want to display implicit or instance implicits (typeclass variables) in the output.

### Sort

`sort` is simply a wrapper around a level, allowing it to be used as an expression.


### Bound variables

Bound variables are implemented as natural numbers representing [deBruijn indices](https://en.wikipedia.org/wiki/De_Bruijn_index).

### Free variables

Free variables are used to convey information about bound variables in situations where the binder is currently unavailable. Usually this is because the kernel has traversed into the body of a binding expression, and has opted not to carry a structured context of the binding information, instead choosing to temporarily swap out the bound variable for a free variable, with the option of swapping in a new (maybe different) bound variable to reconstruct the binder. This unavailability description may sound vague, but a literal explanation that might help is that expressions are implemented as trees without any kind of parent pointer, so when we descend into child nodes (especially across function boundaries), we end up just losing sight of the elements above where we currently are in a given expression.

When an open expression is closed by reconstructing binders, the bindings may have changed, invalidating previously valid deBruijn indices. The use of unique names or deBruijn levels allow this re-closing of binders to be done in a way that compensates for any changes and ensures the new deBruijn indices of the re-bound variables are valid with respect the reconstructed telescope (see [this section](../kernel_concepts/instantiation_and_abstraction.md#implementing-free-variable-abstraction)).

Going forward, we may use some form of the term "free variable identifier" to refer to the objects in whatever scheme (unique IDs or deBruijn levels) an implementation may be using.

### `Const`

The `const` constructor is how an expression refers to another declaration in the environment, it must do so by reference. 

In example below, `def plusOne` creates a `Definition` declaration, which is checked, then admitted to the environment. Declarations cannot be placed directly in expressions, so when the type of `plusOne_eq_succ` invokes the previous declaration `plusOne`, it must do so by name. An expression is created: `Expr.const (plusOne, [])`, and when the kernel finds this `const` expression, it can look up the declaration referred to by name, `plusOne`, in the environment:

```
def plusOne : Nat -> Nat := fun x => x + 1

theorem plusOne_eq_succ (n : Nat) : plusOne n = n.succ := rfl 
```

Expressions created with the `const` constructor also carry a list of levels which are substituted into any unfolded or inferred declarations taken from the environment by looking up the definition the `const` expression refers to. For example, inferring the type of `const List [Level.param(x)]` involves looking up the declaration for `List` in the current environment, retrieving its type and universe parameters, then substituting `x` for the universe parameter with which `List` was initially declared.


### Lambda, Pi

`lambda` and `pi` expressions (Lean proper uses the name `forallE` instead of `pi`) refer to function abstraction and the "forall" binder (dependent function types) respectively. 


```
  binderName      body
      |            |
fun (foo : Bar) => 0 
            |         
        binderType    

-- `BinderInfo` is reflected by the style of brackets used to
-- surround the binder.
```

### Let

`let` is exactly what it sounds like. While `let` expressions are binders, they do not have a `BinderInfo`, their binder info is treated as `Default`.

```
  binderName      val
      |            |
let (foo : Bar) := 0; foo
            |          |
        binderType     .... body
```


### App

`app` expressions represent the application of an argument to a function. App nodes are binary (have only two children, a single function and an single argument), so `f x_0 x_1 .. x_N` is represented by `App(App(App(f, x_0), x_1)..., x_N)`, or visualized as a tree:

```
                App
                / \
              ...  x_N
              /
            ...
           App
          / \
       App  x_1
       / \
     f  x_0

```

An exceedingly common kernel operation for manipulating expressions is folding and unfolding sequences of applications, getting `(f, [x_0, x_1, .., x_N])` from the tree structure above, or folding `f, [x_0, x_1, .., x_N]` into the tree above.


### Projections

The `proj` constructor represents structure projections. Inductive types that are not recursive, have only one constructor, and have no indices can be structures.

The constructor takes a name, which is the name of the type, a natural number indicating the field being projected, and the actual structure the projection is being applied to.

Be aware that in the kernel, projection indices are 0-based, despite being 1-based in Lean's vernacular, where 0 is the first non-parameter argument to the constructor.

For example, the kernel expression `proj Prod 0 (@Prod.mk A B a b)` would project the `a`, because it is the 0th field after skipping the parameters `A` and `B`.

While the behavior offered by `proj` can be accomplished by using the type's recursor, `proj` more efficiently handles frequent kernel operations.

### Literals

Lean's kernel optionally supports arbitrary precision Nat and String literals. As needed, the kernel can transform a nat literal `n` to `Nat.zero` or `Nat.succ m`, or convert a string literal `s` to `String.mk List.nil` or `String.mk (List.cons (Char.ofNat _) ...)`.

String literals are lazily converted to lists of characters for testing definitional equality, and when they appear as the major premise in reduction of a recursor.

Nat literals are supported in the same positions as strings (definitional equality and major premises of a recursor application), but the kernel also provide support for addition, multiplication, exponentiation, subtraction, mod, division, as well as boolean equality and "less than or equal" comparisons on nat literals.