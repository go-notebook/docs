---
title: "Ruby cells"
weight: 25
description: "A ```ruby block is a cell: Ruby on rbgo with the go-scikit bindings, sharing its variables with the Go cells."
tags: [cells, ruby]
---

A ` ```ruby ` block is a Ruby cell (go-notebook v0.3.0 and later). The Ruby is
[rbgo](https://github.com/go-embedded-ruby/ruby), a Ruby bytecode VM written
in Go, with the [go-scikit bindings](https://github.com/go-scikit/ruby)
registered over the same Go libraries a Go cell imports:

| Ruby | Go library | Counterpart |
| --- | --- | --- |
| `Stats` | [go-statistics/stats](https://github.com/go-statistics/stats) | `scipy.stats` |
| `Special` | [go-numerics/special](https://github.com/go-numerics/special) | `scipy.special` |
| `Optimize` | [go-numerics/optimize](https://github.com/go-numerics/optimize) | `scipy.optimize` |
| `Integrate` | [go-numerics/integrate](https://github.com/go-numerics/integrate) | `scipy.integrate` |
| `DataFrame` | [go-dataframe/dataframe](https://github.com/go-dataframe/dataframe) | `pandas` |
| `Plot` | [go-plot/plot](https://github.com/go-plot/plot) | `matplotlib.pyplot` |
| `NDArray` | [go-ndarray/ndarray](https://github.com/go-ndarray/ndarray) (rbgo's own) | `numpy.ndarray` |

Every binding calls the Go function it is named after; the bindings' README
lists every name.

The kernel adds `github.com/go-scikit/ruby` to the notebook's work module by
itself, and builds the worker with `-tags rbgo_core` (the Ruby language and its
numeric stack, without gems). A notebook needs nothing more than go-notebook.

To try the same Ruby line by line, as irb, without a notebook file, run
[`notebook irb`](irb.md).

## A mixed notebook

A Go cell makes the data, a Ruby cell fits it with a go-scikit binding, and a
Go cell reads the fit back (the file's prose is left out here):

````markdown
```go
xs := []float64{2.1, 3.4, 1.9, 5.6, 4.4, 3.3, 2.8, 4.9}
```

```ruby
fit = Stats::Normal.fit(xs)
loc, scale = fit.loc, fit.scale
ks = Stats.ks_1samp(xs) { |x| fit.cdf(x) }
pvalue = ks.pvalue
fit
```

```go
import "fmt"

fmt.Printf("loc %.4f, scale %.4f, KS p-value %.4f\n", loc, scale, pvalue)
```
````

In a module whose `go.mod` has `tool github.com/go-notebook/notebook/cmd/notebook`
and requires `github.com/go-notebook/notebook v0.3.0`:

```console
$ go tool notebook run first.md
── cell 0 (line 6, ran in 0s)
── cell 1 (line 13, ran in 366µs)
   #<Stats::Normal loc=3.55, scale=1.2359207094308275>
── cell 2 (line 23, ran in 3µs)
   loc 3.5500, scale 1.2359, KS p-value 0.9381
