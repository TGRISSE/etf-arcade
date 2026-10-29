# etf-arcade

A build plan for a coding agent. This repository does not contain the finished application: it holds the specification, the example inputs and the visual reference from which an agent builds a neon-arcade ETF screener that runs from static files. To use it, give your agent `AGENTS.md`; it reads the docs, builds one slice at a time in its own copy of the repository, and points every check at the project folder you name.

## What gets built

- A small Python package, `etf_arcade`, that turns a project folder (`universe.csv`, `var/prices.csv`, `settings.toml`, optional `theme.css` and `review/*.json`) into one generated file, `web/data.js`.
- A static dashboard in `web/` with fifteen tabs (Overview, Heatmap, Leadership, Chart wall, Groups, Rotation, Correlation, Structure & PCA, Risk & return, Signal lab, Screener, Calendar, Market, Fund, Data) that reads `web/data.js` and works when opened straight from disk, with no server.
- A dark neon look: magenta and cyan accents, glow only on titles and focused controls, no animation or screen effects.
- Settings shared by every tab and kept in the browser: group filter, return period, chart history, short or full names, and a benchmark switch; plus an optional review mode that exports corrections as a file for the next build.

It is built in twelve slices plus slice 1b (projects) and eight parity slices, 13a to 13h, that complete every tab. The order is 1, 1b, 2, 3, 11, 12, 4, 5, 6, 7, 8, 9, 10, 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h; each slice ends in something you can open in a browser or check with a command. `STATUS.md` says where the build stands and which slice is next.

The command line the agent builds is `python -m etf_arcade [--project DIR] {sample|fetch|build|review-status}`. It exists after slice 1 (`--project` after slice 1b, `review-status` after slice 12); `sample` writes clearly labelled SYNTHETIC prices, `fetch` downloads adjusted closes with yfinance, `build` writes `web/data.js`. Every command exits 0 on success and 2 on failure, and a failed build leaves the previous `web/data.js` in place.

## Two example projects

| Project | Where | Contents |
|---|---|---|
| Default (US) | repository root | `data/universe.csv` (520 US-listed ETFs in 59 groups), `settings.toml` (benchmarks SPY, AGG and a 60/40 blend; a T-bill proxy as cash) |
| Europe | `examples/europe/` | `universe.csv` (141 UCITS ETFs in 31 groups), `settings.toml` (three benchmarks, no cash series), `README.md` |

`data/universe_ucits.csv` is the source list of the European mirror. Your own project is a folder of the same shape, selected with `--project DIR`; it may live outside this repository.

## Repository map

| Path | What it is |
|---|---|
| `AGENTS.md` | Entry point for the building agent: read order, hard rules, working loop |
| `STATUS.md` | Which slices are built, build order, next step |
| `docs/` | Product description, slice plan, theme, reference sheets, `skeleton.html` visual reference |
| `data/universe.csv`, `data/universe_ucits.csv` | The default universe and the European mirror list (committed facts) |
| `settings.toml` | The default project's title, benchmarks and cash series |
| `examples/europe/` | The Europe example project |
| `web/theme.css` | Every colour, font and spacing token (single source), shipped with the plan |
| `LICENSE`, `NOTICE` | Apache-2.0 and third-party notices |
| *built by the agent* | `pyproject.toml`, `etf_arcade/`, `tests/`, `web/index.html`, `web/app.js`, `web/tabs/`, and the generated, git-ignored `var/prices.csv`, `web/data.js`, `web/project.css`, `web/series/` |

## Docs

| File | Read it for |
|---|---|
| `AGENTS.md` | Entry point for coding agents: read order and hard rules |
| `docs/PRODUCT.md` | What the finished site looks like, tab by tab |
| `docs/BUILD_SLICES.md` | The build order and the twelve slices plus 1b and 13a to 13h, with steps and checklists |
| `docs/THEME.md` | Colour tokens, fonts, glow and layout rules |
| `docs/reference/DATA.md` | Files, settings, `data.js` schema, build and publish safety, package and CLI spec |
| `docs/reference/METRICS.md` | Every formula and its null rules |
| `docs/reference/UI.md` | Browser-side contract: state, links, tables, charts, `app.js` and `index.html` spec |
| `docs/reference/PARITY.md` | The item-by-item spec of slice 13 (13a to 13h) and the coverage table of every feature |
| `docs/PITFALLS.md` | Twenty known mistakes to avoid, one line each |
| `docs/skeleton.html` | A static page showing the panel and table patterns (open it from disk) |

## Data and disclaimer

This is a research and education tool, not investment advice. Prices fetched with yfinance come from Yahoo Finance and are subject to Yahoo's terms of use; check them before you redistribute any data. Adjusted closes depend on how the provider adjusts for distributions. Missing values are shown as an em dash and are never filled in. Sample data is generated, not market data, and is named as such in the header. The universe lists are facts (names and identifiers) and are not recommendations; re-check a row before relying on it.

## Licence

Apache-2.0. See `LICENSE` and `NOTICE`.
