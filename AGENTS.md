# Agent guide

etf-arcade is a build plan, and you are the builder. This repository ships no pre-built slices: you build every slice (1, 1b, 2 to 12, then 13a to 13h) in your own copy of it, following these docs. The result is an ETF-only neon-arcade screener: a Python package turns a project folder (`universe.csv`, `var/prices.csv`, `settings.toml`) into `web/data.js`, and a static dashboard in `web/` reads it from `file://`.

First task: slice 1, "Skeleton, data path and Screener tab" in `docs/BUILD_SLICES.md`.

## Read order

1. `README.md`: what is in the repository and what you build.
2. `STATUS.md`: which slices are built and the build order.
3. `docs/PITFALLS.md`: twenty known mistakes, one line each.
4. `docs/PRODUCT.md`: your tab's section, plus sections 3 and 5 (a slice with no tab reads only sections 3 and 5).
5. `docs/BUILD_SLICES.md`: your slice.
6. Only the sections of `docs/reference/DATA.md`, `METRICS.md`, `UI.md`, `PARITY.md` (slices 13a to 13h) and `docs/THEME.md` that the slice names. Slices 1 and 1b are specified down to file, function, constant and field names in `DATA.md` sections 2 to 15 and `UI.md` sections 15 and 16.

Field names, tab ids, metric ids, reason codes and storage keys must match those docs exactly. If a doc is ambiguous or wrong, fix the doc first, then the code, and note it in `STATUS.md`.

## Hard rules

- Neutral and standalone: no personal or company names, absolute development paths or outside project dependencies in shipped files (third-party licence files and `NOTICE` keep their copyright lines).
- Data safety: missing is `null`, shown as an em dash with the reason on hover. Never forward-fill into a return, never splice two tickers, never assume a zero cash rate, never invent market data. Synthetic data carries `source=synthetic` in `prices.csv` and `synthetic: true` in `data.js`; the header names the source. No warning banners.
- A failed build leaves the previous `web/data.js` untouched and exits 2 (every command exits 0 ok, 2 failure). `metrics.py` and `build.py` never import `fetch.py` or `sample.py`, and `metrics.py` never imports `quality.py` or `review.py`; `prices.csv` is the only price input. Every tunable constant lives in `etf_arcade/constants.py`.
- One CLI: `python -m etf_arcade [--project DIR] {sample|fetch|build|review-status}`. The project folder holds `universe.csv`, `var/prices.csv`, `settings.toml`, optional `theme.css` and optional `review/*.json`; it may live outside the repository. Point every check at the project folder your task names.
- Benchmarks are the ordered `[[benchmarks]]` list of `settings.toml` (the first is the default); every reference-dependent field is stored as an object keyed by benchmark id, and the settings strip has a benchmark switch that never resets the fund, filters, sort or custom metrics. Cash is `[cash]` (`ticker`, `label`, `kind`); its label comes from settings, never from code, and without `[cash]` cash-dependent metrics are null (`cash_missing`).
- UI text uses `project.universe_label` wherever it names the funds in the plural; short fund names are the default, full names and tickers live in hover text and details.
- Theme: colours, fonts and glow exist only as tokens in `web/theme.css`. No colour literals in `.js` files. Glow only on titles, the active tab, focused or hovered controls and panel hover borders. No scanlines, curvature, flicker or animation beyond 150 ms hovers. Body text never glows.
- Layout: one or at most two analytical panels per row, stacked below 900 px, no horizontal page scroll, labels fit their panel, short names by default with full name and ticker on hover.
- Shared settings (groups, return period, chart history, names, benchmark) persist and apply to every tab. Under a group filter, full-universe results (PCA, market breadth, implied correlation) say "unavailable".
- Classic scripts only; the page must work from `file://`.
- Tests: slice 1 has exactly 6 tests, slice 1b adds 2 and every later slice adds at most 3 (stdlib `unittest`, synthetic data only). When a number is computed both in Python and in the browser, one of them is a parity test: the browser function run on the same input (the published arrays) must give the published value.
- Private catalogs, price files, generated data and release folders are never committed (`.gitignore`); examples that ship are labelled synthetic or are facts-only lists.

## Working loop

1. Pick the first unbuilt slice in `STATUS.md` (build order: 1, 1b, 2, 3, 11, 12, 4, 5, 6, 7, 8, 9, 10, 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h).
2. Read it and its named docs; look at `docs/skeleton.html` for the panel patterns.
3. Build the code and the tab module the slice lists; add any new `data.js` field to `DATA.md` first. "Files to create or change" names the main files: also touch `web/index.html` (script tags), `web/app.css` (new classes, existing tokens only; never edit `web/theme.css`, which the plan ships and may replace), the reference docs and `STATUS.md` as the slice needs. Where a doc leaves a detail open, decide, write the decision into the reference doc as "as built", and go on.
4. Run `python -m unittest -q`, `python -m etf_arcade sample`, `python -m etf_arcade build` (from slice 1b on, with `--project DIR` when the task names a project; slice 1 runs on the repository-root project), then check the page in a browser.
5. Tick the slice checklist in `docs/BUILD_SLICES.md`, update `STATUS.md`, stop. One slice per change.

`web/data.js` holds one project at a time, so rebuild the project you are asked to leave on screen when you are done.

Other agents may share the tree: edit only the files your slice lists and never revert changes you did not make.

## Keep STATUS.md current

After every slice update `STATUS.md`: slice state, commands run and results, open issues, next step (rewrite the Next step line every time; a stale one misleads the next session). Keep it short.
