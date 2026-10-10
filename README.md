# docs

[![docs](https://github.com/go-notebook/docs/actions/workflows/docs.yml/badge.svg)](https://github.com/go-notebook/docs/actions/workflows/docs.yml)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-047857?style=flat-square)](LICENSE)

**The go-notebook documentation site**, published at
<https://go-notebook.github.io/docs/>.

A [Hugo](https://gohugo.io/) site with the [tannevaled/hextra](https://github.com/tannevaled/hextra)
fork of the [Hextra](https://github.com/imfing/hextra) theme, built the same way
as the [go-ndarray documentation](https://github.com/go-ndarray/docs), with
go-notebook's branding. It documents
[`go-notebook/notebook`](https://github.com/go-notebook/notebook).

## Versions

A version of the documentation is **the minor version of notebook it
describes**, or `main` while no release is tagged:

| Source | Published under |
| --- | --- |
| branch `main`, with `params.notebook.version` in `hugo.yaml` | `https://go-notebook.github.io/docs/<MAJOR.MINOR>/`, or `/docs/main/` while it says `main` |
| the newest release (else `main`) | also `https://go-notebook.github.io/docs/latest/` |

`https://go-notebook.github.io/docs/` redirects to `latest/`. Each push to
`main` replaces the directory of the version it describes, and nothing else.

To document the first release, set `params.notebook.version` to its tag
(`v0.1.0`) with the pages: `0.1` is published beside `main`, `latest` moves to
it, and `main` stays as it was last published.

## Mirrored page

`content/en/design.md` is a copy of
[`docs/design.md`](https://github.com/go-notebook/notebook/blob/main/docs/design.md)
in go-notebook/notebook, from its first `##` heading to the end, kept verbatim
so that re-syncing it is a copy, not an edit. To re-sync from a checkout
(`$NB`):

```sh
{ sed '/^## What exists/,$d' content/en/design.md
  sed -n '/^## What exists/,$p' "$NB/docs/design.md"; } > design.tmp
mv design.tmp content/en/design.md
```

then update the commit in the `> Mirrored from` line.

The examples' outputs on the other pages are real transcripts of `notebook`;
re-run them rather than editing a value.

## Layout

| Path | What |
| --- | --- |
| `content/en/` | the pages; `_index.md` is the home, `weight` orders the sidebar |
| `i18n/en.yaml` | the site's strings, beside the theme's |
| `hugo.yaml` | `baseURL`, the notebook version described, the navbar (version › theme › search › GitHub), the brand mounts |
| `layouts/_partials/custom/version-select.html` | the version selector (reads `versions.json`), from go-ndarray/docs |
| `layouts/_partials/custom/content-begin.html` | the outdated-version banner (reads `versions.json`) |
| `layouts/_partials/favicons.html` | the brand's favicons |
| `assets/css/custom.css` | the brand emerald as Hextra's primary colour, the table headers |
| `themes/hextra` | submodule: [tannevaled/hextra](https://github.com/tannevaled/hextra), pinned to a release tag |
| `branding` | submodule: [go-notebook/brand](https://github.com/go-notebook/brand) (the mark, its PNGs and ICO) |
| `scripts/publish-version.sh` | puts one build into a `gh-pages` checkout and rewrites `versions.json`, `latest` and the root redirect |

## Working locally

```sh
git clone --recurse-submodules https://github.com/go-notebook/docs.git
cd docs
mkdir -p data && sh themes/hextra/scripts/page-history.sh > data/pagehistory.json
hugo server --baseURL http://localhost:1313/docs/main/
```

then open <http://localhost:1313/docs/main/>. Hugo 0.146 or later; no Python,
no Node.

## Publication

`.github/workflows/docs.yml`:

| Job | When | What |
| --- | --- | --- |
| `build` | pull requests, `main` | builds with `--baseURL …/docs/<version>/`, checks internal links and anchors (lychee, offline), uploads the site |
| `links` | pull requests, `main` | fetches every external URL the pages name |
| `deploy` | `main` | `scripts/publish-version.sh` into `gh-pages` (started empty on the first deploy), then a commit and a push |

GitHub Pages serves the `gh-pages` branch.

## Licence

BSD-3-Clause.
