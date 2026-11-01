---
title: "Adding a cell language"
weight: 45
description: "A cell language is one registered analysis.Language: how to implement it, mark dynamic cells, say whether its cells run, and plug it in without touching the kernel, the generator or the graph."
tags: [cells, kernel]
---

A cell language is **one registered implementation** of
[`analysis.Language`](https://github.com/go-notebook/notebook/blob/main/analysis/lang.go),
keyed by the first word of the fence's info string. The graph, the kernel, the
generator and the HTML export look a cell's language up with
`analysis.Lookup` and never test for a language by name. Go, Ruby and Julia are
registered; a fence whose word is not registered stays prose.

```go
type Language interface {
	Name() string // the fence word: "go", "ruby", "julia", ...
	Analyze(src string, lim Limits) (Names, []Diagnostic, error)
	Crosses(owner, user *Cell, name string) bool
	Run() Run
}

type Names struct {
	Defines, Uses []string
	Dynamic       bool
}
```

## The steps

1. **Implement `Language` and call `analysis.Register` from `init`.**
   `Analyze` returns the names a cell defines and uses and its syntax errors
   (a `Diagnostic` carries a byte offset in the cell). It refuses a source
   over `lim.MaxBytes`, or nested deeper than `lim.MaxDepth`, with an error and
   never a panic.
2. **Say what crosses to other languages.** `Crosses(owner, user, name)` is the
   owner's rule for whether a cell of another language sees `name`. Within one
   language every name is shared. A Go cell sees a Ruby top-level local, not a
   Ruby method or constant; a language that cannot hold every name (Go
   keywords) implements `NameChecker`.
3. **Mark dynamic cells.** Set `Names.Dynamic` when a cell's names cannot be
   resolved statically: R's `<<-`, `assign` and `get` with computed names,
   `eval(parse())`, `attach`, `library()`. The graph makes a dynamic cell
   depend on **every earlier cell**, and every later cell that reads any name
   depend on **it**, so it runs after anything before it changed and
   invalidates whatever may read what it defined.
4. **Say whether the cells run.** `Run{Runs: true}` runs them. Otherwise the
   language is **check-only**: `NotRun` is the cell's failure (Julia's names
   its milestone M3), and `Blocked`, `Reads` and `ReadsBlocked` are the
   messages of the cells that read it, directly or through other cells.
5. **Optional capabilities**, found by type assertion:

   | Interface | Gives |
   |---|---|
   | `CellAnalyzer` | the full `Cell` (chunks, funcs, types, display) instead of `Names` |
   | `Highlighter` | the go-rouge lexer for the HTML export; without it the source is escaped |
   | `Runtime` | the Go module the worker links (import path, module, default version); the kernel adds it to the work module |
   | `TypeLearner` | variables typed when the cell has run, as Ruby's |

6. **Write the run backend** for a language that runs: how its cells are
   written into the generated program. It is a `gen` backend
   (`gen/backend.go`; Ruby's is in `gen/ruby.go`), the one piece that lives
   outside `analysis`.

## Testing

Test with a fake language that implements only `Language`, as
`analysis/lang_test.go` does, and fuzz the dispatch with unknown or hostile
fence words (`FuzzDispatch`). The project keeps 100 % coverage.
