---
title: "R cells"
weight: 27
description: "A ```r block is an R cell: parsed and checked with go-embedded-r's parser, its names in the dependency graph — but not run yet."
tags: [cells, r]
---

A ` ```r ` block is an R cell (go-notebook v0.8.0 and later). R cells are
**parsed and checked, but not run yet**:
[go-embedded-r](https://github.com/go-embedded-r/r), the pure-Go R these
cells are written for, has a complete parser today (its v0.1.0), and its
evaluator is the next milestones (M2 and M3).

## What works

**Parsing.** An R cell is parsed by go-embedded-r's port of R 4.6's grammar,
which gives the same trees and tokens as R 4.6.2 on all of R's own
`src/library` and `tests` (932 files). A syntax error is reported at the
notebook's line and column, with R's own message:

````markdown
```r
y <- 2
x <- y y
```
````

```console
$ notebook run broken.md
── cell 0 (line 4, not run)
   error: 5:8: cell 0: syntax error: unexpected symbol
notebook: 1 cell(s) failed
```

The parse is bounded (4 MiB per cell, a nesting depth of 1000, a million
nodes), so a notebook from anyone cannot make it run long or exhaust memory;
a cell over the limits fails with a message saying so.

**Names.** An R cell joins the [rules of cells](cells.md) and the dependency
graph. Its names follow R's scoping at the top level of the environment the
cell runs in:

| Construct | In the notebook |
| --- | --- |
| `x <- v`, `x = v`, `v -> x`, `"x" <- v` | defines `x` |
| `assign("x", v)` with a literal name | defines `x` |
| `x$a <- v`, `x[i] <- v`, `names(x) <- v` | defines `x`, and reads the `x` it modifies |
| `for (i in s)` at the top level | defines `i` |
| `f <- function(…) …`, `f = \(…) …` at the top level | defines the function `f`: it stays R's |
| a free variable of the cell, or of a function's body, that the cell does not assign | a use: the cell depends on the cell that defines it |
| the parameters and local assignments of a function, and what `local()` assigns | stay inside it |

**Dynamic cells.** Some R cannot be analysed from its text: `x <<- v`,
`assign` or `get` with a computed name, `eval`, `parse`, `source`, `attach`,
`library`, `require`, `environment<-`. Such a cell is **dynamic**: it depends
on every cell before it, and every later cell that reads a name depends on
it, so nothing it may define is missed.

**Across languages.** An R cell's variables are seen by Go, Ruby and Julia
cells, and an R cell sees theirs, as between Ruby and Go (see
[Ruby cells](ruby.md)); a function assigned at the top level stays in R. A
variable named like a Go keyword (`type`) is not seen by Go cells.

## What does not work yet: running

Running an R cell fails, clearly, and the cells that need it say why:

````markdown
```go
n := 10
```

```r
square <- function(v) v^2
total <- sum(square(seq_len(n)))
```

```go
total + 1
```

```go
n * 2
```
````

```console
$ notebook run r.md
── cell 0 (line 4, ran in 11µs)
── cell 1 (line 8, not run)
   error: R cells are parsed and checked, but not run yet: execution arrives with go-embedded-r's evaluator (milestones M2 and M3)
── cell 2 (line 13, not run)
   error: blocked: it depends on cell 1, an R cell, and R cells are not run yet
── cell 3 (line 17, ran in 24µs)
   20
notebook: 2 cell(s) failed
```

(Real transcripts, from go-notebook v0.8.0.)

- an R cell never runs, and `notebook run` exits non-zero;
- every cell that reads what an R cell defines, directly or through other
  cells, in any language, is **blocked**, not silently skipped;
- the other cells run as usual.

go-notebook starts no external `R`: the cells will run on go-embedded-r's own
evaluator when it exists. The HTML page shows an R cell's source uncoloured
(go-rouge has no R lexer).

## In the browser

The [playground](https://go-notebook.github.io/playground/) (v0.9.0 and later)
parses and checks R cells in the same way, in WebAssembly: the + menu and the
cell's language menu offer R, its badge reads **R · not runnable yet**, a
syntax error is shown at its line in R's words, and the cells that read it
are blocked, in the page's neutral style and not as failures.

The playground's R example notebook (`#5-r`) has an R cell, a Go cell blocked
on it and a Go cell that runs alone.