```

In the Ruby cell `xs` is an Array of Floats; in the last Go cell `loc`,
`scale` and `pvalue` are `float64`s. (A real transcript, from go-notebook
v0.3.0 with go-scikit/ruby v0.1.0; the line numbers are those of the full
file.) A larger example, with an NDArray and a plot, is
[examples/mixed.md](https://github.com/go-notebook/notebook/blob/main/examples/mixed.md).

## The rules

Ruby cells follow the [rules of Go cells](cells.md): one definition per name,
no assignment to another cell's variable, and a name starting with `_` stays
in its cell. The names come from rbgo's own parser: a cell defines the locals
it assigns at its top level, and the methods, constants, classes and modules
it defines there; it uses the names it reads before assigning them.

The Ruby cells share one interpreter, so their top-level locals persist from
cell to cell. Dependencies, caching and re-runs work across the two languages
as for Go alone: a Ruby cell runs again only when it, or a cell it reads,
changed.

A cell that ends in an expression shows its value: a `Plot` figure as SVG,
anything else as its `inspect` (an NDArray in NumPy's form). An exception
fails the cell with `file:line: message (Class)` at the notebook's line,
followed by the backtrace frames in the notebook (twelve at most).

## How values cross between Go and Ruby

**Only variables cross.** A Go cell sees a Ruby cell's top-level locals, not
its methods, constants or classes, nor a local named like a Go keyword or
predeclared identifier (`len`, `type`, `int`). A Ruby cell sees a Go cell's
variables, not its funcs or types.

**Go to Ruby.** Before a Ruby cell runs, each Go variable it reads becomes a
Ruby local of the same name (a constant when the name starts with an
upper-case letter):

- numbers, strings and booleans as such;
- slices as Arrays, maps as Hashes;
- an `*ndarray.Array` as the NDArray itself, not a copy;
- any other value (a `*plot.Figure`, a struct) as an object that comes back to
  Go as itself.

**Ruby to Go.** After a Ruby cell runs, each local it assigns becomes a Go
variable whose type is the type of its value:

| Ruby value | Go type |
| --- | --- |
| nil | `any` (nil) |
| true, false | `bool` |
| Integer | `int64` (`*big.Int` when it does not fit) |
| Float | `float64` |
| String | `string` |
| Symbol | `ruby.Symbol` |
| Complex | `complex128` |
| Rational | `*big.Rat` |
| NDArray | `*ndarray.Array`, the same array |
| Array of all Floats, all Integers, all Strings, all booleans | `[]float64`, `[]int64`, `[]string`, `[]bool` |
| any other Array | `[]any` |
| Hash with String keys | `map[string]any` |
| Hash with Symbol keys | `map[ruby.Symbol]any` |
| any other Hash | `map[any]any` |
| a go-scikit object | the Go value it wraps: `*plot.Figure`, `stats.TTestResult`, `*optimize.Result`, … |
| any other Ruby object | `ruby.Value`, which a Go cell can `Call` |

The type comes from the value, so it is known once the Ruby cell has run. A Go
cell that reads a Ruby variable whose type is not known yet waits: the Ruby
cell runs, then the Go cell is compiled with the type and runs, in the same
run. When a later edit changes the type, only the Go cells that read the
variable are compiled again.

## Limits

A notebook is a program, and it runs with your rights. A Ruby cell can reach
what a Go cell can: files through `File`, `IO` and `Dir`, other processes
through `Process`, the environment through `ENV`, and the bindings that write
files (`DataFrame#write_csv`, `Plot::Figure#save_svg`). The `rbgo_core` build
leaves out the network libraries (`socket`, `net/*`, `openssl`): requiring one
raises `LoadError`. Run only notebooks you trust.

Every cell runs in the worker, a process of its own, so the kernel survives
whatever a cell does:

| What the cell does | What happens |
| --- | --- |
| runs too long | with `--timeout 30s`, its worker is killed and the cell fails with "stopped: the cell ran longer than 30s"; the next run starts a new worker |
| holds too much memory | with `--memory 512` (MiB), the garbage collector keeps under the limit as long as it can, then the worker is stopped and the cell fails with "stopped: the cell went over its memory limit" |
| recurses without end | Ruby raises `SystemStackError`, shown with twelve frames |
| asks for an impossible allocation (`'x' * 2**62`) | Ruby raises `NoMemoryError` |
| calls `exit` or `exit!` | a `SystemExit` fails the cell; the worker lives on |
| kills its own process (`Process.kill`, not on Windows; `os.Exit` in Go) | the cell says so, and the next run restarts the worker |

Both limits apply to Go and Ruby cells alike and are off by default. The
memory limit counts the Go heap, sampled every 20 ms; memory held outside it
(a large stack, a mapped file) is bounded only by the operating system.

## In the browser

The [playground](https://github.com/go-notebook/playground) runs Ruby cells
too, with no server: the same rbgo and go-scikit bindings, compiled to
WebAssembly. The page fetches the Ruby core (`playground-ruby.wasm`) only when
the open notebook has a Ruby cell, so a visitor who runs only Go never
downloads it. While the first core downloads, the page shows what go-notebook
does, in cards that link to these pages; a core already in the browser's cache
opens the notebook without them. The Ruby there has no bridge to the page's JavaScript and no file
system, and every figure a cell shows goes through an allowlist SVG sanitizer
before it reaches the page. Try the Ruby examples on
[go-notebook's own playground](https://go-notebook.github.io/playground/#3-ruby),
[go-statistics](https://go-statistics.github.io/playground/#5-ruby) and
[go-numerics](https://go-numerics.github.io/playground/#6-ruby).
