# Instantiation and abstraction

Instantiation refers to substitution of bound variables for the appropriate arguments. Abstraction refers to replacement of free variables with the appropriate bound variable when replacing binders. Lean's kernel uses deBruijn indices for bound variables and unique identifiers for free variables.

For our purposes, a free variable is a variable in an expression that refers to a binder which has been "opened", and is no longer immediately available to us, so we replace the corresponding bound variable with a free variable that has some information about the binder we're leaving behind.

To illustrate, let's say we have some lambda expression `(fun (x : A) => <body>)` and we're doing type inference. Type inference has to traverse into the `<body>` part of the expression, which may contain a bound variable that refers to `x`. When we traverse into the body, we can either add `x` to some stateful context of binders and take the whole stateful context into the body with us, or we can temporarily replace all of the bound variables that refer to `x` with a free variable, allowing us to traverse into the body without having to carry any additional context.

If we eventually come back to where we were before we opened the binder, abstraction allows us to replace all of the free variables that were once bound variables referring to `x` with new bound variables that again refer to `x`, with the correct deBruijn indices.

## Implementing free variable abstraction

For deBruijn levels, the free variables keep track of a number that says "I am a free variable representing the nth bound variable *from the top of the telescope*". 

This is the opposite of a deBruijn index, which is a number indicating "the nth bound variable from the bottom of the telescope".

Top and bottom here refer to visualizing the expression's telescope as a tree:

```
      fun
      /  \
    a    fun
        /   \
      b      ...
              \
              fun
             /   \
            e    bvar(0)
```

For example, with a lambda `fun (a b c d e) => bvar(0)`, the bound variable refers to `e`, by referencing "the 0th from the bottom".

In the lambda expression `fun (a b c d e) => fvar(4)`, the free variable is a deBruijn level representing `e` again, but this time as "the 4th from the top of the telescope".

Why the distinction? When we create a free variable during strong reduction, we know a couple of things: we know that the free variable we're about to sub in might get moved around by further reduction, we know how many open binder are *ABOVE* us (because we had to visit them to get here), and we know we might need to quote/abstract this expression to replace the binders, meaning we need to re-bind the free variable. However, in that moment, we do NOT know how many binders remain below us, so we cannot say how many variables from the bottom that variable might be when it's eventually abstracted/quoted.

For implementations using unique identifiers to tag free variables, this problem is solved by having the actual telescope that's being reconstructed during abstraction. As long as you have the expression and a list of the uniquely-tagged free variables, you can abstract, because the position of the free variables within the list indicates their binder position.

