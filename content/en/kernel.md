---
title: "The kernel: what runs, and what is kept"
linkTitle: "Kernel"
weight: 40
description: "How the kernel orders the cells, decides which to run, and keeps results and values across rebuilds and runs."
tags: [kernel]
---

Each run takes the notebook's current code cells and does **the least work
that brings every result up to date**. Four steps get there: the analysis of
the cells, the generated program, the cell keys, and the value store.

## The dependency graph

Package [`analysis`](https://github.com/go-notebook/notebook/tree/main/analysis)
finds each cell's **definitions** and **free identifiers** from the parse alone.
It needs no type information, so it works on code that does not compile yet.

A cell depends on the cells that define the names it reads. That gives the
graph, the order of execution, and the static errors: **cycles**, **double
definitions** and **assignments to another cell's variable** (see
[Cells](cells.md#the-three-rules)). The order is therefore a function of the
code, never of the order cells were run in or appear in.

## The generated program

Package [`gen`](https://github.com/go-notebook/notebook/tree/main/gen) writes the
Go program the kernel compiles:

- each cell becomes a function;
- its variables become package-level variables, so every cell can read them and
  their values outlive the call. That needs their types, so `gen` type-checks
  the cells one by one, in dependency order, with `go/types`;
- the cells' text is kept as it is and edited in place: `:=` becomes `= `,
  `var x T` becomes `x = *new(T)` (so running the cell again resets it),
  constants move to package level, and the display value is wrapped in a call
  that shows it;
- `//line nb.md:L:C` directives point every line back into the notebook.

The kernel compiles this program into a **worker process** and drives it
through package [`runtime`](https://github.com/go-notebook/notebook/tree/main/runtime):
one JSON request per line on the worker's standard input (`run`, `get`, `set`,
`exit`) and one reply per line on its standard output. What a cell prints goes
to a pipe of its own, so it can never be mistaken for a reply.

Go's build cache makes rebuilding after a one-cell edit cost about a tenth of a
second.

## A cell's key

A cell's **key** is a hash of:

- its source;
- the import paths of the packages it refers to;
- the keys of the cells it reads.

Its position is not part of the key, so **moving cells runs nothing**; neither
does an import another cell adds. Because a key includes the keys of its
inputs, a change propagates to exactly the cells downstream of it.

## What runs

| A cell whose key… | … |
| --- | --- |
| did not change | **keeps its result** and is not run |
| changed | runs, and so does every cell whose key changes because of it |
| did not change, but made a **volatile** value that a running cell reads | runs again, to make that value (see below) |
| reads a cell that failed | is **not run**, and reports `not run: cell N failed` |

When the program changes, the worker is rebuilt and restarted. The variables
made by unchanged cells are **handed to the new worker** instead of being
computed again.

A worker that dies (`os.Exit`, a crash outside the recovered call) is restarted
on the next run.

## Kept values

To hand a value over, or to keep it on disk, the kernel encodes it:

- with its own `MarshalBinary` when it has one;
- with gob when every field it can hold is exported (gob would silently drop
  the others).

Anything else — a func, a channel, unexported state — is **volatile**: its cell
runs again, but only when a cell that reads it has to run.

## On disk

Results and values are kept in the **work directory**, so a new kernel, or the
next `notebook run`, starts where the last one stopped. The
[`notebook` command](cli.md#the-work-directory) gives each notebook its own
work directory under the user's cache directory.

## The assumption

All of this assumes **a cell computes the same values from the same inputs**.
A cell that reads the clock or an unseeded random generator does not; such a
cell should be run explicitly — `Kernel.RunCell` runs a cell again even if
nothing changed, together with every cell that reads it.

The key does not yet include module versions: a `go get -u` does not invalidate
results. [go-notebook/notebook#3](https://github.com/go-notebook/notebook/pull/3)
adds the Go toolchain version and the work module's `go.sum` to the key.

## Using the kernel from Go

```go
k, err := kernel.New(kernel.Options{Dir: work, Name: "analysis.md"})
if err != nil { … }
defer k.Close()

rep, err := k.Run(cells) // cells: []gen.Code{{Source: …, Line: …}, …}
for _, r := range rep.Results {
	// r.Ran, r.Took, r.Output, r.Display, r.Error
}
```

`Report.Built` says whether the worker was rebuilt, `Report.Build` how long
generating and building took, and `Report.Diags` holds the static errors.
