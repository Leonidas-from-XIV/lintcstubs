![Build and test](https://github.com/edwintorok/lintcstubs/actions/workflows/workflow.yml/badge.svg)

Lintcstubs — Static analyzer for OCaml C primitives
===================================================

These are a suite of tools for finding bugs in OCaml C stubs.

* `lintcstubs_genwrap` generates `__wrap_` wrappers all C primitives with assertion checks for the type of arguments and return values.
* `lintcstubs_genmain` generates a `main` function that spawns threads and calls all C primitives.
* `lintcstubs` a tool containing a static analysis pass for detecting race conditions from the incorrect use of the OCaml runtime lock

Currently they require OCaml 4.14+.

# Installation

## Using `opam`

```
opam install lintcstubs
```

# Why?

Static analyzers built for C won't know about the special rules for OCaml GC safety, and they won't know how the OCaml values are laid out in memory.

The wrapper generated by `genwrap` can be used at runtime (to validate that the returned value have the right "shape"), by using `binutils`'s `-wrap` feature.
It can also be used by a static analyzer to detect issues at build time (e.g. returning an integer, but forgetting to wrap it with `Val_int`).

Static analyzers work better when given an entire program, thus `lintcstubs_genmain` is useful to generate an entrypoint.
This avoids false positives about `NULL` dereferences (OCaml values are never `NULL`, the runtime will raise an exception instead).
The generated main function also makes it explicit that these functions are invoked in a multi-threaded environment.

# Caveats

* only some very basic shapes are supported currently (primitive types, arrays, tuples). When the shape is unknown no assertion check is done for that parameter/field.
* Is_block() tracking isn't precise for allocated values
* write some examples
* show example on how to use with GobPie

# Usage:

Consult the [official](https://v2.ocaml.org/manual/intfc.html#ss:c-prim-impl)
[manual](https://v2.ocaml.org/manual/intfc.html#ss:c-unboxed) on how to implement C primitives correctly.


## For static analysis

```
lintcstubs_arity_cmt ocamlfile.cmt >primitives.h
lintcstubs_genwrap ocamlfile.cmt >test_analyze.c
lintcstubs_genmain ocamlfile.cmt >>test_analyze.c
```

`ocamlfile.cmt` is a `-bin-annot` file for `ocamlfile.ml`. Your build system should've produced one, check that it is using the `-bin-annot` flag if not.
You can also create one manually (add include and `ocamlfind` flags as necessary):
```
ocamlc -bin-annot -c ocamlfile.ml
```

Then run your static analyzer (in this case the  `goblint` based `lintcstubs`):
```
lintcstubs --disable warn.integer --disable warn.info --disable warn.deadcode test_analyze.c ocamlfile_stubs.c
```

Where `ocamlfile_stubs.c` is your C primitives implementation corresponding to `ocamlfile.ml`.

## For runtime checking

This requires the `binutils` linker:

```
lintcstubs_genwrap ocamlfile.cmt >test_wrap.c
ocamlc -c test_wrap.c
ocamlopt ocamlfile.ml test_wrap.o test_stubs.o -ccopt -Wl,-wrap,stub1,-wrap,stub2
```

Where `stub1`, `stub2`, etc. are the names of your C primitive implementations.

This mode is still under development.

# How it works

This is part of a suite of static analysis tools for C stubs described in a [paper](https://arxiv.org/abs/2307.14909) submitted to the ICFP 2023 OCaml workshop.