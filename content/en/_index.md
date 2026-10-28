---
title: "go-notebook documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "Go notebooks with native speed, kept values and a reproducible order."
---

**A notebook is a Markdown file whose ` ```go ` blocks are cells**, and its
` ```ruby ` blocks too ([Ruby cells](ruby.md)); ` ```julia ` blocks are parsed
and checked, not run yet ([Julia cells](julia.md)). The go-notebook kernel:

- **compiles it** into a real Go program, so cells run at native speed;
- **orders the cells by what they use**, worked out from the code itself as
  [marimo](https://marimo.io/) does for Python, not by the order you ran them in;
- **runs only what changed.** A cell whose code and inputs did not change keeps
  its result, and its values are handed over instead of being computed again.
  That holds across rebuilds and across runs: results and values are kept on
  disk.

To try one without installing anything, open the
[playground](https://go-notebook.github.io/playground/): notebooks with Go and
Ruby cells, run in WebAssembly in the browser, with no server.

## Install

```sh
go install github.com/go-notebook/notebook/cmd/notebook@latest
```

The module needs Go 1.27.2 or later.

## A first notebook

````markdown
# Spectrum

```go
import "math"

n := 1_000_000
```

The sum below runs once; editing the last cell does not run it again.

```go
sum := 0.0
for i := 1; i <= n; i++ {
	sum += 1 / float64(i*i)
}
```

```go
math.Sqrt(6 * sum)
```
````

Run it, change the last cell to `math.Pi - math.Sqrt(6*sum)`, and run it again:

```console
$ notebook run spectrum.md
── cell 0 (line 4, ran in 12µs)
── cell 1 (line 12, ran in 2.663ms)
── cell 2 (line 19, ran in 36µs)
   3.1415916986605086
$ notebook run spectrum.md
── cell 0 (line 4, kept)
── cell 1 (line 12, kept)
── cell 2 (line 19, ran in 42µs)
   9.54929284535666e-07
```

The second run is a new process: `sum` comes back from the kernel's store, it
is not summed again. (A real transcript, from `notebook` built at main
`fd7f8c1` on an Apple M4 Max; timings vary by machine.)

## Where to go next

| Page | What it covers |
| --- | --- |
| [Notebooks](notebooks.md) | what a notebook file is, and why it is Markdown |
| [Cells](cells.md) | the rules a cell follows, and the errors that enforce them |
| [Ruby cells](ruby.md) | Ruby cells with the go-scikit bindings, how values cross between Go and Ruby, and the limits |
| [Julia cells](julia.md) | Julia cells: parsed and checked with go-embedded-julia's parser, their names in the graph — not run yet |
| [Using the playground](playground.md) | the browser playground: editing, adding, running, switching language, deleting and moving cells, with Jupyter's keys; its limits |
| [irb](irb.md) | `notebook irb`: an interactive Ruby session on the same Ruby and bindings, as irb |
| [The notebook command](cli.md) | `run`, `html`, `watch`, `serve` and `irb`, and their flags |
| [The kernel](kernel.md) | how the kernel decides what to run, and what it keeps |
| [Design](design.md) | what exists already, what each gets wrong, and the decisions behind go-notebook |

go-notebook is part of an ecosystem with
[go-ndarray](https://go-ndarray.github.io/) (NumPy-style arrays) and
[go-fft](https://go-fft.github.io/) (`numpy.fft`). The source is at
[github.com/go-notebook/notebook](https://github.com/go-notebook/notebook),
under the BSD-3-Clause licence.
