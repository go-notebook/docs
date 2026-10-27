---
title: "Using the playground"
linkTitle: "Playground"
weight: 28
description: "The browser playground: edit, add, run, switch language, delete, move and download cells, with Jupyter's keys."
tags: [playground]
---

The [playground](https://go-notebook.github.io/playground/) runs a notebook
in the browser, with no server. Go cells run in an interpreter compiled to
WebAssembly; Ruby cells run in [rbgo](https://github.com/go-embedded-ruby/ruby).
These are its controls from go-notebook/playground v0.5.0. The keys are
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

**+ Add cell** asks what the new cell is: Go, Ruby, Julia or Markdown. A
Julia cell is parsed and checked, not run yet, and the menu says so. The language
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
