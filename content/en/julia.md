---
title: "Julia cells"
weight: 26
description: "A ```julia block is a Julia cell: parsed with go-embedded-julia's parser, its names in the dependency graph, and run by its interpreter in one kernel."
tags: [cells, julia]
---

A ` ```julia ` block is a Julia cell (parsed since go-notebook v0.6.0, **run**
on go-notebook's main branch, in the release after v0.17.0).
[go-embedded-julia](https://github.com/go-embedded-julia/julia) is the
pure-Go Julia these cells run on: its parser for the names and the syntax
errors, its interpreter (v0.15.0) for the running. No external `julia` is
started.

## Running

The cells of a notebook share one Julia kernel, in the worker. A global, a
function or a struct a cell defines is there in the next cell; what a cell
prints is its own output, and the value of its last expression is shown as the
Julia REPL shows it, unless it is `nothing` or the cell ends in a semicolon.

````markdown
```julia
square(x) = x^2
total = sum(square(i) for i in 1:4)
```

```julia
println("total = ", total)
total + 1
```
````

```console
$ notebook run julia.md
── cell 0 (line 4, ran in 1.102209s)
   30
── cell 1 (line 9, ran in 1.478ms)
   total = 30
   31
```

(A real transcript, from go-notebook's main branch on an Apple M4 Max; the
first cell pays the kernel's cold start.)

An exception fails the cell with julia's text of it after `ERROR:`, after the
output the cell wrote before it; the cells that read it do not run, and the
kernel keeps what it had.

**The image.** The kernel boots from go-embedded-julia's prebuilt Base image,
which is not part of the module. The first Julia cell a worker runs fetches the
image of release v0.15.0 and loads it only if its SHA-256 is the one go-notebook
pins; the verified file is kept in your user cache directory
(`go-embedded-julia/images`) and checked again each time. The cold start is
about a second. On a machine that is offline and has no cached image every
Julia cell fails with *the Julia kernel could not start*, and the other cells
run.

**Limits.** A notebook may come from anyone, so each cell is bounded, as an R
cell is: 30 seconds, 50 million statements, 512 MiB of array and dictionary
elements, and 16 MiB of output per stream. A cell over a limit is stopped and
says which; `--timeout` and `--memory` still apply, and cancelling the run
stops a running cell. The kernel is sandboxed: its file system holds julia's own
tree and its environment is empty.

**Known limitations** (go-embedded-julia v0.15.0):

- At the top level of a cell, a loop that updates a global raises
  `UndefVarError`, where julia 1.12's REPL updates it
  ([#54](https://github.com/go-embedded-julia/julia/issues/54)). Put the loop in
  a function.
- Some exceptions, `MethodError` among them, show as their Go representation,
  not as julia's message
  ([#55](https://github.com/go-embedded-julia/julia/issues/55)).
- A process loads one image release
  ([#49](https://github.com/go-embedded-julia/julia/issues/49)).

## Parsing and names

The names below are the same whether a cell runs or not.

**Parsing.** A Julia cell is parsed by go-embedded-julia's port of
JuliaSyntax, the parser julia 1.12 ships, which gives the same `Expr` trees as
julia on its whole test corpus. A syntax error is reported at the notebook's
line and column, with julia's own message:

````markdown
```julia
y = 2
x = (1,
```
````

```console
$ notebook run broken.md
── cell 0 (line 4, not run)
   error: 5:8: cell 0: syntax error: Expected `)` or `,`
notebook: 1 cell(s) failed
```

The parse is bounded — 4 MiB per cell, a nesting depth of 1000 — so a
notebook from anyone cannot make it run long or exhaust memory; a cell over
the limits fails with a message saying so.

**Names.** A Julia cell joins the [rules of cells](cells.md) and the
dependency graph: one definition per name, no assignment to another cell's
variable, a name starting with `_` kept in its cell. Its names follow Julia's
own scope rules for an interactive context, as written in
[Scope of Variables](https://docs.julialang.org/en/v1/manual/variables-and-scoping/)
(Julia manual, 1.12):

| Construct | In the notebook |
| --- | --- |
| `x = 1`, `a, b = …`, `x::Int = 1`, `global x = 1` at the top level, or in `if`/`begin` there | defines the variable |
| `f(x) = …`, `function f … end`, `macro m … end` (as `@m`) | defines a function or macro |
| `const C = …`, `struct S`, `abstract type`, `primitive type`, `module M` | defines a constant or type |
| `using A`, `using A: x, y`, `import A.B`, `import A as B` | defines what it binds (`A`; `x` and `y`; `B`) |
| `function`, `macro` and `struct` bodies, `let`, `do`, `->`, comprehensions and generators | hard scopes: an assignment there makes a local |
| `for`, `while`, `try` at the top level | soft scopes: an assignment there to an existing global assigns it (as in the REPL and IJulia); otherwise it makes a local of the loop |
| a name read before the cell assigns it, in any scope where no local binds it | a use: the cell depends on the cell that defines it |

These rules are checked against julia itself: 61 cells, from each kind of
definition to keyword defaults, `where` bounds and nested generators, are
lowered by julia 1.12.6 (through `REPL.softscope`, the soft-scope rule of the
REPL), and the globals julia declares and reads are the names go-notebook
finds.

**Across languages.** Julia variables stay in Julia, as R's do: they are not
shared with Go, Ruby, R, C or Fortran cells in either direction yet. A Go cell
that reads a Julia name finds it undefined, and a Julia cell does not see a Go
variable; a Julia result reaches the reader as printed text. Between Julia
cells every name is shared. The HTML page shows a Julia cell highlighted with
go-rouge's `julia` lexer.

## In the browser

The [playground](https://go-notebook.github.io/playground/) does not run Julia
cells: its WebAssembly core parses and checks them, the cell's badge reads
**Julia — not runnable yet**, and the cells that read it are blocked. Running
them there is a separate question, pending with go-embedded-julia.

Try the [Julia example notebook](https://go-notebook.github.io/playground/#4-julia)
there for the checks; `notebook run` and `notebook serve` run the cells.
