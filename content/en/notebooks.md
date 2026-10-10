---
title: "Notebooks"
weight: 10
description: "A notebook is a Markdown document; its go blocks are the cells."
tags: [format]
---

A notebook is a Markdown file. Its fenced code blocks whose info string's first
word is `go` are **code cells**; everything between them is **prose**, kept byte
for byte.

````markdown
# Spectrum

Some prose, with a list, a table, a link: any Markdown.

```go
x := 40 + 2
```

```python
# A block in another language is prose, not a cell.
```
````

## Fences

Fences follow [CommonMark](https://spec.commonmark.org/): three or more
backticks or tildes, indented by at most three spaces, closed by a fence of the
same character at least as long. A code block in another language is prose.

## Nothing is written back

No output is ever stored in the file. Results live in the kernel's value store
(see [The kernel](kernel.md)) and in exports such as the HTML page written by
[`notebook html`](cli.md#html).

Because of that:

- GitHub, and every Markdown viewer, shows a notebook as a document;
- a diff of a notebook is a diff of text, with no execution counts or outputs;
- reading and writing a notebook round-trips exactly:
  `format.Parse(src).String() == src` for every input, and that property is
  fuzzed.

## Why not a valid Go file

A `.go` file was considered first and rejected. Go allows no statements at the
top level, and `x := load(path)` or `fmt.Println(x)` at the top level of a cell
is most of what a notebook is. Wrapping every cell in a function costs the user
more than editor support on the raw file gains. The kernel produces the valid
Go program instead: see [The kernel](kernel.md#the-generated-program).

## The packages

The Markdown is read and written by package
[`format`](https://github.com/go-notebook/notebook/tree/main/format). Its
`Parse` returns a `Notebook` whose `Cells` are either code or prose, each with
its line in the file.
