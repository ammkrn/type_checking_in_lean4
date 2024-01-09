# Names

The first of the kernel's primitive types is `Name`, which is sort of what it sounds like; it provides kernel items with a way of addressing things.

```
Name ::= anonymous | str Name String | num Name Nat
```

Elements of the `Name` type are displayed as dot-separated names, which users of Lean are probably familiar with. For example, `num (str (anonymous) "foo") 7` is displayed as `foo.7`. 

# Implementation notes

The implementation of names assumes UTF-8 strings, with characters as unicode scalars (these assumptions about the implementing language's string type are also important for the string literal kernel extension). 

Some information on the lexical structure of names can be found [here](https://github.com/leanprover/lean4/blob/504b6dc93f46785ccddb8c5ff4a8df5be513d887/doc/lexical_structure.md?plain=1#L40)

The exporter does not explicitly output the anonymous name, and expects it to be the 0th element of the imported names.


