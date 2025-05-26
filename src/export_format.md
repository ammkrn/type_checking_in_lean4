# Export format

An exporter is a program which emits Lean declarations using the kernel language, for consumption by external type checkers. Producing an export file is a complete exit from the Lean ecosystem; the data in the file can be checked with entirely external software, and the exporter itself is not a trusted component. Rather than inspecting the export file itself to see whether the declarations were exported as the developer intended, the exported declarations are checked by the external checker, and are displayed back to the user by a pretty printer, which produces output far more readable than the export file. Readers can (and are encouraged to) write their own external checkers for Lean export files.

The official exporter is [lean4export](https://github.com/leanprover/lean4export).


The master branch of the official exporter uses the same base format as lean 3 [here](https://github.com/leanprover/lean3/blob/master/doc/export_format.md), with the addition of the new-to-lean4 items, which are projections, literals, and explicitly support for let expressions. The new stuff is outlined in the lean4export readme.

A slightly modified version of the export format supported by [this fork](https://github.com/ammkrn/lean4export/tree/format2024) of lean4export is described below. These modifications export reducibility hints, quotient declarations, recursors, and iota reduction rules (rec rules). These additional exports were added to give more flexibility in implementation and for performance, but they also allow for the development of more minimized software used for experimentation, and can make bootstrapping and testing a checker easier for developers.

There are also [ongoing discussions](https://github.com/leanprover/lean4export/issues/3) about how best to evolve the export format.

## (ver 0.1.2)

For clarity, some of the compound items are decorated here with a name, for example `(name : T)`, but they appear in the export file as just an element of `T`.

The export scheme for mutual and nested inductives is as follows: 
+ `Inductive.inductiveNames` contains the names of all types in the `mutual .. end` block. The names of any other inductive types used in a nested (but not mutual) construction will not be included.
+ `Inductive.constructorNames` contains the names of all constructors for THAT inductive type, and no others (no constructors of the other types in a mutual block, and no constructors from any nested construction).

**NOTE:** readers writing their own parsers and/or checkers should initialize names[0] as the anonymous name, and levels[0] as universe zero, as they are not emitted by the exporter, but are expected to occupy the name and level indices for 0.

```
File ::= ExportFormatVersion Item*

ExportFormatVersion ::= nat '.' nat '.' nat

Item ::= Name | Universe | Expr | RecRule | Declaration

Declaration ::= 
    | Axiom 
    | Quotient 
    | Definition 
    | Theorem 
    | Inductive 
    | Constructor 
    | Recursor

nidx, uidx, eidx, ridx ::= nat

Name ::=
  | nidx "#NS" nidx string
  | nidx "#NI" nidx nat

Universe ::=
  | uidx "#US"  uidx
  | uidx "#UM"  uidx uidx
  | uidx "#UIM" uidx uidx
  | uidx "#UP"  nidx

Expr ::=
  | eidx "#EV"  nat
  | eidx "#ES"  uidx
  | eidx "#EC"  nidx uidx*
  | eidx "#EA"  eidx eidx
  | eidx "#EL"  Info nidx eidx eidx
  | eidx "#EP"  Info nidx eidx eidx
  | eidx "#EZ"  nidx eidx eidx eidx
  | eidx "#EJ"  nidx nat eidx
  | eidx "#ELN" nat
  | eidx "#ELS" (hexhex)*
  -- metadata node w/o extensions
  | eidx "#EM" mptr eidx

Info ::= "#BD" | "#BI" | "#BS" | "#BC"

Hint ::= "O" | "A" | "R" nat

RecRule ::= ridx "#RR" (ctorName : nidx) (nFields : nat) (val : eidx)

Axiom ::= "#AX" (name : nidx) (type : eidx) (uparams : nidx*)

Def ::= "#DEF" (name : nidx) (type : eidx) (value : eidx) (hint : Hint) (uparams : nidx*)

Opaq ::= "#OPAQ" (name : nidx) (type : eidx) (value : eidx) (uparams : nidx*)

Theorem ::= "#THM" (name : nidx) (type : eidx) (value : eidx) (uparams: nidx*)

Quotient ::= "#QUOT" (name : nidx) (type : eidx) (uparams : nidx*)

Inductive ::= 
  "#IND"
  (name : nidx) 
  (type : eidx) 
  (isRecursive: 0 | 1)
  (isNested : 0 | 1)
  (numParams: nat) 
  (numIndices: nat)
  (numInductives: nat)
  (inductiveNames: nidx {numInductives})
  (numConstructors : nat) 
  (constructorNames : nidx {numConstructors}) 
  (uparams: nidx*)

Constructor ::= 
  "#CTOR"
  (name : nidx) 
  (type : eidx) 
  (parentInductive : nidx) 
  (constructorIndex : nat)
  (numParams : nat)
  (numFields : nat)
  (uparams: nidx*)

Recursor ::= 
  "#REC"
  (name : nidx)
  (type : eidx)
  (numInductives : nat)
  (inductiveNames: nidx {numInductives})
  (numParams : nat)
  (numIndices : nat)
  (numMotives : nat)
  (numMinors : nat)
  (numRules : nat)
  (recRules : ridx {numRules})
  (k : 1 | 0)
  (uparams : nidx*)
```
