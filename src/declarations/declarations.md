# Declarations

Declarations are the big ticket items, and are the last domain elements we need to define.


```
declarationInfo ::= Name, (universe params : List Level), (type : Expr)

declar ::= 
  Axiom declarationInfo
  | Definition declarationInfo (value : Expr) ReducibilityHint
  | Theorem declarationInfo (value : Expr) 
  | Opaque declarationInfo (value : Expr) 
  | Quot declarationInfo
  | InductiveType 
      declarationInfo
      is_recursive: Bool
      num_params: Nat
      num_indices: Nat
      -- The name of this type, and any others in a mutual block
      allIndNames: Name+
      -- The names of the constructors for *this* type only, 
      -- not including the constructors for mutuals that may 
      -- be in this block.
      constructorNames: Name*
      
  | Constructor 
      declarationInfo 
      (inductiveName : Name) 
      (numParams : Nat) 
      (numFields : Nat)

  | Recursor 
        declarationInfo 
        numParams : Nat
        numIndices : Nat
        numMotives : Nat
        numMinors : Nat
        RecRule+
        isK : Bool

RecRule ::= (constructor name : Name), (number of constructor args : Nat), (val : Expr)
```


## Checking a declaration


For all declarations, the following preliminary checks are performed before any additional procedures specific to certain kinds of declaration:

+ The universe parameters in the declaration's `declarationInfo` must not have duplicates. For example, a declaration `def Foo.{u, v, u} ...` would be prohibited.

+ The declaration's type must not have free variables; all variables in a "finished" declaration must correspond to a binder.

+ The declaration's type must be a type (`infer declarationInfo.type` must produce a `Sort`). In Lean, a declaration `def Foo : Nat.succ := ..` is not permitted; `Nat.succ` is a value, not a type.

### Axiom

The only checks done against axioms are those done for all declarations which ensure the `declarationInfo` passes muster. If an axiom has a valid set of univers parameters and a valid type with no free variables, it is admitted to the environment.

### Quot

The `Quot` declarations are `Quot`, `Quot.mk`, `Quot.ind`, and `Quot.lift`. These declarations have prescribed types which are known to be sound within Lean's theory, so the environment's quotient declarations must match those types exactly. These types are hard-coded into kernel implementations since they are not prohibitively complex.

### Definition, theorem, opaque

Definition, theorem, and opaque are interesting in that they both a type and a value. Checking these declarations involves inferring a type for the declaration's value, then asserting that the inferred type is definitionally equal to the ascribed type in the `declarationInfo`.

In the case of a theorem, the `declarationInfo`'s type is what the user claims the type is, and therefore what the user is claiming to prove, while the value is what the user has offered as a proof of that type. Inferring the type of the received value amounts to checking what the proof is actually a proof of, and the definitional equality assertion ensures that the thing the value proves is actually what the user intended to prove.

#### Reducibility hints

Reducibility hints contain information about how a declaration should be unfolded. An `abbreviation` will generally always be unfolded, `opaque` will not be unfolded, and `regular N` might be unfolded depending on the value of `N`. The `regular` reducibility hints correspond to a definition's "height", which refers to the number of declarations that definition uses to define itself. A definition `x` with a value that refers to definition `y` will have a height value greater than `y`.