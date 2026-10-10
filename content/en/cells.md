---
title: "Cells"
weight: 20
description: "A cell is Go with statements allowed at the top level; three rules make the order a function of the code."
tags: [cells]
---

A cell is Go with statements allowed at the top level, as in a REPL:

```go
import "strings"

func words(s string) []string { return strings.Fields(s) }

w := words("to be or not to be")
len(w) // the final expression is shown
```

```console
$ notebook run words.md
── cell 0 (line 4, ran in 86µs)
   6
```

## Declarations and statements

- Top-level `func`, `type` and `import` are **declarations of the notebook**.
- Everything else is a **statement of the cell**, run in order.
- The names a cell defines at its top level — by `:=`, `var`, `const`, `func`
  or `type` — are **notebook variables**: every cell, and every function, can
  read them.

## The final expression is shown

A cell's final expression statement is its **display value**. A call with
several results (`fmt.Println`'s `n, err`) is run for its effect and not shown.

A value with a `Display() map[string][]byte` method (the
[`runtime.Displayer`](https://github.com/go-notebook/notebook/blob/main/runtime/runtime.go)
interface) shows itself: the map goes from a MIME type — `text/plain`,
`text/html`, `image/svg+xml`, `image/png`, … — to the bytes of that
representation. Values without one show their `%v`.

What a cell prints, through `os.Stdout`, the `println` builtin or C code, is
captured and shown with it. (On systems without `dup2` — Windows, js, wasip1 —
`os.Stdout` and `os.Stderr` are swapped instead, and `println` is not captured.)

## The three rules

They make the order of execution a function of the code, as in marimo, rather
than of the order cells were run in.

### One definition per name

Each notebook variable is defined by exactly one cell. Go already forbids two
package-level declarations of a name, which gives this rule for free.

### No assignment to another cell's variable

A cell may not assign to a variable another cell defines: the dependency graph
would not see the change.

````markdown
```go
x := 1
```

```go
x = 2
```
````

```console
$ notebook run rules.md
…
   error: 8:1: cell 1: cannot assign to x, which cell 0 defines; define a new name instead
notebook: 1 cell(s) failed
```

Mutation through a method or a slice element cannot be seen statically and is
not refused (marimo makes the same choice).

### Private names stay in their cell

A name starting with `_`, and `err` — which nearly every cell declares and none
means to share — are local to their cell. Two cells may each have their own
`err`.

## Errors point at the notebook

The generated program carries `//line` directives, so compiler errors and panic
stacks name the notebook's own lines, not the generated file's.

A cell that fails stops the cells that read it, and they say why:

````markdown
```go
var xs []int
first := xs[0]
```

```go
first + 1
```
````

```console
$ notebook run panic.md
── cell 0 (line 4, ran in 104µs)
   error: panic: runtime error: index out of range [0] with length 0
   …
   main._nbcell0(…)
   	…/panic.md:5 +0x3c
   …
── cell 1 (line 9, kept)
   error: not run: cell 0 failed
notebook: 2 cell(s) failed
```

(The stack is shortened here; the frame of the cell names `panic.md:5`, the
line of `xs[0]`.)

## Variables that cannot be shared

A notebook variable becomes a package-level variable of the generated program,
which needs its type to be written there. A variable whose type cannot be
written outside its package (an unexported type) cannot be shared: the error
says so and suggests a private name (`_x`).
