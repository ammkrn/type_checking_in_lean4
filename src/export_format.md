# Export format

An exporter is a program which emits Lean declarations using the kernel language, for consumption by external type checkers. Producing an export file is a complete exit from the Lean ecosystem; the data in the file can be checked with entirely external software, and the exporter itself is not a trusted component. Rather than inspecting the export file itself to see whether the declarations were exported as the developer intended, the exported declarations are checked by the external checker, and are displayed back to the user by a pretty printer, which produces output far more readable than the export file. Readers can (and are encouraged to) write their own external checkers for Lean export files.

The official exporter is [lean4export](https://github.com/leanprover/lean4export).

Current versions of lean4export use an ndjson format now specified [here](https://github.com/leanprover/lean4export/blob/master/format_ndjson.md)


