# The big picture

To give the reader a road map, the entire procedure of checking an export file consists of these steps:

+ Parse an export file, yielding a collection of components for each primitive sort: names, levels, and expressions, as well as a collection of declarations.

+ The collection of parsed declarations represents an environment, which is a mapping from each declaration's name to the declaration itself; these are the actual targets of the type checking process.

+ For each declaration in the environment, the kernel requires that the declaration is not already declared in the environment, has no duplicate universe parameters, that the declaration's type is actually a type and not a value (that `infer declar.ty` returns an expression `Sort <n>`), and that the declaration's type has no free variables.

+ For definitions, theorems, and opaque declarations, assert that inferring the type of the definition's value yields an expression which is definitionally equal to the type the user assigned to the declaration. This is where the rubber meets the road in terms of asserting that proofs are correct, and for theorems, this is the step that correponds to "the user says this is a proof of `P`, does the value actually constitute a valid proof of `P`".

+ For inductive declarations, their constructors, and recursors, check that they are properly formed and comply with the rules of Lean's type theory (more on this later). 

+ If the export file includes the primitive declarations for quotient types, ensure those declarations have the correct types, and that the `Eq` type exists, and is defined properly (since quotients rely on equality).

+ Finally, pretty print any declarations requested by the user, so they can check that the declarations checked match the declarations they exported.

