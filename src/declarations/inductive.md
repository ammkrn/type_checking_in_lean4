
# The Secret Life of Inductive Types

### Inductive

For clarity, the whole shebang of "an inductive declaration" is a type, a list of constructors, and a list of recursors. The declaration's type and constructors are specified by the user, and the recursor is derived from those elements. Each recursor also gets a list of "recursor rules", also known as computation rules, which are value level expressions used in iota reduction (a fancy word for pattern matching). Going forward, we will do our best do distinguish between "an inductive type" and "an inductive declaration".

Lean's kernel natively supports mutual inductive declarations, in which case there is a list of (type, list constructor) pairs. The kernel supports nested inductive declarations by temporarily transforming them to mutual inductives (more on this below).

### Inductive types

The kernel requires the "inductive type" part of an inductive declaration to actually be a type, and not a value (`infer ty` must produce some `sort <n>`). For mutual inductives, the types being declared must all be in the same universe and have the same parameters.

### Constructor

For any constructor of an inductive type, the following checks are enforced by the kernel:

+ The constructor's type/telescope has to share the same parameters as the type of the inductive being declared.

+ For the non-parameter elements of the constructor type's telescope, the binder type must actually be a type (must infer as `Sort _`).

+ For any non-parameter element of the constructor type's telescope, the element's inferred sort must be less than or equal to the inductive type's sort, or the inductive type being declared has to be a prop.

+ No argument to the constructor may contain a non-positive occurrence of the type being declared (readers can explore this issue in depth [here](https://counterexamples.org/strict-positivity.html?highlight=posi#positivity-strict-and-otherwise)).

+ The end of the constructor's telescope must be a valid application of arguments to the type being declared. For example, we require the `List.cons ..` constructor to end with `.. -> List A`, and it would be an error for `List.cons` to end with `.. -> Nat`

#### Nested inductives 

Checking nested inductives is a more laborious procedure that involves temporarily specializing the nested parts of the inductive types in a mutual block so that we just have a "normal" (non-nested) set of mutual inductives, checking the specialized types, then unspecializing everything and admitting those types.

Consider this definition of S-expressions, with the nested construction `Array Sexpr`:

```
inductive Sexpr
| atom (c : Char) : Sexpr
| ofArray : Array Sexpr -> Sexpr
```

Zooming out, the process of checking a nested inductive declaration has three steps:

1. Convert the nested inductive declaration to a mutual inductive declaration by specializing the "container types" in which the current type is being nested. If the container type is itself defined in terms of other types, we'll need to reach those components for specialization as well. In the example above, we use `Array` as a container type, and `Array` is defined in terms of `List`, so we need to treat both `Array` and `List` as container types.

2. Do the normal checks and construction steps for a mutual inductive type.

3. Convert the specialized nested types back to the original form (un-specializing), adding the recovered/unspecialized declarations to the environment.

An example of this specialization would be the conversion of the `Sexpr` nested inductive above as:

```
mutual
  inductive Sexpr
    | atom : Char -> Sexpr
    | ofList : ListSexpr -> Sexpr

  inductive ListSexpr 
    | nil : ListSexpr
    | cons : Sexpr -> ListSexpr -> ListSexpr 

  inductive ArraySexpr
    | mk : ListSexpr -> ArraySexpr
end
```

Then recovering the original inductive declaration in the process of checking these types. To clarify, when we say "specialize", the new `ListSexpr` and `ArraySexpr` types above are specialized in the sense that they're defined only as lists and arrays of `Sexpr`, as opposed to being generic over some arbitrary type as with the regular `List` type.


### Recursors

TBD