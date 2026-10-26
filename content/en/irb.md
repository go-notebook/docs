---
title: "An interactive session: notebook irb"
linkTitle: "irb"
weight: 27
description: "notebook irb: irb on the notebook's Ruby — rbgo with the go-scikit bindings — line by line, each input in a worker that a limit or Ctrl-C can stop."
tags: [cli, ruby]
---

`notebook irb` (go-notebook v0.5.0 and later) is an interactive Ruby session,
as Ruby's `irb`, on the Ruby that runs [Ruby cells](ruby.md):
[rbgo](https://github.com/go-embedded-ruby/ruby) with the
[go-scikit bindings](https://github.com/go-scikit/ruby) — `Stats`, `Special`,
`Optimize`, `Integrate`, `DataFrame`, `Plot` and `NDArray`. There is no
notebook file: inputs are typed, evaluated and shown one at a time.

```console
$ notebook irb
irb(main):001> x = [2.1, 3.4, 1.9, 5.6]
=> [2.1, 3.4, 1.9, 5.6]
irb(main):002> fit = Stats::Normal.fit(x)
=> #<Stats::Normal loc=3.25, scale=1.4739402972983673>
irb(main):003> fit.loc
=> 3.25
irb(main):004> _ * 2
=> 6.5
irb(main):005> def sq(v)
irb(main):006*   v * v
irb(main):007* end
=> :sq
irb(main):008> sq(3)
=> 9
irb(main):009> x = x.sum
=> 13.0
irb(main):010> raise "boom"
boom (RuntimeError)
	from (irb):10:in '<main>'
irb(main):011> s = "multi
irb(main):012" line string"
=> "multi\nline string"
irb(main):013> exit
```

(A real transcript, from `notebook irb < session.rb`: when standard input is
not a terminal, each line is echoed after its prompt.)

The first session builds the worker program, once (`building the Ruby worker`;
it fetches and compiles rbgo the first time); later sessions start at once. It
needs the Go toolchain, as `notebook run` does.

## What it does as irb does

| | |
| --- | --- |
| **Results** | `=> ` and the value's `inspect` after each input; `_` is the last value |
| **Exceptions** | `message (Class)` and the frames in the session (twelve at most); the session goes on, and `_` keeps the last value that was not one |
| **Reassignment** | any local may be assigned again: there is no dependency graph here, unlike in a notebook |
| **Multi-line input** | an input that is not finished — an open `def`, `do`, bracket, string, heredoc, or a line ending in an operator or `\` — continues on the next line, under irb's continuation prompt (`*`, or the quote of an open string) |
| **Ending** | `exit`, `quit`, `exit!` or Ctrl-D; Ruby's `exit` inside an input ends it too |
| **Ctrl-C** | abandons the line being typed; during an input, stops it (see below) |

Whether an input is finished is decided by rbgo's own parser
([go-ruby-parser](https://github.com/go-ruby-parser/parser)): an input it finds
ending in the middle of a construct continues. Where the parser accepts an
unterminated literal as it is (a string, a heredoc, a regexp, `=begin`), the
port of IRB's lexer in [go-ruby-irb](https://github.com/go-ruby-irb/irb) says
it is open. Any other syntax error is evaluated, for Ruby to report.

## The line editor

On a terminal, lines are edited with
[go-ruby-reline](https://github.com/go-ruby-reline/reline), a pure-Go port of
Ruby's Reline, through its `terminal` package — on Linux, macOS and Windows 10
and later:

- emacs keys (Ctrl-A/E/K/U/W/Y, Alt-F/B, …), the arrow, Home, End and Delete
  keys, undo;
- history with Up and Down, and Ctrl-R to search it; a multi-line input comes
  back whole;
- bracketed paste: a pasted block is inserted as it is, newlines included;
- **Tab completion** of locals, methods of `main`, keywords, constants and the
  bindings (`St` → `Stats`, `Stats::Nor` → `Stats::Normal`), and of the methods
  of a variable's value or a constant path's (`x.so` → `x.sort`,
  `Stats::Normal.fi` → `Stats::Normal.fit`), as IRB's default completor does. A
  word is completed by asking the worker about the names it holds: completion
  never evaluates an expression the word contains (`xs.first.ti` completes to
  nothing), so pressing Tab runs none of your code. A completion that takes
  more than five seconds (a `methods` method that does not return) stops the
  worker, and the next input says so.

When standard input is not a terminal, lines are read as they come, with no
editing, and echoed after their prompt, so `notebook irb < script.rb` prints a
transcript and a session can be scripted and tested.

## Limits: each input in a worker

The interpreter runs in a worker process, as a notebook's cells do, and
`--timeout` and `--memory` apply to **each input**:

```console
$ notebook irb --timeout 3s --memory 256
irb(main):001> kept = 1
=> 1
irb(main):002> loop { }
stopped: the input ran longer than 3s, the limit; its worker was killed
notebook irb: the session's state (locals, methods, classes) is lost; the next input starts a new worker
irb(main):003> kept
undefined local variable or method 'kept' for main (NameError)
	from (irb):3:in '<main>'
```

rbgo cannot be interrupted in the middle of an evaluation, so stopping an
input — a limit, or Ctrl-C — kills its worker: the prompt stays, the next
input gets a new worker, and the session says that the old one's locals,
methods and classes are gone. The worker is in a process group of its own, so
Ctrl-C in the terminal reaches the session, which stops the worker; it is not
the worker that dies of it.

The worker's standard input is empty (`gets` returns `nil`): its own is the
protocol with the session.

## Figures

A value that shows itself — a `Plot` figure — is never printed as SVG:

```console
irb(main):014> fig, ax = Plot.subplots
=> [#<Plot::Figure 640x480>, #<Plot::Axes>]
irb(main):015> ax.plot([1, 2, 3], [3, 1, 2])
=> nil
irb(main):016> fig
=> #<Plot::Figure 640x480>  (a figure: --svg-dir saves it as SVG)
```

With `--svg-dir figures`, each figure an input shows is written to
`figures/figure-N.svg`, and the line says where.

## Saving the session as a notebook

`--save session.md` writes, when the session ends, the inputs that ran as
` ```ruby ` cells, which `notebook run session.md` runs again. A notebook has
rules a session does not — each name is defined by one cell, and `_` stays in
its cell — so the inputs are grouped:

- one cell per input, by default;
- an input that defines or assigns a name an earlier cell defines (`x = x.sum`
  after `x = [...]`, a method defined again) is joined to that cell, with every
  input in between;
- an input that reads `_` is joined to the one before it, which then assigns
  `_` its value (`_ = begin … end`), as irb did;
- should the result still break a rule (two methods that call each other,
  defined in two inputs), all the inputs make one cell.

The inputs that raised an exception are left out, and the notebook says how
many. For the session above:

````markdown
```ruby
x = [2.1, 3.4, 1.9, 5.6]
fit = Stats::Normal.fit(x)
_ = begin
fit.loc
end
_ * 2
def sq(v)
  v * v
end
sq(3)
x = x.sum
```

```ruby
s = "multi
line string"
```
````

The saved notebook is a record of what ran, not of the session's state: a
worker stopped in the middle of the session does not change it.

## History

The inputs are kept in `irb_history` under the user config directory's
`go-notebook` (`~/.config/go-notebook/irb_history` on Linux,
`~/Library/Application Support/go-notebook/irb_history` on macOS,
`%AppData%\go-notebook\irb_history` on Windows), in irb's format (a multi-line
input's lines joined by `\`):

- the **inputs only, never their values**;
- the last `--history-size` of them (1000, irb's default; `0` keeps no file);
- **mode 0600** (read and written by its owner only), in a directory made with
  mode 0700; it is written through a new file renamed over the old one, and
  another session's inputs saved meanwhile are kept;
- a file that cannot be read, or one over 8 MiB, is left alone, and the session
  keeps no history.

## Flags

| Flag | Default | Meaning |
| --- | --- | --- |
| `-t`, `--timeout` | `0` (no limit) | stop an input that runs longer than this |
| `-m`, `--memory` | `0` (no limit) | stop an input whose worker holds more than this many MiB of Go heap |
| `-s`, `--save` | | write the session, at its end, as a notebook |
| `--svg-dir` | | save each figure as an SVG file in this directory |
| `--history` | see above | the history file |
| `--history-size` | `1000` | how many inputs the history keeps; `0`: no history file |
| `-w`, `--work` | `go-notebook/irb` under the user cache directory | where the worker is built |
| `--runtime-src` | | use the go-notebook source tree at this path instead of a released runtime |

`NOTEBOOK_REPLACE` points modules of the worker's module at local checkouts,
as for a notebook (see [the command](cli.md#flags)).

## Not in it

Go cells: a session is Ruby only. irb's own commands beyond `exit` and `quit`
(`ls`, `show_source`, `edit`, …), `pp` and `inspect` modes, and colour are not
there yet.
