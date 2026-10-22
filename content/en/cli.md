---
title: "The notebook command"
linkTitle: "Command"
weight: 30
description: "notebook run, html, watch and serve: run a notebook, write its page, follow its edits."
tags: [cli]
---

```sh
go install github.com/go-notebook/notebook/cmd/notebook@latest
```

```text
usage: notebook run|html|watch|serve [flags] notebook.md

  run    run the notebook and print each cell's results
  html   run it and write the results as a page next to it (or -o)
  watch  run it again every time the file changes, keeping the page current
  serve  run it again every time the file changes, showing it live in a browser (-addr)
```

Edit the `.md` in any editor. Every verb goes through the same kernel, so each
one runs only the cells whose code or inputs changed since the last run of
that notebook, whichever verb ran it (see [The kernel](kernel.md)).

## run

```sh
notebook run analysis.md
```

Runs the notebook and prints each code cell: its index, its line in the file,
and either how long it ran or `kept` when its result was kept from before;
then what it printed, what it showed and its error, indented.

```console
$ notebook run spectrum.md
── cell 0 (line 4, kept)
── cell 1 (line 12, kept)
── cell 2 (line 19, ran in 42µs)
   9.54929284535666e-07
```

The exit status is non-zero when a cell failed (`notebook: 1 cell(s) failed`),
so `notebook run` can gate a CI job.

## html

```sh
notebook html analysis.md              # writes analysis.html
notebook html -o report.html analysis.md
```

Runs the notebook and writes a standalone HTML page: the prose rendered as
Markdown, each code cell with what it printed, showed or failed with. The page
has **no scripts and no external resources**, and raw HTML in the prose is left
out (goldmark's safe default), so a page made from someone else's notebook runs
nothing of theirs but what its cells chose to display.

## watch

```sh
notebook watch analysis.md
```

Runs the notebook every time the file changes, and keeps `analysis.html` (or
`-o`) current. Each run prints one line:

```console
21:28:21  0 of 3 cells ran, build 252ms: ok
```

It looks at the file's modification time every `-interval` (300 ms by
default), and runs until interrupted.

## serve

```sh
notebook serve analysis.md
notebook serve -addr 127.0.0.1:9000 analysis.md
```

The same loop as `watch`, but the page is served at `-addr` (default
`127.0.0.1:8888`) instead of written to a file:

```console
$ notebook serve spectrum.md
serving spectrum.md at http://127.0.0.1:8888/
21:28:21  0 of 3 cells ran, build 252ms: ok
```

- The page **reloads itself** through a server-sent event when the results
  change, and keeps its scroll position.
- A notebook that cannot run at all (for example, an import that cannot be
  fetched) is shown as an **error page** instead of a stale one.

The served page is the `html` page plus the small script that reloads it.

## Flags

| Flag | Default | Verbs | Meaning |
| --- | --- | --- | --- |
| `-o` | `<notebook>.html` | `html`, `watch` | the HTML file to write |
| `-addr` | `127.0.0.1:8888` | `serve` | where `serve` listens |
| `-interval` | `300ms` | `watch`, `serve` | how often the file is looked at |
| `-work` | under the user cache directory | all | the [work directory](kernel.md#on-disk) |
| `-runtime-src` | | all | use the go-notebook source tree at this path instead of a released runtime |
| `-timeout` | `0` (no limit) | all | stop a cell, in Go or in Ruby, that runs longer than this (`30s`, `2m`): its worker is killed, the cell fails, and the next run starts a new worker |
| `-memory` | `0` (no limit) | all | stop a cell whose worker's Go heap holds more than this many MiB: the garbage collector keeps under it as long as it can, then the worker is stopped and the cell fails |

Flags go between the verb and the notebook: `notebook html -o out.html nb.md`.

`NOTEBOOK_REPLACE=module=dir,module=dir` points modules of the work module at
local checkouts, to run a notebook against a library being changed (for example
`github.com/go-scikit/ruby=../ruby` for the Ruby bindings).

## The work directory

By default each notebook has its own work directory under the user's cache
directory (`go-notebook/<hash of the notebook's absolute path>`), so a second
`notebook run` of the same file starts where the first stopped. Pass `-work`
to choose another, for example a fresh one to run everything again.

## The runtime the notebook links

The compiled notebook links package `runtime` of go-notebook. A `notebook`
installed from a release tag makes the notebook's work module require that same
release. A `notebook` built from a source checkout uses that checkout instead
(through a `replace` directive), since its pseudo-version names no published
runtime with the same code; `-runtime-src` points at a checkout explicitly.
