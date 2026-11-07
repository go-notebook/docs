---
title: "R cells"
weight: 27
description: "A ```r block is an R cell: parsed with go-embedded-r's parser, run on its evaluator in one shared environment, and bounded in memory, steps and time."
tags: [cells, r]
---

A ` ```r ` block is an R cell (parsed since go-notebook v0.8.0, run since
v0.10.0). The R is [go-embedded-r](https://github.com/go-embedded-r/r), a
pure-Go R: go-notebook starts no external `R`. Its parser checks the cell and
its evaluator runs it. Since v0.12.0 every R cell is bounded (see
[Limits](#limits)). go-notebook v0.13.0 runs go-embedded-r v0.6.0: environments
as values, the apply family and the string functions (v0.5.0), and regular
expressions that follow R's TRE (`perl=TRUE` and approximate matching are
refused).

```r
xs <- c(2.1, 3.4, 1.9, 5.6)
total <- sum(xs)
total / length(xs)
```

## Running

**One environment.** The R cells of a notebook share one global environment,
so a variable assigned in one cell is there in the next, as in an R console.
A cell re-runs only when it or an R cell it reads changes.

**Output.** What a cell prints is its output, as an R console shows it:
`print`, `cat`, and the value of every visible top-level expression. An
invisible value (`x <- 1`, `invisible(x)`) prints nothing. `message()` output
and the cell's warnings go to the standard error, the warnings in R's own form
(`Warning message:` then `In f(x) : text`; `Warning messages:` and a numbered
line each when there are several). An error fails the cell as
`Error in f(x) : text`, after the output written before it, and the cells that
read it do not run. A cell that goes over a limit fails the same way.

A cell shows text only, no figure.

## Limits

A notebook may come from anyone, so each R cell runs with its own bounds:

| Limit | Default | Ends a cell with |
| --- | --- | --- |
| memory | 512 MiB of vectors allocated in total over the cell, garbage included | `Error: memory limit exceeded: more than 536870912 bytes of vectors allocated` |
| steps (calls plus loop iterations) | 50 million | `Error: evaluation step limit reached` |
| time | 30 s | `stopped: the R cell ran longer than 30s, the limit` |

The step count is measured: `for (i in 1:n) s <- s + i` stops at 16,666,666
iterations under 50 million steps, so a simulation of that size runs and an
endless loop ends in seconds. The session and its variables survive a limit.

A program that embeds the kernel can change them with `kernel.Options.R`: a
zero field is the default, and `-1` is no bound. `--timeout` and `--memory`
apply to R cells as to every cell, and they kill the worker, session
included.

## Parsing

An R cell is parsed by go-embedded-r's port of R 4.6's grammar, which gives
the same trees and tokens as R 4.6.2 on all of R's own `src/library` and
`tests` (932 files). A syntax error is reported at the notebook's line and
column, with R's own message:

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

## Names

An R cell joins the [rules of cells](cells.md) and the dependency graph. Its
names follow R's scoping at the top level of the environment the cell runs in:

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

**R variables stay in R.** They are not shared with Go, Ruby, Julia or C cells
in either direction yet: a Go cell that reads an R name finds it undefined,
and an R cell does not see a Go variable.

## In the browser

The [playground](https://go-notebook.github.io/playground/) does not run R
cells: its WebAssembly cores do not link the evaluator. It parses and checks
them: the + menu and the cell's language menu offer R, its badge reads
**R · not runnable yet**, a syntax error is shown at its line in R's words,
and the cells that read it are blocked, in the page's neutral style and not as
failures. Run the notebook with `notebook run` to execute its R cells.
