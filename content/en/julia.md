---
title: "Julia cells"
weight: 26
description: "A ```julia block is a Julia cell: parsed and checked with go-embedded-julia's parser, its names in the dependency graph — but not run yet."
tags: [cells, julia]
---

A ` ```julia ` block is a Julia cell (go-notebook v0.6.0 and later). Julia
cells are **parsed and checked, but not run yet**:
[go-embedded-julia](https://github.com/go-embedded-julia/julia), the pure-Go
Julia these cells are written for, has a complete parser today, and its
interpreter is being written. The
[plan](https://github.com/go-scikit/roadmap/blob/main/bib/julia.md) has the
milestones: M1, the parser, is done; M2 (lowering) is in progress; M3, the
runtime, is the one that makes Julia cells run.

## What works

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

**Across languages.** A Julia cell's variables are seen by Go and Ruby cells,
and a Julia cell sees theirs, as between Ruby and Go (see
[Ruby cells](ruby.md)); functions, macros, constants and types stay in Julia.

## What does not work yet: running

Running a Julia cell fails, clearly, and the cells that need it say why:

````markdown
```go
n := 10
```

```julia
xs = [i^2 for i in 1:n]
total = sum(xs)
```

```go
total + 1
```

```go
n * 2
```
````

```console
$ notebook run julia.md
── cell 0 (line 4, ran in 11µs)
── cell 1 (line 8, not run)
   error: Julia cells are parsed and checked, but not run yet: execution arrives with go-embedded-julia's runtime (milestone M3)
── cell 2 (line 13, not run)
   error: blocked: it depends on cell 1, a Julia cell, and Julia cells are not run yet
── cell 3 (line 17, ran in 28µs)
   20
notebook: 2 cell(s) failed
```

(Real transcripts, from go-notebook's main branch on an Apple M4 Max.)

- a Julia cell never runs, and `notebook run` exits non-zero;
- every cell that reads what a Julia cell defines — directly or through other
  cells, in any language — is **blocked**, not silently skipped;
- the other cells run as usual.

go-notebook starts no external `julia`: the cells will run on
go-embedded-julia's own runtime, in the same worker as Go and Ruby cells,
when it exists. The HTML page shows a Julia cell's source uncoloured (go-rouge
has no Julia lexer yet).

## In the browser

The [playground](https://go-notebook.github.io/playground/) parses and checks
Julia cells in the same way, in WebAssembly: a syntax error is shown at its
line, the cell's badge reads **Julia — not runnable yet**, and the cells that
read it are blocked.
