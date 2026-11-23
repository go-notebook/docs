---
title: "Julia cells"
weight: 26
description: "A ```julia block is a Julia cell: parsed with go-embedded-julia's parser, its names in the dependency graph, and run by its interpreter in one kernel."
tags: [cells, julia]
---

A ` ```julia ` block is a Julia cell (parsed since go-notebook v0.6.0, **run**
in go-notebook v0.18.0; on go-embedded-julia v0.25.0 since v0.24.0).
[go-embedded-julia](https://github.com/go-embedded-julia/julia) is the
pure-Go Julia these cells run on: its parser for the names and the syntax
errors, its interpreter (v0.25.0) for the running. No external `julia` is
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

(A real transcript, from the change that became go-notebook v0.18.0, on an Apple M4 Max; the
first cell pays the kernel's cold start.)

An exception fails the cell with julia's text of it after `ERROR:`, after the
output the cell wrote before it; the cells that read it do not run, and the
kernel keeps what it had.

**The image.** The kernel boots from go-embedded-julia's prebuilt Base image,
which is not part of the module. The first Julia cell a worker runs fetches the
image of release v0.25.0 (25,695,765 bytes) and loads it only if its SHA-256 is the one go-notebook
pins (`a1460281911fa0bc0ec8c9b42180f698998cea9b41e51a4c95237b6c75c007db`); the verified file is kept in your user cache directory
(`go-embedded-julia/images`) and checked again each time. The fetch is bounded
to five minutes and tried once. The cold start is about a second. On a machine
that is offline and has no cached image every Julia cell of the run fails at
once with *the Julia kernel could not start*, and the other cells run.

**Limits.** A notebook may come from anyone, so each cell is bounded, as an R
cell is: 30 seconds, 50 million statements, a memory budget of 512 MiB, and
16 MiB of output per stream. The memory budget counts the elements of the arrays
and dictionary tables a cell allocates, in total and garbage included, at 16
bytes each. It is a floor, not the heap: a boxed value costs more, strings are
not counted, and one array may not have more than 2^24 elements whatever the
budget. A cell over a limit is stopped and says which, in the kernel's words;
`--timeout` and `--memory` still apply to the worker. The kernel is sandboxed:
its file system holds julia's own tree and its environment is empty.

**Soft scope.** As in IJulia, a loop at the top level that updates a global
works: `s=0; for i in 1:10; s+=i; end` gives 55. A `MethodError` shows Julia's
own text (no method matching, closest candidates).

Since go-notebook v0.21.0, a Float64 `A * x` runs through
[go-blas](https://github.com/go-blas/blas), a non-ASCII `uppercase`,
`lowercase` or `normalize` works, and an n×0 matrix crosses.

Since go-notebook v0.23.0, a matrix product such as `A * A` and its display run
within the default limits. Showing a Float64 matrix runs julia's type
inference, which inside the interpreter cost about 70 million statements on a
kernel's first matrix; the v0.24.0 image ships that inference already done
([#119](https://github.com/go-embedded-julia/julia/issues/119)). Displaying a
matrix of another element type (Int, Float32, Complex) may still pay it the
first time.

Since go-notebook v0.24.0 (go-embedded-julia v0.25.0), LU and Cholesky
factorizations work, forwarded to [go-blas](https://github.com/go-blas/blas)
in Float64 and Float32: `lu(A)`, `A \ b` on a general square matrix,
`factorize`, `cholesky(A)` and `cholesky(A) \ b`. A singular matrix raises
`SingularException` and a matrix that is not positive definite raises
`PosDefException`, as in julia.

````markdown
```julia
A = [4.0 3.0; 6.0 3.0]
b = [10.0, 12.0]
x = A \ b
```
````

**Known limitations** (go-embedded-julia v0.25.0):

- `inv`, `qr`, `eigen`, `svd`, triangular solves such as
  `UpperTriangular(A) \ B`, and complex matrices are refused for now
  ([#67](https://github.com/go-embedded-julia/julia/issues/67)).
- Seeded random streams and SHA hashes still differ from julia's
  ([#126](https://github.com/go-embedded-julia/julia/issues/126)).

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

**Across languages.** Since go-notebook v0.19.0, Julia variables cross with Go
cells in both directions (`Kernel.Get`/`Set`), and Julia is no longer
`Isolated`: a Go cell that reads a Julia name depends on the Julia cell. This
breaks v0.18.0, where Julia variables stayed in Julia.

- Types that cross: Int64, Float64 (bits kept, NaN payloads and -0.0
  included), Bool, String, and Vector and Matrix of Float64 or Int64
  (column-major). Go `int` and `[]int` cross as Int64.
- A Julia value of another type stays in Julia; the Go cell that reads it is
  deferred, naming the type.
- A Go value Julia cannot take fails the Julia cell that reads it.
- Inputs are measured before any copy: one array has at most 2^24 elements, and
  a cell's inputs together are at most its memory limit. A cell can therefore
  hold up to twice its limit: its inputs plus its own allocations.

Between Julia cells every name is shared. The HTML page shows a Julia cell highlighted with
go-rouge's `julia` lexer.

## In the browser

Since playground v0.18.0 the [playground](https://go-notebook.github.io/playground/)
runs Julia cells (playground v0.24.0: go-embedded-julia v0.25.0) compiled to WebAssembly, with no
external `julia`. It is the heaviest runtime the page has, so it is **loaded on
first use**: a Julia cell's badge reads *Julia · loads when run (~28 MB)*, and
Run downloads a third core (20 MB gzipped, the second core plus the interpreter)
and the Base image (7.5 MB gzipped, checked against its pinned SHA-256 before it
is decoded), then boots the kernel: about **28 MB and about 20 s** the first
time, shown on the cell as it loads, then cached by the browser. Opening a
notebook, and running the cells above its first Julia cell, download nothing for
Julia; the default, Ruby, R and Fortran cores never carry it. If the load fails
(network, memory) the Julia cells say so and the other languages run.

- **Limits**: **512 MiB** of arrays, **20 million statements**, **20 s** (R's is
  10 s: a Julia cell compiles what it uses the first time) and 16 MiB of output.
  A cell that goes past one is stopped with a message that says which, and the
  worker is not killed. Deep recursion is a Julia `StackOverflowError`, not a
  lost core. Since playground v0.23.0, `A = [1.0 2.0; 3.0 4.0]; B = A * A`
  shows `B` within these limits on a fresh kernel (it went past the 20 million
  statements before). Since playground v0.24.0, a 2×2 `A \ b` runs within
  them too.
- **Julia is isolated from Go in the page.** The variables that cross in
  `notebook run` (above) do not cross in the page, in either direction: a cell
  that reads one fails with the other language's own message and a note saying
  why. Julia variables persist from one Julia cell to the next.
- **Each run starts from a reset kernel.** Since playground v0.21.0, a run resets
  the kernel the page booted (`Kernel.Reset`,
  [go-embedded-julia#93](https://github.com/go-embedded-julia/julia/issues/93))
  instead of replacing the worker. A rerun takes 25 to 60 ms, and over 30 runs
  the page's memory stays flat. The reset undoes what the earlier run changed,
  and it stops the tasks and timers that run started, so an `@async` loop or a
  periodic `Timer` never prints into the next run. A `struct` a notebook defines
  is not a redefinition the second time it runs. The worker is replaced only
  when a reset has to load the image again or fails.

Try the [Julia example notebook](https://go-notebook.github.io/playground/#4-julia).
