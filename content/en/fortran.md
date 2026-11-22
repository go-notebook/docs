---
title: "Fortran cells"
weight: 28
description: "A ```fortran block is a Fortran cell: checked and run with go-embedded-fortran, in one session shared by the Fortran cells, kept apart from the other languages."
tags: [cells, fortran]
---

A ` ```fortran ` block is a Fortran cell (checked since go-notebook v0.16.0,
run since v0.17.0). The Fortran is
[go-embedded-fortran](https://github.com/go-embedded-fortran/fortran) (v0.17.0),
a pure-Go Fortran: no external compiler is started. Its `sema` session checks
the cell and gives the names and the errors, its `eval.Session` runs it. A
cell is a program's loose material and units alike: declarations and
statements at top level, subroutines, functions, modules, derived types, even
a `program`. Free form, implicit typing off.

## Cells share one session

A notebook with Fortran cells makes the worker import go-notebook's `fcells`
package, which owns one `eval.Session`. A Fortran cell sees the variables
(with the values the earlier cells left), procedures, modules and derived
types of the Fortran cells before it, in notebook order: `n = n + 1` in a cell
after the one that has `integer :: n = 3` changes the same `n`. A cell with an
error leaves nothing behind for the next ones. Names are case-insensitive.
Redefinition shadows in Fortran (a cell that runs again defines a new version
that later cells see); between two cells of the notebook it is refused, as
go-notebook's rule of one definition per name.

The names a cell reads from earlier Fortran cells are its uses and give the
graph its edges: a variable, a procedure, a derived type named in a
declaration and a module named in `use` all do.

## Fortran variables stay in Fortran

They are not shared with Go, Ruby, Julia, R or C cells in either direction yet
(`analysis.Isolated`, as R's): a Go cell that reads a Fortran name finds it
undefined, and a Fortran cell does not see a Go variable. A result reaches the
reader as printed text; a cell shows text only.

## Output and errors

What a cell prints is its output; standard error and the text of `STOP` are its
standard error; `STOP` ends the cell, not the session. A refused cell or a
failure is reported at `line L, column C` of the cell, and a cell that fails
while it runs keeps what it had done.

## Limits

A notebook may come from anyone. A cell may make at most 2^32 DO iterations and
procedure calls, run 30 s, and the evaluator's own bounds apply: 2^28 elements
per array, 2^26 characters per string, 2^15 nested calls. Output is cut at
16 MiB per stream. The step count, not a timer, is what ends a loop in
WebAssembly. `--timeout` and `--memory` apply as to every cell and kill the
worker, session included. `INCLUDE` is refused: no file is read.

## What the evaluator refuses

It says so, positioned: type extension and `select type`, pointer components,
vector subscripts, `elsewhere`, `dim=` and `mask=`, complex variables,
character arrays, `read` from a unit, and an array declared with an
initialisation.

## The playground

Fortran cells run in the browser playground, with no server and no compiler:
the evaluator is linked into the playground's second WebAssembly core (the
one it loads for a notebook with a Fortran, R or Ruby cell; a Julia cell loads a
third core that has it too), through
`fcells.Session.Eval`; the default core stays Go-only. The playground bounds a
cell tighter than the kernel does: 200 million DO iterations and procedure
calls, 10 s and 16 MiB of output per stream. An endless `do` ends with
`step limit reached: the cell ran too many loop iterations and calls` and the
page stays usable. The second core grew by 1.3 MB (0.28 MB gzipped).

Since playground v0.22.0 (go-embedded-fortran v0.31.0), constructs the
evaluator used to run with wrong results are refused with a message instead:
types with final procedures or generic assignment, operator or I/O bindings,
assumed-length `character(*)` function results, and `value` on non-scalar or
derived dummies. The same release adds `Gw.0` output, re-initialised
`intent(out)` derived dummies, BOZ conversions as bits, and the declared bounds
of explicit-shape dummies.

[examples/fortran.md](https://github.com/go-notebook/notebook/blob/main/examples/fortran.md)
runs end to end: `notebook run examples/fortran.md`.
