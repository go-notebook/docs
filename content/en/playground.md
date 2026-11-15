---
title: "Using the playground"
linkTitle: "Playground"
weight: 28
description: "The browser playground: edit, add, run, switch language, delete, move and download cells, with Jupyter's keys."
tags: [playground]
---

The [playground](https://go-notebook.github.io/playground/) runs a notebook
in the browser, with no server. Go cells run in an interpreter compiled to
WebAssembly; Ruby cells run in [rbgo](https://github.com/go-embedded-ruby/ruby), R cells in
[go-embedded-r](https://github.com/go-embedded-r/r) and Fortran cells in
[go-embedded-fortran](https://github.com/go-embedded-fortran/fortran).
These are its controls from go-notebook/playground v0.6.0. The keys are
Jupyter's, with the same meanings; every key has a button too, because a
phone has no keyboard.

## The Markdown is the notebook

A notebook is a Markdown file whose ` ```go ` and ` ```ruby ` blocks are
cells. In the playground every edit goes through the same `format` package
the `notebook` command uses, and the file is written back from it.
**Download .md** gives back what you opened, byte for byte (`~~~` fences and
fence options included), plus your edits and nothing else. **Open .md**
reads a local file with the same rules.

## Editing a cell

Type in a cell to edit it. The cell says *editing* and has a solid outline.
**Esc** selects the cell instead: it says *selected* and has a dashed
outline, and the keys below act on the whole cell. **Enter** edits it again.
A Markdown cell shows as text; its **Edit** button, or Enter, opens its
source, and Esc shows it as text again.

Since playground v0.11.0 a code cell is a code editor with syntax
highlighting (go-rouge), and a Markdown cell keeps a plain text box. For a
screen reader each code cell is one textbox, named "Go cell N" (or the cell's
language). IME input works, the browser's Ctrl+F finds code, a selection that
runs from prose into code copies, and the wheel scrolls the page (a long line
scrolls sideways). A cell is as tall as its code. The page first shows a plain
text box and keeps it if the editor fails to load. The editor adds 1.8 MB
(brotli) to the page's download.

Since playground v0.13.0 the code is set in Go Mono at 13 CSS px (a host can
set `Config.FontPx`, 8 to 32). A **minimap** on the right of each cell shows
its whole source; a click on it moves the caret there. It is hidden under
480 px.

### Line annotations

A note can be attached to a line of a code cell (playground v0.13.0). Put the
caret on the line and press **Ctrl+Shift+M**: a marker appears in the gutter
and an inline panel opens under the line, where you type the note. Click the
marker, or press Ctrl+Shift+M again, to open or close the panel. The panel is
a labelled, keyboard-reachable control, so a screen reader announces it.

The notes live in the Markdown, on one line after the cell's closing fence:

````markdown
```go
x := 1
y := x * 2
```
<!-- gonb:notes {"v":1,"notes":[{"line":2,"text":"doubled"}]} -->
````

`line` counts from 1 inside the cell. The line is an HTML comment, so any
other Markdown reader ignores it, and **Download .md** and **Open .md** keep
it: the notes survive a round trip.

| | button | key on a selected cell |
|:--|:--|:--|
| run this cell (and the cells above it) | ▶ Run | Shift+Enter, Ctrl/⌘+Enter (also while editing) |
| run every cell | Run all | |
| add a cell | **+ Add cell** between the cells and at the end | A above, B below; Alt+Enter runs and adds below |
| change its language | the language menu on the cell | M Markdown, Y code |
| delete it | Delete | D D |
| undo the delete | Undo delete | Z |
| move it | ↑ ↓ | Ctrl+Shift+↑ / ↓ |

On a narrow screen a cell's buttons fold behind **⋯**.

## Choosing and changing a language

**+ Add cell** asks what the new cell is: Go, Fortran, R, Ruby, C, Julia or Markdown,
the languages go-notebook registers. A C or Julia cell is parsed and
checked, not run yet, and the menu says so (*C · not runnable yet*,
*Julia · not runnable yet*); a cell that reads it is
shown as blocked. C cells are checked in order, so a cell sees the typedefs
and macros of the C cells above it (playground v0.12.0). An R cell **runs**
(playground v0.14.0), see [R cells](#r-cells). The language
of the code cell above comes first, marked *(as above)*. A cell's language
menu changes the language later. Only the fence changes, from ` ```ruby `
to ` ```go ` for instance. **The code is not translated**, and the cell
says so. Then the cell runs again.

A fenced block in any other language is not a cell. It stays text, marked
*not run: shown as text*.

## Running

The playground keeps no values between runs, and it has no dependency
graph. So running a cell runs **every cell above it too**, in order, and none
of the cells below it; those keep their last results, greyed until they run
again. The `notebook` command does more: its kernel orders the cells by what
they use and runs only what changed ([the kernel](kernel.md)).

## R cells

Since playground v0.14.0 an R cell runs in the page, in the second core, on
[go-embedded-r](https://github.com/go-embedded-r/r)'s evaluator, with no
external R. R's variables are isolated: they persist from one R cell to the
next, and do not cross to the Go or Ruby cells. A cell is bounded: **512 MiB**
of vectors, **20 million steps**, **10 s** and a nesting depth of **300**. A
cell that goes past one is stopped with a message that says which. If the core
crashes it is restarted, and the cell shows a message. The second core is
about 4.5 MB larger.

## Fortran cells

Since playground v0.16.0 a Fortran cell runs in the page, in the second core,
on [go-embedded-fortran](https://github.com/go-embedded-fortran/fortran)'s
evaluator, with no compiler. Fortran's variables, procedures and modules
persist from one Fortran cell to the next, and do not cross to the other
languages. A cell is bounded: **200 million** DO iterations and calls, **10 s**
and 16 MiB of output per stream; an endless `do` ends with a message that says
so and the page stays usable. The second core is about 1.3 MB larger. See
[Fortran cells](fortran.md).

## Limits

A notebook you open may come from anyone. The playground gives it:

- **30 s per cell.** A cell that runs longer is stopped as the kernel's
  `--timeout` stops it: `stopped: the cell ran longer than 30s, the limit;
  its worker was killed`. The next run starts a fresh runtime.
- **1 MiB per notebook.** A larger file is refused when opened.
- **No memory limit**, unlike the kernel's `--memory`. The page cannot bound
  the heap of the WebAssembly worker while a cell runs; the browser's own
  limit is the only one.
- No network, no file system and no access to the page, in Go or in Ruby.
  Every figure goes through two SVG sanitizers before it is shown.

## Where the controls come from

The gestures follow Jupyter Notebook 7 and JupyterLab, read in their
sources:

- jupyterlab/jupyterlab:
  - `docs/source/user/notebook.md`: edit and command modes;
  - `packages/notebook-extension/schema/tracker.json`: the keys;
  - `packages/notebook/src/notebookfooter.ts`: the add button after the last cell;
  - `packages/notebook/src/default-toolbar.tsx`: the cell type menu;
  - `packages/cell-toolbar-extension/schema/plugin.json`: the cell toolbar.
- The language per cell follows Polyglot Notebooks (the dotnet/interactive
  README), where each cell names its own language.
