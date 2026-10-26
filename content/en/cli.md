---
title: "The notebook command"
linkTitle: "Command"
weight: 30
description: "notebook run, html, watch and serve: run a notebook, write its page, follow its edits; notebook irb: an interactive Ruby session."
tags: [cli]
---

```sh
go install github.com/go-notebook/notebook/cmd/notebook@latest
```

```text
Usage:
  notebook [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  html        Run the notebook and write its results as a page next to it (or --output)
  irb         Start an interactive Ruby session, as irb, on the notebook's Ruby and bindings
  run         Run the notebook and print each cell's results
  serve       Run the notebook again every time it changes, showing it live in a browser
  watch       Run the notebook again every time it changes, keeping its page current
```

`notebook --help` lists the verbs, `notebook <verb> --help` the flags of one,
and `notebook --version` the release.

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
`-o`/`--output`) current. Each run prints one line:

```console
21:28:21  0 of 3 cells ran, build 252ms: ok
```

It looks at the file's modification time every `--interval` (300 ms by
default), and runs until interrupted.

## serve

```sh
notebook serve analysis.md
notebook serve --addr 127.0.0.1:9000 analysis.md
```

The same loop as `watch`, but the page is served at `--addr` (default
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

## irb

```sh
notebook irb
notebook irb --timeout 30s --memory 2048 --save session.md --svg-dir figures
notebook irb < script.rb
```

An interactive Ruby session, as `irb`, on the Ruby of [Ruby cells](ruby.md)
(rbgo with the go-scikit bindings), with no notebook file (v0.5.0 and later):
`=> inspect` after each input, `_` the last value, exceptions shown without
ending the session, reassignment allowed, multi-line input, a line editor with
history and Tab completion. Each input runs in a worker process, to which
`--timeout` and `--memory` apply; `--save` writes the session as a notebook
that `notebook run` runs. [The irb page](irb.md) has the details.

## Flags

| Flag | Default | Verbs | Meaning |
| --- | --- | --- | --- |
| `-o`, `--output` | `<notebook>.html` | `html`, `watch` | the HTML file to write |
| `--addr` | `127.0.0.1:8888` | `serve` | where `serve` listens |
| `--interval` | `300ms` | `watch`, `serve` | how often the file is looked at (must be positive) |
| `-w`, `--work` | under the user cache directory | all | the [work directory](kernel.md#on-disk) |
| `--runtime-src` | | all | use the go-notebook source tree at this path instead of a released runtime |
| `-t`, `--timeout` | `0` (no limit) | all | stop a cell, in Go or in Ruby, that runs longer than this (`30s`, `2m`): its worker is killed, the cell fails, and the next run starts a new worker; for `irb`, an input |
| `-m`, `--memory` | `0` (no limit) | all | stop a cell whose worker's Go heap holds more than this many MiB: the garbage collector keeps under it as long as it can, then the worker is stopped and the cell fails; for `irb`, an input |
| `-s`, `--save` | | `irb` | write the session, at its end, as a notebook of Ruby cells |
| `--svg-dir` | | `irb` | save each figure an input shows as an SVG file in this directory |
| `--history` | `go-notebook/irb_history` under the user config directory | `irb` | the history file (mode 0600, the inputs only) |
| `--history-size` | `1000` | `irb` | how many inputs the history keeps; `0`: no history file |

Flags are POSIX (since v0.4.0): a long name takes two dashes, a one-letter
shorthand one (`notebook html -o out.html nb.md`,
`notebook run --timeout 30s nb.md`), before or after the notebook. A verb
refuses a flag it does not read. The single-dash spelling of v0.3 and earlier
(`-timeout 30s`) is still accepted, with a note on stderr.

`NOTEBOOK_REPLACE=module=dir,module=dir` points modules of the work module at
local checkouts, to run a notebook against a library being changed (for example
`github.com/go-scikit/ruby=../ruby` for the Ruby bindings).

## The work directory

By default each notebook has its own work directory under the user's cache
directory (`go-notebook/<hash of the notebook's absolute path>`), so a second
`notebook run` of the same file starts where the first stopped. Pass `--work`
to choose another, for example a fresh one to run everything again.

## The runtime the notebook links

The compiled notebook links package `runtime` of go-notebook. A `notebook`
installed from a release tag makes the notebook's work module require that same
release. A `notebook` built from a source checkout uses that checkout instead
(through a `replace` directive), since its pseudo-version names no published
runtime with the same code; `--runtime-src` points at a checkout explicitly.
