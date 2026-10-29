# Build slices

The build plan for etf-arcade. Twelve slices plus slice 1b and the parity slices 13a to 13h, each ending in something you can open in a browser or check with a command. This repository ships no pre-built code: you, the coding agent, build every slice in your own copy of the repository, following these docs, and point every check at the project folder your task names (`--project DIR`, from slice 1b; before that the repository root).

## Build order

1, 1b, 2, 3, 11, 12, 4, 5, 6, 7, 8, 9, 10, 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h. Slice numbers are names, not the order.

Slices 11 (data quality and the Data tab) and 12 (review marks) depend only on slices 1 to 3 (12 also uses 11's Data tab and flags) and need nothing from slices 4 to 10. Whatever they would have borrowed from a later slice is defined in the slice that first needs it: the cutoff constant `CUTOFF_LAG_BDAYS` and the cutoff date are defined by slice 11 and only consumed by slices 7 and 9. In return, every aggregate built in slices 4 to 9 must use the cleaned prices and skip excluded funds (DATA.md section 8). Never start a slice whose "Depends on" is not built.

## Names that must not drift

Field names, tab ids and metric ids already exist in the docs. Copy them; never invent a synonym.

| Thing | Defined in | Examples |
|---|---|---|
| `data.js` fields | `docs/reference/DATA.md` section 9 | `r_1m r_3m r_6m r_ytd r_1y r_3y_ann vol_1y maxdd_1y first_date last_date days_stale status why` |
| Metric ids for later slices | `docs/reference/METRICS.md` | `sharpe_3y beta_3y rel_3m z_13w rrg_ratio` (the id is the `data.js` key, unchanged; reference-dependent ones hold an object keyed by benchmark id) |
| Tab ids, labels, slice numbers | `docs/reference/UI.md` section 3 | `overview heatmap leaders wall groups rotation correlation pca risk lab screener calendar market fund data` |
| State keys, hash keys, storage key | `docs/reference/UI.md` section 4 | `period history names benchmark hidden review tab fund series q sort x y m`, `arcade.settings.v1`, `arcade.review.v1:<dir_name>` |
| Settings keys | `docs/reference/DATA.md` section 2b | `project constants benchmarks cash factors`; `title subtitle currency universe_label`; `id label components`; `ticker label kind`; `equity bond` |
| Reason codes | `docs/reference/DATA.md` section 9 | `no_data no_history_to_anchor cash_missing cash_incomplete reference_missing review_broken_price factors_missing` |
| CSS tokens | `web/theme.css` and `docs/THEME.md` | `--accent`, `--g1` to `--g12` |
| Constants | `etf_arcade/constants.py`, listed in `docs/reference/DATA.md` section 12 | `STALE_DAYS ANCHOR_TOL_DAYS DD_MIN_CLOSES MIN_PRICED_FRACTION LINE_GAP_DAYS CUTOFF_LAG_BDAYS` |
| CLI | `docs/reference/DATA.md` sections 2b and 15 | `python -m etf_arcade [--project DIR] {sample\|fetch\|build\|review-status}` |

## How to work a slice

1. Read the slice's section below, then only the docs it names in "Read first". Do not read other tabs' sections.
2. Read `docs/THEME.md` once if you touch CSS or charts. Colours, fonts and spacing come from `web/theme.css` tokens only; no literals in `.js` files or inline styles.
3. Look at `docs/skeleton.html` (opens standalone from disk) for the panel pattern: title, kicker, legend, chart area, footnote; one-panel and two-panel rows; table panel.
4. Add the tab module `web/tabs/<tab id>.js`, register it as `docs/reference/UI.md` section 3 says, and add its `<script>` tag to `web/index.html` (order: `app.js`, `vendor/d3.min.js`, `tabs/*.js`, `review.js`, `data.js`).
5. Extend `etf_arcade/metrics.py` and `etf_arcade/build.py` for the new `data.js` fields. First add the new fields to `docs/reference/DATA.md` section 9 (this plan names them; the build adds any detail it needs), then code them. Missing is `null`, never a fill.
6. Tests: slice 1 has exactly 6 tests, slice 1b adds 2 and every later slice adds at most 3, in `tests/test_sliceN.py` (stdlib `unittest`, synthetic data only). Test the rule that could silently break, not the plumbing.
7. Run the standard verify block below, plus the slice's own commands.
8. Tick the slice's checklist in this file, update `STATUS.md`, stop. Do not start the next slice in the same change.

Standard verify block (run from the repository root; `sample` refuses to overwrite a `yfinance` file, so add `--force` when you mean to; from slice 1b add `--project DIR` before the command name when your task names a project folder):

```
python -m unittest -q
python -m etf_arcade sample
python -m etf_arcade build
```

To look at the result, open `web/index.html` in any browser. `sample` and `build` only rewrite gitignored files (`<project>/var/prices.csv`, `web/data.js`, `web/project.css`). Headless check without a server: `<browser>` is a Chrome or Edge executable (on Windows `chrome.exe` or `msedge.exe` from its install folder, on macOS the binary inside the app bundle, on Linux `google-chrome`); replace `<repo>` with the checkout path:

```
<browser> --headless --disable-gpu --virtual-time-budget=4000 --dump-dom "file:///<repo>/web/index.html#tab=screener&check=1" > out.html
<browser> --headless --disable-gpu --window-size=1400,900 --screenshot=<abs path>/shot.png "file:///<repo>/web/index.html#tab=screener"
```

Look at `out.html` for the expected rows and text and for `overlaps 0, clipped 0` in `#check` (UI.md section 8; from slice 3 on, for every tab the slice touches), and at `shot.png` for layout. Use absolute paths for `--screenshot` (a relative path writes nothing on Windows). Headless browsers keep a minimum layout width, so `--window-size=390,844` crops instead of reflowing: check 390 px by opening a scratch page that holds `<iframe width="390" height="844" src="file:///<repo>/web/index.html#tab=...">` (or the browser's device toolbar) and confirm `document.documentElement.scrollWidth <= clientWidth` inside it. To check a view under a group filter (a setting kept in `localStorage`, not in the hash), open a scratch page that writes the settings key (UI.md section 4) and then sets `location.href` to the dashboard; use a fresh `--user-data-dir` per run so runs do not share storage. Keep `out.html`, screenshots and scratch pages outside the repository, or delete them; never commit them.

Rules for every slice: the dashboard opens from `file://` (classic scripts, no `fetch()`); a failed build keeps the previous `web/data.js` and exits 2; no warning banners (the header names the data source); no animation beyond 150 ms hover transitions; at most two analytical panels per row.

---

## Slice 1: Skeleton, data path and Screener tab

**Goal.** `sample` or `fetch`, then `build`, then open `web/index.html` and see a sortable table of the whole universe in the neon theme.

**Depends on.** Nothing. This repository already holds the inputs it needs: `data/universe.csv`, `web/theme.css`, `docs/skeleton.html`.

**Read first.** `docs/reference/DATA.md` sections 1 to 10, 12, 13 and 15; `docs/reference/METRICS.md` sections 1 and 3; `docs/reference/UI.md` sections 1 to 3, 5, 7, 10, 11, 15 and 16; `docs/PRODUCT.md` section 4.11.

**Files to create.** `pyproject.toml`; `etf_arcade/` with `__init__.py`, `__main__.py`, `constants.py`, `universe.py`, `prices.py`, `sample.py`, `fetch.py`, `metrics.py`, `build.py`, `cli.py`; `web/index.html`, `web/app.js`, `web/tabs/screener.js`; `tests/__init__.py`, `tests/test_slice1.py`. Generated and ignored by git: `var/prices.csv`, `web/data.js`.

**Steps.** Each step is specified in the section named; build exactly that.
1. `pyproject.toml`, the package skeleton and `constants.py` (DATA 12 and 15: the constants marked slice 1, grouped and commented).
2. `universe.py`: loader and validator (DATA 2).
3. `prices.py`: `source`, `read`, `write`, `replace` (DATA 3).
4. `sample.py`: the deterministic SYNTHETIC generator, with the gap, stale, no-data and short-history tickers (DATA 4).
5. `fetch.py`: the yfinance downloader (DATA 5); it is only imported by the CLI.
6. `metrics.py`: `compute`, `FIELDS`, `RETURNS` (DATA 7, METRICS 3).
7. `build.py`: validated atomic publish of `web/data.js` (DATA 9 and 10). Slice 1 has no `project` field and no `project.css`; the header label is the literal `ETFs`; slice 1b adds both.
8. `cli.py` and `__main__.py`: `sample`, `fetch`, `build`; exit 0 ok, 2 failure (DATA 15). Slice 1 reads `data/universe.csv` and `var/prices.csv` relative to the working directory; slice 1b adds `--project`.
9. `web/index.html` (UI 16; no `project.css` link yet), `web/app.js` (UI 15: shell, router, helpers, shared table, error and placeholder panels) and `web/tabs/screener.js` (UI 15.6).
10. `tests/test_slice1.py`: the six tests of DATA 13 (items 1 to 6).

**data.js fields.** Top level `schema generated as_of source synthetic funds groups`. Per fund `ticker name short_name group family currency status first_date last_date days_stale r_1m r_3m r_6m r_ytd r_1y r_3y_ann vol_1y maxdd_1y why`. `groups[]`: `name family index n`.

**Done.**
- [ ] `python -m etf_arcade sample` writes a SYNTHETIC `var/prices.csv`; `build` writes `web/data.js`.
- [ ] `web/index.html` shows the Screener with every universe fund, the source in the header, working sort and search.
- [ ] A missing `data.js` shows the error panel with the commands to run.
- [ ] `#tab=heatmap` (any other tab) shows `Not built yet — slice N`.
- [ ] A failed `build` exits 2 and leaves `web/data.js` byte-identical; fewer than half the universe priced is a failure.
- [ ] `python -m unittest -q` passes (6 tests).

**Verify.** Standard verify block. Expected for the committed universe: `build` reports 520 funds, as_of 2025-12-26, source synthetic; the Screener shows "520 of 520 ETFs"; two `sample --force` runs give the same `var/prices.csv` hash; a missing `data.js`, a schema mismatch and a throwing tab each show their error panel. `python -m etf_arcade fetch` (needs `pip install yfinance` and network) followed by `build` gives `synthetic: false`; `python -m etf_arcade sample --force` restores synthetic data.

**Out of scope.** Projects (1b), settings, drawer, charts, benchmarks, every other tab.

---

## Slice 1b: Projects

**Goal.** One build serves several projects (a universe, its prices, a title and optional theme and constants) selected with `--project DIR`, so a second universe, such as the European example or a private one, needs no code change.

**Depends on.** Slice 1.

**Read first.** `docs/reference/DATA.md` sections 2, 2b, 3, 12, 13 (items 7 and 8) and 15.

**Files to create or change.** `etf_arcade/project.py` (new), `etf_arcade/cli.py`, `etf_arcade/build.py`, `web/index.html` (the `project.css` link), `tests/test_slice1b.py` (new). Already in the repository and only verified here: `settings.toml` (the default project), `examples/europe/` (`universe.csv`, `settings.toml`, `README.md`), `data/universe_ucits.csv`.

**Steps.**
1. `project.py`: `load`, `apply`, `DEFAULT_META`, `DEFAULTS`; validation of `settings.toml` including the shape of `[[benchmarks]]` and `[cash]` (DATA 2b and 15). `universe.TICKER` is the single ticker regex (it already allows `^` and Yahoo suffixes); `project.py` imports it.
2. `cli.py`: global `--project DIR` (default `.`); `project.load` then `project.apply(proj.constants)` before any command; `build` gets the project object (DATA 15).
3. `build.py`: `proj` parameter; the `project` object in `data.js` (validated); `web/project.css` written from `DIR/theme.css` or the empty comment, before `data.js` (DATA 10).
4. `web/index.html` links `project.css` after `theme.css`.
5. Check the shipped files: the root `settings.toml` and `examples/europe/settings.toml` load without error (their `[[benchmarks]]` and `[cash]` are only shape-checked until slice 4), and `examples/europe` builds from sample prices.
6. `tests/test_slice1b.py`: the two tests of DATA 13 (items 7 and 8).

**data.js additions.** Top level `project`: `{title, subtitle, currency, universe_label, dir_name}`.

**Done.**
- [ ] `--project DIR` works for `sample`, `fetch` and `build`; the default project behaves as in slice 1.
- [ ] `settings.toml`: `[project]` and `[constants]` applied; unknown constant, unknown key or wrong type exits 2 with `data.js` untouched.
- [ ] `build` writes `web/project.css` and `data.js.project`; the header shows the title and subtitle, and the count line uses `universe_label`.
- [ ] `examples/europe` builds from sample prices (141 funds, 31 groups, title `ETF ARCADE EU`).
- [ ] `python -m unittest -q` passes (8 tests).

**Verify.**

```
python -m etf_arcade sample --force && python -m etf_arcade build
python -m etf_arcade --project examples/europe sample --force && python -m etf_arcade --project examples/europe build
python -m unittest -q
python -m etf_arcade sample --force && python -m etf_arcade build    # restore the default project
```

**Out of scope.** Consuming benchmarks and cash (slice 4), review marks and `review-status` (slice 12), FX conversion.

---

## Slice 2: Shared chrome and settings

**Goal.** A settings strip (groups, return period, chart history, names) that persists, applies to the Screener, and a detail drawer plus deep links.

**Depends on.** Slices 1 and 1b.

**Read first.** `docs/PRODUCT.md` section 3 (all subsections) and section 5; `docs/reference/UI.md` sections 2, 4, 5, 6, 7 and 12.

**Files to create or change.** `web/app.js`, `web/index.html` (settings slot, drawer slot, tooltip node), `web/app.css` (new, linked after `theme.css`; strip, picker, drawer classes; `.pill` exists, `.chip` arrives in slice 6), `web/tabs/screener.js`, `tests/test_slice2.py`.

**Steps.**
1. In `app.js` add persisted state `{period, history, names, hidden}` with defaults `r_3m`, `3Y`, `short`, `[]` (slice 4 adds `benchmark`, slice 12 `review`). Persist in `localStorage` key `arcade.settings.v1`, every access in try/catch, invalid saved values fall back to defaults. Every UI string that names the funds in the plural uses `ARCADE.label` (UI.md section 7), including the new strip, picker and drawer texts.
2. Add the hash codec for `tab fund series q sort x y m` (UI.md section 4): tab changes and fund open/close push history; sort, search, settings and redraws use `replaceState`. Settings never go in the hash.
3. Implement `ARCADE.set(patch)` (merge, persist, update hash, one coalesced redraw) `ARCADE.scope(data)` (reduce `funds` and `groups` to visible groups; colours keep the full-universe `groups[].index`) and `ARCADE.fund(ticker)` (unscoped lookup into `ARCADE_DATA`).
4. Build the strip: group picker (search, Show all, Hide all, swatch checkbox per group listed by family, scope line such as `18 of 59 groups selected across all tabs`), period select (`r_1m r_3m r_6m r_ytd r_1y r_3y_ann`, labelled 1M 3M 6M YTD 1Y 3Y), history select (`YTD 1Y 2Y 3Y 5Y MAX`), names toggle (`short` or `full`).
5. The tab bar already has its roles, arrow, Home and End keys and narrow-screen scroll (slice 1). Still missing: restore the departing tab's `q/sort/x/y/m` from `ARCADE.tabState(id)` on return (memory only, no storage key; other per-tab controls live there too) and Escape handling for tooltip, drawer and picker.
6. Add the tooltip (`ARCADE.tip.show/hide`) and the drawer (side panel, bottom sheet under 900 px): name, ticker, group, family, currency, history range, status, all metrics with null reasons, link to the Fund tab, Escape closes and returns focus.
7. Screener: period column first and default sort follows the period until a header is clicked; names follow the setting; hover text is `<name> (<ticker>)`; a row opens the drawer. Remove the local group `<select>`: the shared group picker replaces it. A deep-linked fund in a hidden group still opens, with the pill `Outside current group filter`.
8. Add `tests/test_slice2.py` with two static tests: no hex colour literal in `web/*.js` and `web/tabs/*.js`, and every `web/tabs/*.js` registers an id from the `TABS` array, which the test parses from `app.js` with a regex. Both ignore the exact paths `web/data.js`, `web/series/*` and `web/vendor/*` (`web/tabs/data.js` stays covered).

**data.js additions.** None.

**Done.**
- [ ] Hiding a group removes it from the Screener, the picker scope line updates, and a reload keeps it hidden.
- [ ] Changing the return period moves that column first and re-sorts until the reader clicks a header; changing history never resets sort or search.
- [ ] `#tab=screener&fund=SPY` opens the drawer; Escape closes it; Back moves between tabs and funds only.
- [ ] A shared link with `#tab=...` does not overwrite another reader's settings.
- [ ] Short names by default, full name and ticker on hover; Full toggle swaps them.
- [ ] 390 px wide: settings collapse, no horizontal page scroll.

**Verify.** Standard verify block; headless dump of `#tab=screener&fund=SPY` (default project; with another project use one of its tickers, for example the first row of its `universe.csv`) contains the drawer text; `python -m unittest tests.test_slice2 -q`.

**Out of scope.** Charts, benchmarks (the Benchmark select arrives in slice 4).

---

## Slice 3: Fund tab and charting base

**Goal.** Open any ETF in the Fund tab and see its facts, a price chart and a drawdown chart over the chosen history.

**Depends on.** Slice 2.

**Read first.** `docs/PRODUCT.md` section 4.14 (slice 3 parts only); `docs/reference/UI.md` sections 4 (history window), 9 and 13 item 14; `docs/THEME.md` section "Chart colour rules"; `docs/reference/METRICS.md` section 3.2 (rows `dd_now`, `maxdd`, `r_si`) and section 9 (row `drawdown_t`); `docs/reference/DATA.md` section 9 (row `series_file` and the series-file paragraph) and section 10.

**Files to create or change.** `web/vendor/d3.min.js` and `web/vendor/d3.LICENSE.txt` (d3 7.9.0, downloaded once from `https://cdn.jsdelivr.net/npm/d3@7.9.0/dist/d3.min.js` and `https://cdn.jsdelivr.net/npm/d3@7.9.0/LICENSE` and committed; never loaded from a CDN at runtime; this one download is allowed even where price downloads are not), `NOTICE` (d3 entry, ISC licence), `web/tabs/fund.js`, `web/index.html` (script order from UI.md section 1), `web/app.js` (`ARCADE.fit`, `.series`, `.lineChart`), `web/app.css`, `etf_arcade/constants.py`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice3.py`.

**Steps.**
1. Vendor d3 and its licence file; add the script tag after `app.js` (order: `app.js`, `vendor/d3.min.js`, `tabs/*.js`, `data.js`) and the d3 (ISC) entry to `NOTICE`.
2. Series file per `DATA.md` section 9: `web/series/<ticker>.js` registers `{generated, dates, adj_close}` on `window.ARCADE_SERIES[<ticker>]`; a gap is a missing date. Add `LINE_GAP_DAYS` to `constants.py` and publish it as `params.line_gap_days`.
3. `build.write_series` writes the files into `web/series.tmp`, swaps the directory in, and `data.js` is replaced last (DATA.md section 10); `funds[].series_file` is `series/<ticker>.js`, absent for `no_data` funds.
4. Add `r_si`, `dd_now`, `maxdd` to `metrics.py` per METRICS.md and extend `metrics.FIELDS` with them, each null with a reason code (`insufficient_history`, or `no_data` for a fund without closes), so `build._validate` keeps checking `why`.
5. `ARCADE.series(ticker)` loads a series file by injecting a `<script>` tag (cached, resolves when loaded) and shows "Reload: data was rebuilt" when its `generated` differs from `ARCADE_DATA`. Build the shared `ARCADE.lineChart` (axes, gap breaks at `params.line_gap_days`, crosshair, end labels with dodge, at most 18 labels); slices 4 to 7 reuse it.
6. `fund.js`: ETF select listing the visible groups plus the current fund (choosing one adds a history entry), header (names, ticker, group, family, status pills), KPI tiles (the slice-1 returns, `r_si`, `vol_1y`, `maxdd_1y`, `maxdd`, `dd_now`), price chart over the history window (UI.md section 4; rebased to 100; log scale for 5Y and MAX), drawdown chart computed in the browser from `adj_close` with the peak taken over full history (METRICS section 9). Colours through `ARCADE.css` only.
7. Peers table: the visible members of the ETF's group through `ARCADE.table`, sorted by the shared period, current ETF underlined, row click switches ETF. Fund tab with no fund in the hash shows `Pick a fund`. The drawer's "Open Fund tab" link goes live.
8. `ARCADE.checkLabels()` and the `check=1` hash flag (UI.md section 8): from here on every headless dump uses `&check=1` and must read `overlaps 0, clipped 0`.
9. One test: a series with a gap produces a series file with the date absent and `maxdd` computed on the full history.

**data.js additions.** Top level `params`. Per fund `r_si dd_now maxdd series_file`.

**Done.**
- [ ] `#tab=fund&fund=SPY` draws price and drawdown charts from `file://`.
- [ ] Changing chart history changes the charted range only; sort, search and `fund` stay.
- [ ] A fund with a gap shows a broken line; a `no_data` fund shows the empty state with its reason.
- [ ] The select switches ETF and the peers table lists the same group with the current ETF underlined.
- [ ] No colour literal in chart code (the static test passes).
- [ ] Chart axis labels fit the panel at 390 px.

**Verify.** Standard verify block; `python -m unittest -q` (including `tests.test_slice2`); headless screenshot of the Fund tab at 1400 and 390 px.

**Out of scope.** Benchmark line and benchmark-dependent tiles (slice 4), Chart wall.

---

## Slice 11: Data tab and data-quality screen

**Goal.** One tab that shows exactly what the build checked and flagged, with the data-quality screen behind it. Build it directly after slice 3.

**Depends on.** Slice 3 (so only slices 1 to 3).

**Read first.** `docs/PRODUCT.md` section 4.15; `docs/reference/UI.md` section 13 item 15; `docs/reference/DATA.md` sections 8, 10, 11 and 12; `docs/reference/METRICS.md` section 10.

**Files to create or change.** `web/tabs/data.js`, `web/index.html`, `web/app.css`, `etf_arcade/quality.py` (new), `etf_arcade/constants.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice3.py` (see step 2), `tests/test_slice11.py`.

**Steps.**
1. `quality.py` owns the screen, `screen(prices, issues=(), tickers=None) -> (cleaned prices, flags)` exactly as DATA section 11 says, importing only `constants.py`; add `GAP_DAYS MOVE_WARN SPIKE_MOVE SPIKE_REVERT GAP_BREAK_DAYS CUTOFF_LAG_BDAYS` to `constants.py` (DATA section 12). Checks per DATA section 11: missing prices, unknown tickers, dropped rows (the `issues` from `prices.read`), staleness, gaps, large moves, spikes, gap breaks, short history, coverage, the cutoff date (`flags["coverage"]["cutoff"]`, computed here once and only reported; slices 7 and 9 consume it), source label.
2. `build.py` calls `screen` before `metrics.compute()` and feeds the cleaned prices to metrics and series files. Spike exclusion and gap-break segmentation happen inside the screen (stored prices stay unchanged, no interpolation, no splice); `metrics.py` never imports `quality.py`. Because this can change earlier metrics, rerun all tests and update DATA.md section 7 as DATA 11 says. The slice 3 test builds a fund with a gap longer than `GAP_BREAK_DAYS`; update its expectation to the segment after the gap (tests that call `metrics.compute` directly are unaffected).
3. Add top-level `quality` to `data.js` (shape and example in DATA section 11): `counts` (including `move`, `break`, `short_3y`), `params` (the live constants), coverage, dropped-row reasons and one entry per fund with `n_obs`, flags, gaps, spikes and moves.
4. Data tab (module `web/tabs/data.js`, tab id `data`): history timeline (one bar per fund from first to last date, coloured by group), coverage and quality table with search and the four flag chips `stale` `gap` `spike` `break`, method notes showing the live windows (`quality.params`), publication record (`generated`, `source`, `synthetic`). The counts are for the whole universe and are labelled `all <universe_label>`; under a group filter only the table and the timeline are scoped.
5. Tests: (a) a spike (up 40% then back within 5%) is excluded from the weekly series and reported; (b) a gap longer than `GAP_BREAK_DAYS` makes metrics use only the segment after it, sets `break_date` and leaves the stored prices unchanged.

**data.js additions.** Top level `quality` (shape per DATA section 11; per-fund lists live inside it).

**Done.**
- [ ] The Data tab shows every flag the build produced and nothing else.
- [ ] A stale, a short-history and a no-data fund each appear with the right status.
- [ ] Spike and gap-break handling is tested, and a failed build still leaves the live page unchanged (the slice-1 test still passes).
- [ ] The checklists of slices 1 to 3 are ticked and their tests still pass.

**Verify.** Standard verify block; break `var/prices.csv` on purpose, run `build`, check the exit code is 2 and `web/data.js` is unchanged, then restore with `python -m etf_arcade sample --force`.

**Out of scope.** Review marks (slice 12), scheduled refresh, any server component.

---

## Slice 12: Review marks

**Goal.** A reader flags broken prices, excludes a fund or moves it to another group in the browser, exports the marks, saves the file into the project folder and rebuilds; `prices.csv` and `universe.csv` stay untouched. A later mark can clear earlier ones.

**Depends on.** Slice 11 (and so slices 1 to 3); build it right after 11.

**Read first.** `docs/PRODUCT.md` section 4.16; `docs/reference/DATA.md` section 14; `docs/reference/METRICS.md` section 12; `docs/reference/UI.md` section 14.

**Files to create or change.** `etf_arcade/review.py` (new, stdlib and `constants.py` only), `etf_arcade/build.py`, `docs/reference/DATA.md`, `docs/reference/UI.md`, `etf_arcade/cli.py` (`review-status`), `web/review.js` (new classic script after the tab scripts), `web/app.js` (toggle, state key, status pills), `web/index.html`, `web/tabs/fund.js`, `web/tabs/data.js`, `web/app.css`, `tests/test_slice12.py`.

**Steps.**
1. `review.py`: `load(dir, universe)` reads and validates the marks (DATA section 14), resolves `clear` marks and returns the effective marks plus the warnings (marks for tickers outside the universe); `apply(prices, funds, review)` drops ranges and replaces groups. No writes.
2. `build.py`: read prices, apply review, screen, metrics; leave excluded funds out of every aggregate through the member filter of DATA 8; add the fields below. Invalid file: exit 2, `data.js` kept; a mark for an unknown ticker only adds a warning.
3. `cli.py`: `review-status` (also prints the warnings).
4. `app.js`, `review.js`, the Fund action bar (with `Clear`) and the Data queue with the warnings list, per UI.md section 14.
5. Tests: (a) a `broken_price` range removes exactly its closes from the series file and metrics, nothing filled; a whole-series mark gives `review_broken`, null metrics, `why` `review_broken_price`; (b) an `exclude` mark keeps the fund in `funds[]` with status `review_excluded`, metrics computed, and `groups[].n` counting the rest; a later `clear` mark restores it; a mark for an unknown ticker builds with exit 0 and shows in `review.warnings`.

**data.js additions.** Fund `status` values `review_broken`, `review_excluded`; fund `group_original`; `why` code `review_broken_price`; top level `review` `{files, marks, counts, warnings}`. The aggregates added later (slices 4 to 9) skip these funds through the member filter; slice 5 tests it for group aggregates.

**Done.**
- [ ] A broken range leaves a gap in the charts and no trace in any metric; a whole-series flag shows `Prices removed` and em dashes.
- [ ] An excluded fund stays listed, dimmed, and is outside every aggregate the build publishes (today `groups[].n`; later slices add theirs).
- [ ] A moved fund sits in the target group on every tab, with `Moved from <group>` in hover.
- [ ] A `clear` mark cancels the earlier marks it names; the Fund tab `Clear` action exports one.
- [ ] A mark for a ticker that is not in `universe.csv` is a warning in the Data tab and in `review-status`, not a build failure.
- [ ] Review off hides all actions; on, they work by keyboard.
- [ ] Export, save into `review/`, rebuild: pending marks vanish and the Data tab lists the marks with file names. Pending marks change no number.
- [ ] A malformed file (`to` before `from`) exits 2 and leaves `web/data.js` byte-identical.
- [ ] `python -m unittest -q` passes.

**Verify.** Standard verify block; export one mark per mode (including `clear`), save under `review/`, run `review-status` and `build`; then break the file and check exit 2.

**Out of scope.** Server endpoints, editing `universe.csv` or `prices.csv`, multi-user merge. Built marks are cancelled by a later `clear` mark, never edited from the browser.

---

## Slice 4: Benchmarks, cash and the Market tab

**Goal.** The benchmarks and the cash series the project configures in `settings.toml` feed excess returns, Sharpe and beta, a benchmark switch in the settings strip, and the Market tab.

**Depends on.** Slice 3. In the build order it comes after slices 11 and 12, so `build.py` already screens prices and applies review marks, and every aggregate here skips excluded funds (DATA.md section 8).

**Read first.** `docs/reference/METRICS.md` sections 4, 5 and 11; `docs/PRODUCT.md` section 4.13 and the benchmark-dependent parts of 4.14; `docs/reference/UI.md` sections 4 and 7 and section 13 items 13 and 14; `docs/reference/DATA.md` sections 2b, 3, 4, 5 and 9 (the reference-dependent rule).

**Files to create or change.** `etf_arcade/sample.py`, `etf_arcade/fetch.py`, `etf_arcade/prices.py`, `etf_arcade/cli.py` (pass the extra tickers), `etf_arcade/metrics.py`, `etf_arcade/build.py`, `web/app.js`, `web/tabs/market.js`, `web/tabs/fund.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice4.py`.

**Steps.**
1. Extra tickers. Benchmark component tickers and the cash ticker are series in `prices.csv` (universe members or not; DATA 3). `prices.read(path, universe, extra=(), cash=None)` accepts the `extra` tickers besides the universe and exempts the `cash` ticker from the `adj_close > 0` rule (finite only). `build` fails when the cash ticker is a universe member, then splits the frame: universe and benchmark rows go through the screen and metrics, cash rows bypass the screen.
2. `sample` and `fetch` also produce the benchmark and cash tickers (DATA 4 and 5). Synthetic ones are labelled SYNTHETIC like everything else.
3. `metrics.py`: `blend` (weekly-rebalanced chain for any `components`, METRICS 4, no splicing when a component is missing), weekly cash return (7-day carry only, never past the last observation, null when any day lacks a rate), and per benchmark `rel_3m`, `rel_1y`, `beta_3y`; `sharpe_3y` from cash only. `metrics.FIELDS` gains `sharpe_3y`; a new `metrics.REF_FIELDS = ["rel_3m", "rel_1y", "beta_3y"]` lists the object-valued fields. `sharpe_3y` is null with `cash_missing` (no `[cash]`) or `cash_incomplete`; zero cash is never assumed.
4. `build.py`: top-level `benchmarks` (the `[[benchmarks]]` list in order, each with `series_file`), `cash` (or `null`) and `market` with the METRICS section 11 ids (`above_50d above_200d new_hi new_lo dispersion_21d impl_corr_63d`); reference-dependent fund fields as objects keyed by benchmark id with the `why` shape of DATA 9; series files `series/_bench_<id>.js` and `series/_cash.js`; `_validate` checks the object rule.
5. `app.js`: state `benchmark` (persisted in `arcade.settings.v1`, default the first benchmark, invalid saved value falls back); a Benchmark select in the settings strip (hidden without benchmarks); `ARCADE.benchmark()` and `ARCADE.ref(row, field)`; `ARCADE.why` reads object-valued `why` entries with the selected id. Switching never resets the fund, `series`, filters, searches, sorts or custom metrics (UI 4).
6. `market.js` (lines through `ARCADE.lineChart`): the six line cards and the member squares of PRODUCT 4.13; the reference drawdown card follows the selected benchmark; under a group filter the breadth and implied-correlation cards show the standard unavailable note (nothing is recomputed); the cash card is named by `cash.label`, is not group-scoped, and says `No cash series` without `[cash]`.
7. Fund tab gains the selected benchmark as a dashed line and the tiles `rel_3m`, `rel_1y`, `sharpe_3y`, `beta_3y`, each labelled `vs <benchmark label>` where it depends on the benchmark. A missing benchmark value shows `—` with `reference_missing`, never a number.
8. Tests: (a) the blend chain goes null after a missing component and does not restart by splicing; (b) `sharpe_3y` is null with `cash_incomplete` when one cash week is missing and with `cash_missing` when the project has no `[cash]`.

**data.js additions.** Top level `benchmarks cash market`. Per fund `rel_3m rel_1y beta_3y` (objects keyed by benchmark id) and `sharpe_3y`.

**Done.**
- [ ] Market tab draws six cards and the member squares; gaps stay gaps.
- [ ] The cash series is always named by `cash.label`; a missing cash week gives an em dash and a reason on Sharpe; no `[cash]` gives `cash_missing`.
- [ ] The benchmark label shows wherever a number or line depends on it; `rel_*`, beta, the reference drawdown and the Fund tab line all follow the Benchmark select, and the switch keeps fund, filters, searches, sorts and custom metrics.
- [ ] Under a group filter breadth and implied correlation read `Unavailable for a filtered group selection...`.
- [ ] Member squares are not clickable.
- [ ] A benchmark ticker that is also a universe member shows one shared series (no duplicate, no special case).

**Verify.** Standard verify block; optionally a real-data run with `python -m etf_arcade fetch` (needs `pip install yfinance`); headless screenshots of `#tab=market` and `#tab=fund&fund=<ticker>` for two different benchmarks.

**Out of scope.** Alpha and factor regressions, Risk tab.

---

## Slice 5: Overview and Heatmap

**Goal.** A landing page with KPI tiles, an ETF treemap labelled by group, a group leaderboard and group index lines, and a Heatmap tab with selectable colour and size metrics.

**Depends on.** Slice 4.

**Read first.** `docs/PRODUCT.md` sections 4.1 and 4.2; `docs/reference/UI.md` section 9 and section 13 items 1 and 2; `docs/reference/METRICS.md` sections 3.2 (rows `mom_12_1 vs_200d`) and 6.

**Files to create or change.** `web/tabs/overview.js`, `web/tabs/heatmap.js`, `web/app.js`, `web/app.css`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice5.py`.

**Steps.**
1. Add per-fund `mom_12_1 vs_200d` (null with reasons when history is short).
2. Add group aggregates to `groups[]`: `group_return` (one value per standard return period: the return of the group index ending at the coverage cutoff, METRICS 6), `med_r_1m med_r_3m med_r_1y`, `breadth_200d`; add the `group_index` series (equal-weight chain, METRICS section 6, chain breaks counted) and a `returns` object (the six period returns) on each `benchmarks[]` entry for the leaderboard benchmark line. Aggregates take their members through the member filter of DATA 8 (excluded and broken-price funds never enter).
3. `overview.js`: six KPI tiles, treemap (equal tiles, text-free, group titles only, hover and click), leaderboard (top 15 and bottom 15 when over 30 groups) with the reference line, rebased group index lines (through `ARCADE.lineChart`), Top 12 and Bottom 12 with no shared fund. Rows hold one or two panels.
4. `heatmap.js`: colour metric and size metric selectors, group-by `group` or `family`, tile text only when the tile is large, groups-by-horizons table with diverging cells (scaled by the 95th percentile of absolute values).
5. All panels use scoped data; aggregates fitted to the full universe obey the filtered-selection rule.
6. Set `DEFAULT_TAB` to `overview` and use it as the unknown-tab fallback (the default tab is `screener` before this slice).
7. Tests: (a) the index chain skips members without a return (not zeroed) and `group_return["r_3m"]` equals the ratio of `group_index` levels at the cutoff and at its 3-month anchor; (b) a `review_excluded` fund does not change its group's `group_return` or `med_r_3m` (equal to a build without it).

**data.js additions.** Per fund `mom_12_1 vs_200d`. In `groups[]`: `group_return med_r_1m med_r_3m med_r_1y breadth_200d`. Top level `group_index` (series per group name); `benchmarks[].returns`.

**Done.**
- [ ] KPI tiles show real values and obey the group filter.
- [ ] Treemap tiles carry no text; group blocks are titled; hover shows `<name> (<ticker>)`; click opens the drawer.
- [ ] Changing the return period updates every panel without a reload.
- [ ] Heatmap controls change the treemap only; the table sort follows the period until the reader clicks a header.
- [ ] No panel is wider than two per row; at 390 px they stack.

**Verify.** Standard verify block; headless screenshots of `#tab=overview` and `#tab=heatmap` at 1400 and 390 px; `python -m unittest -q`.

**Out of scope.** Chart wall, Groups tab statistics tables.

---

## Slice 6: Chart wall and Groups

**Goal.** A wall of small price cards sectioned by group, and a Groups tab with indices, relative performance and statistics.

**Depends on.** Slice 5 (reuses `group_index`).

**Read first.** `docs/PRODUCT.md` sections 4.4 and 4.5; `docs/reference/UI.md` section 8, section 9 and section 13 items 4 and 5; `docs/reference/METRICS.md` sections 6 and 9.

**Files to create or change.** `web/tabs/wall.js`, `web/tabs/groups.js`, `web/app.js` (legend chips, lazy building), `web/app.css`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice6.py`.

**Steps.**
1. Extend the series files with weekly arrays on `wdates`, named by METRICS ids: `roll_ret` and `drawdown_t` (full-history peak).
2. Add to `groups[]`: `beat_ref_3m` (an object keyed by benchmark id), `dispersion_1y`, `med_vol_1y`, `med_maxdd_1y`.
3. Legend chips in `app.js` (UI.md section 8): toggle, hover preview, Reset, `aria-pressed`, selection kept per panel.
4. `wall.js`: one card per ETF sectioned by group; controls for chart kind (growth, drawdown, rolling return), sort metric and direction, card size; cards are built lazily with `IntersectionObserver`; shared or per-group y-scale; click opens the drawer.
5. `groups.js`: equal-weight group indices (growth, relative to the reference, drawdown), statistics table with switchable column sets (returns, risk, breadth), per-group panels with member lines in the group colour. Legend chips drive every chart on the tab. Both tabs draw their lines with `ARCADE.lineChart`.
6. Tests: one for `beat_ref_3m` skipping null members (not zeroing them).

**data.js additions.** Series files gain `roll_ret drawdown_t`. In `groups[]`: `beat_ref_3m dispersion_1y med_vol_1y med_maxdd_1y`.

**Done.**
- [ ] The wall renders every visible ETF; cards for off-screen groups are built only on scroll.
- [ ] Sort metric, direction and card size controls work and survive a history change.
- [ ] Clicking a legend chip hides or shows that group on the Groups tab and every chart follows; hover previews and Reset restores all.
- [ ] Labels fit inside each panel at 390 px; full names stay in hover text.
- [ ] Rendering 520 cards stays responsive (no visible freeze on scroll).

**Verify.** Standard verify block; headless screenshots of `#tab=wall` and `#tab=groups`.

**Out of scope.** Rotation, rank charts.

---

## Slice 7: Leadership, Rotation and Calendar

**Goal.** Three tabs about who is leading, how groups rotate, and how months and years have behaved.

**Depends on.** Slice 6.

**Read first.** `docs/PRODUCT.md` sections 4.3, 4.6 and 4.12; `docs/reference/UI.md` section 13 items 3, 6 and 12; `docs/reference/METRICS.md` sections 7.3, 7.4 and 7.5 and the coverage-cutoff row of section 10; `docs/reference/DATA.md` section 11 (row Cutoff).

**Files to create or change.** `web/tabs/leaders.js`, `web/tabs/rotation.js`, `web/tabs/calendar.js`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice7.py`.

**Steps.**
1. Use the coverage cutoff that slice 11 computes (`quality` screen, `CUTOFF_LAG_BDAYS` in `constants.py`): `build` passes the cutoff date into the rotation functions in `metrics.py` as a parameter, never importing `quality.py`; rotation outputs end at the cutoff and every footnote says the actual end date.
2. Per fund add `from_52wh`, `from_52wl`. Per group add `ew_ratio` and `ew_tells` (METRICS 7.4; objects keyed by benchmark id).
3. Top-level `rotation`: per benchmark id, then per group and fund, `rs`, `rrg_ratio`, `rrg_mom`, `rrg_quadrant` (60 weekly levels minimum, last 156 points for groups, 52 for ETFs) and `rank_trail` by month end.
4. Top-level `calendar`: per ticker and per group `monthly_return` (consecutive month-end closes only, a missing month stays null and nulls the next), `year_return` with the month count, seasonal `season_mean season_median season_hit`.
5. `leaders.js`: scatter (`lead_x` against `lead_y`, log x), early-warning cards (group index over the reference, from the published series), participation lines (the published `market.above_200d`; unavailable under a group filter). Participation and rank lines use `ARCADE.lineChart`. `rotation.js`: relative-rotation scatter with tails and end-week slider, rank bump chart, leadership table. `calendar.js`: month-by-year grid for the selected ETF or group (`series` in the hash), seasonality bars, calendar-year table with `*` for part years.
6. Tests: (a) a missing month nulls two monthly returns and is not bridged; (b) quadrant names follow ratio and momentum at 100.

**data.js additions.** Per fund `from_52wh from_52wl`. Groups `ew_ratio ew_tells`. Top level `rotation calendar`.

**Done.**
- [ ] Each tab renders with scoped data; a hidden group disappears from every chart (participation excepted: it reads the unavailable note).
- [ ] Scatter labels follow the budget and never overlap; full text in hover.
- [ ] Calendar part years carry `*` and `n of 12`; missing months are blank, never zero.
- [ ] Rotation tail length is an input; axes are symmetric around 100.
- [ ] A fund or series chosen on one tab is kept when switching tabs.

**Verify.** Standard verify block; headless screenshots of the three tabs.

**Out of scope.** Risk statistics, correlation, PCA.

---

## Slice 8: Risk & return

**Goal.** Risk and return scatter with selectable axes, capture ratios and a drawdown watch table.

**Depends on.** Slice 7.

**Read first.** `docs/PRODUCT.md` section 4.9; `docs/reference/UI.md` section 8 and section 13 item 9; `docs/reference/METRICS.md` sections 3.2 (rows `dd_now maxdd`) and 5 (rows `up_cap down_cap`).

**Files to create or change.** `web/tabs/risk.js`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice8.py`.

**Steps.**
1. Add per fund `up_cap down_cap` (objects keyed by benchmark id).
2. `risk.js`: scatter with x and y selectors (`vol_1y`, `r_3y_ann`, `maxdd_1y`, `beta_3y`, `sharpe_3y`) and colour by group; up-versus-down capture scatter with a diagonal; per-group dot strip with medians; drawdown watch table using the shared `ARCADE.table`.
3. Axis metrics are `x` and `y` in the hash and are kept per tab. Changing chart history must not reset them.
4. One test: capture ratios are null when a regime is empty.

**data.js additions.** Per fund `up_cap down_cap`.

**Done.**
- [ ] Axis selectors change the scatter and are kept after a history change.
- [ ] Funds with a null on either axis are listed in the footnote as not plotted, not placed at zero.
- [ ] Labels follow the budget; hover shows full name and both values.
- [ ] The capture scatter draws its diagonal.

**Verify.** Standard verify block; headless screenshot of `#tab=risk`.

**Out of scope.** Signal lab.

---

## Slice 9: Correlation and Structure & PCA

**Goal.** A correlation heat matrix with peers, and a PCA view of how much the universe moves together.

**Depends on.** Slice 8.

**Read first.** `docs/PRODUCT.md` sections 4.7 and 4.8; `docs/reference/UI.md` sections 6 and 11 and section 13 items 7 and 8; `docs/reference/METRICS.md` sections 7.1 and 7.2 and 10 (coverage cutoff).

**Files to create or change.** `web/tabs/correlation.js`, `web/tabs/pca.js`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice9.py`.

**Steps.**
1. Compute `corr_1y_daily` and `corr_3y_weekly` pairwise-complete with the `CORR_*` constants over windows that end at the coverage cutoff passed in from the slice-11 screen, and the display order by average-linkage clustering written in numpy (no scipy; missing-pair and deterministic leaf-order rules in METRICS 7.1; one order per window). Store the matrices compactly (rounded to 3 decimals, the stated exception to rounding only for display; upper triangle only, nulls kept) and describe the layout in `DATA.md`, since a full 520 by 520 matrix is large.
2. Compute `pca_fixed` on the `PCA_WEEKS` (104) complete-case sample ending at the cutoff, `pca_status`, `pca_n`, `pca_orient`, and `pca_rolling` (52 weeks). Fewer than `PCA_MIN_FUNDS` (8) complete ETFs gives `insufficient_data` and no numbers.
3. Per fund `pc1 pc2` loadings (null when the fund is not in the sample).
4. `correlation.js`: canvas matrix (more than about 2,000 cells), clustered or group ordering, selecting an ETF (`tabState.sel`) shows its 12 closest and 8 furthest peers as bars, and two lists of 15 highest and lowest pairs (ties broken by ticker). Under a group filter show the sub-matrix with the clustering order kept.
5. `pca.js`: explained-variance bars, rolling PC1 share line, loadings scatter with group outlines. Under a group filter every full-universe PCA panel shows `Unavailable for a filtered group selection. Restore all groups to view this full-universe analysis.`
6. Tests: (a) fewer than 8 complete ETFs gives `insufficient_data`; (b) PC1 sign is oriented to a positive mean loading.

**data.js additions.** Top level `correlation` (matrices and order) and `pca` (`pca_status pca_n pca_fixed pca_rolling`). Per fund `pc1 pc2`.

**Done.**
- [ ] Matrix hover shows both names and the value; missing pairs are blank.
- [ ] PCA shows `Insufficient data: needs 8, has M` when the sample is too small.
- [ ] A group filter never relabels a full-universe PCA number as recalculated.
- [ ] Matrix and PCA stay inside their panels at 390 px.

**Verify.** Standard verify block; headless screenshots of both tabs, once with a group hidden.

**Out of scope.** Signal lab, implied correlation changes.

---

## Slice 10: Signal lab

**Goal.** Pick a fund or group, a horizon and a window; see six stacked panels and a cross-section of today's z-scores.

**Depends on.** Slice 9.

**Read first.** `docs/PRODUCT.md` section 4.10; `docs/reference/UI.md` section 13 item 10; `docs/reference/METRICS.md` section 8.

**Files to create or change.** `web/tabs/lab.js`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `docs/reference/DATA.md`, `tests/test_slice10.py`.

**Steps.**
1. Per fund add the canonical signal values `rel_13w z_13w pct_13w` (objects keyed by benchmark id) and, in the series files, weekly arrays of `rel_13w` and `z_13w` on `wdates` over full history (each an object keyed by benchmark id).
2. Add `z_rank` per fund (object keyed by benchmark id).
3. `lab.js`: controls for ETF or group, horizon (4, 13, 26, 52 weeks) and window; six stacked panels (indexed price, excess return, rolling mean with bands, z-score, percentile, distribution of z with today's marker); then a bar chart of today's z for every visible group (UI.md 13.6). Statistics use full history before slicing to `history`.
4. Non-canonical horizon or window recomputes in the browser only if the formula is defined for any window in METRICS section 8; otherwise the control is disabled with a reason.
5. One test: `z_13w` is null with fewer than 104 finite `rel` values and when the standard deviation is zero.

**data.js additions.** Per fund `rel_13w z_13w pct_13w z_rank`; series files gain weekly `rel_13w z_13w`.

**Done.**
- [ ] Six panels stack, one per row, with linked x ranges.
- [ ] A short history shows `Insufficient data: needs N, has M`, never a partial z.
- [ ] Changing chart history does not reset the chosen ETF, horizon or window.
- [ ] The cross-section honours the group filter.

**Verify.** Standard verify block; headless screenshot of `#tab=lab&fund=SPY`.

**Out of scope.** Backtests, trading signals.

---

## Slice 13: Parity (13a to 13h)

Slice 13 completes the dashboard: every control, panel and metric that slices 1 to 12 leave out, specified item by item in `docs/reference/PARITY.md` (its coverage table lists every feature item with the slice that builds it). It has eight sub-slices, each one session of similar size. **Build order: 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h** (letters are names, not the order): 13e publishes the residual structure, factor fits and computed clusters that the later ones read. Each sub-slice adds at most 3 tests in `tests/test_slice13<letter>.py`. Every step names the PARITY item that is its exact spec; the new `data.js` fields are already listed in `DATA.md` section 9 with their sub-slice, and the new constants in section 12.

| Sub-slice | Tabs | Feature items |
|---|---|---|
| 13a | shared data, metrics and helpers | 11 |
| 13e | Correlation, Structure & PCA | 19 |
| 13b | Overview, Heatmap | 21 |
| 13c | Leadership, Rotation | 21 |
| 13d | Chart wall, Groups | 21 |
| 13f | Risk & return, Screener | 19 |
| 13g | Market, Fund, header and settings | 26 |
| 13h | Signal lab, Calendar, Data, review mode | 24 |

---

## Slice 13a: Shared data, metrics and helpers

**Goal.** The data and browser helpers that several parity sub-slices read: factor settings, weekly cash returns, the new per-ETF metrics, relative returns for every period, the 1W period, group-index and benchmark metrics, co-movement series, the metric registry, rolling-statistic helpers and the line-chart upgrades.

**Depends on.** Slice 10 (so every slice 1 to 12).

**Read first.** `docs/reference/PARITY.md` sections 0 and 13a; `docs/reference/DATA.md` sections 2b, 3, 4, 5, 9, 12 and 15; `docs/reference/METRICS.md` sections 1, 3, 4, 5, 6 and 7.2; `docs/reference/UI.md` sections 4, 8, 9 and 11.

**Files to create or change.** `etf_arcade/project.py`, `etf_arcade/prices.py`, `etf_arcade/sample.py`, `etf_arcade/fetch.py`, `etf_arcade/cli.py`, `etf_arcade/constants.py`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `settings.toml`, `examples/europe/settings.toml`, `web/app.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice13a.py`.

**Steps.**
1. `[factors]` in `project.load` (allowed sections, keys, types), factor tickers as extra tickers of `prices.read`, `sample` and `fetch`, `params.factors`, and the two shipped `settings.toml` files (PARITY 13a.1).
2. `_cash.js` gains `wdates` and `cash_w` (PARITY 13a.2).
3. Per-ETF `r_1w r_5y_ann r_10y_ann years vol_3y sortino_3y hit_1y` in `metrics.FIELDS` and `corr_3y` in `metrics.REF_FIELDS` (PARITY 13a.3).
4. `rel_1w rel_1m rel_6m rel_ytd rel_3y_ann` in `metrics.REF_FIELDS` (PARITY 13a.4).
5. The 1W return period in the settings strip and the nine period keys of `group_return` and `benchmarks[].returns` (PARITY 13a.5).
6. `groups[].index_metrics` and `index_why` from the same metric functions on `group_index` cut at `C`, with the YTD rule (PARITY 13a.6).
7. Group medians and shares from member values at `C`, and `groups[].med_z_13w` (PARITY 13a.7).
8. `benchmarks[].returns` at `C`, `benchmarks[].metrics` and `why` (PARITY 13a.8).
9. `pca.pca_rolling` `avg_corr within across corr_ref` (PARITY 13a.9).
10. `ARCADE.METRICS` registry; the drawer lists its metrics from it (PARITY 13a.10).
11. `ARCADE.weekly`, `ARCADE.wret`, `ARCADE.roll` and `params.roll` (PARITY 13a.11).
12. `ARCADE.lineChart`: end labels with values, line and label hover highlight, crosshair cap `TIP_MAX_LINES` (PARITY 13a.12).
13. `ARCADE.series` loading and failure states (PARITY 13a.13).
14. Tests: (a) `cash_w` equals the weekly cash return of `sharpe_3y` and a hole of more than 7 days nulls its Fridays (PARITY 13a.2); (b) the per-ETF nulls of PARITY 13a.3 (`vol_3y`, `r_5y_ann`, `sortino_3y`, `r_1w`); (c) group values at `C` (PARITY 13a.6 and 13a.7: a member's later closes do not move `med_r_3m` or `group_return`, `index_metrics.r_3m == group_return.r_3m`, a January `as_of` with a December `C` nulls `r_ytd`).

**data.js additions.** Per fund `r_1w r_5y_ann r_10y_ann years vol_3y sortino_3y hit_1y` and (objects keyed by benchmark id) `corr_3y rel_1w rel_1m rel_6m rel_ytd rel_3y_ann`. `groups[]`: `group_return` keys `r_1w r_5y_ann r_10y_ann`, `med_z_13w index_metrics index_why`. `benchmarks[]`: `returns` at `C` with nine keys, `metrics`, `why`. `pca.pca_rolling`: `avg_corr within across corr_ref`. `params`: `roll`, `factors`. Series file `_cash.js`: `wdates cash_w`. `why` codes `factors_missing`, `cutoff_prior_year`.

**Done.**
- [ ] Without `[factors]` the build succeeds and `params.factors` is null; a malformed `[factors]` exits 2 with `data.js` byte-identical.
- [ ] Period 1W works on every built tab; the Screener's first return column is `1W`.
- [ ] `index_metrics` equals `group_return` for every period id; `benchmarks[i].metrics.beta_3y[its own id]` is 1.
- [ ] The drawer lists every catalogue metric the data holds with its registry label and null reason.
- [ ] Line charts show `<label> <value>` end labels and fade other series when a line or label is hovered; the Overview group-index tooltip with 59 groups shows 16 lines and `+43 more`.
- [ ] A renamed series file shows the failure text on the Fund tab while the rest renders.
- [ ] `python -m unittest -q` passes (3 new tests).

**Verify.** Standard verify block; `python -m etf_arcade --project examples/europe sample --force && python -m etf_arcade --project examples/europe build`, then restore the default project; headless dumps of `#tab=overview&check=1` and `#tab=fund&fund=<ticker>&check=1` with `overlaps 0, clipped 0`.

**Out of scope.** Factor regressions and residual structure (13e); new panels on any tab.

---

## Slice 13e: Correlation and Structure & PCA (parity)

**Goal.** Equity-factor residuals and factor fits, residual PCA, the residual map with computed clusters, and the missing Correlation and Structure & PCA panels.

**Depends on.** Slice 13a.

**Read first.** `docs/reference/PARITY.md` sections 0 and 13e; `docs/PRODUCT.md` sections 4.7 and 4.8; `docs/reference/METRICS.md` sections 7.1 and 7.2; `docs/reference/UI.md` sections 6, 8 and 13.5; `docs/reference/DATA.md` sections 9 (`correlation`, `pca`) and 12.

**Files to create or change.** `etf_arcade/constants.py`, `etf_arcade/metrics.py`, `etf_arcade/build.py`, `web/tabs/correlation.js`, `web/tabs/pca.js`, `web/app.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice13e.py`.

**Steps.**
1. Add `RESID_MIN_WEEKS PCA_AR_COMPONENTS FACTOR_SHORT_MIN HAC_LAGS T_CRIT` to `constants.py`; publish `params.t_crit` (PARITY 13e.1 to 13e.4, DATA section 12).
2. Equity-factor residuals (PARITY 13e.1).
3. Factor fits: per-ETF `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y alpha_full alpha_t_full` in `metrics.FIELDS` (so they also reach `index_metrics` and `benchmarks[].metrics`), and `groups[].factor_fit` (PARITY 13e.2).
4. Residual PCA, `ar5`, `pc1_resid`, `rpc1`, `rpc2` (PARITY 13e.3).
5. Classical MDS, `mds_x`, `mds_y`, average-linkage clusters and their names, `pca.mds`, `pca.clusters` (PARITY 13e.4).
6. `correlation.corr_3y_residual` and `correlation.groups` (PARITY 13e.5, 13e.8).
7. `correlation.js`: residual window, ETF select, pairs mode with group columns, Group correlation, Correlation regime, Rolling correlation to the reference, layout (PARITY 13e.5 to 13e.11).
8. `pca.js`: absorption and ex-market lines, raw and residual scree, colour by, outlines, shared legend, Residual map, loadings mode, Alpha forest, Alpha against residual risk with its table, Group vs computed cluster, layout and filter rules (PARITY 13e.12 to 13e.20).
9. Tests: (a) the factor fit of PARITY 13e.2 (betas, alpha, `cash_incomplete`, `factors_missing`); (b) residual PCA `insufficient_data` (PARITY 13e.3); (c) two-block clusters and their names (PARITY 13e.4).

**data.js additions.** Per fund `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y alpha_full alpha_t_full rpc1 rpc2 mds_x mds_y cluster`. `groups[]`: `factor_fit`. `pca`: `pca_resid_status pca_resid_n pca_resid_fixed mds clusters`; `pca.pca_rolling`: `ar5 pc1_resid`. `correlation`: `corr_3y_residual groups`. `params`: `t_crit`.

**Done.**
- [ ] The 3Y residual window renders with its own order and is disabled without `[factors]`.
- [ ] Across groups lists no same-group pair; the ETF select drives the peer bars.
- [ ] Group correlation drops hidden groups and keeps the benchmark row; Correlation regime breaks where fewer than 8 ETFs are complete and shows the full-universe note under a group filter.
- [ ] The scree shows raw and residual bars with separate counts; without `[factors]` the residual side, the residual map and the alpha panels say `No factor series is configured`.
- [ ] Colour by Computed cluster recolours both scatters and the legend; outlines toggle off.
- [ ] Alpha forest markers are filled only when `|t| >= 1.96`; 12M and YTD rows show their row count.
- [ ] Under a group filter the map, residual scree and rolling lines show the full-universe note; the alpha panels drop hidden groups.
- [ ] `python -m unittest -q` passes (3 new tests).

**Verify.** Standard verify block; headless dumps of `#tab=correlation&check=1` and `#tab=pca&check=1`, once with a group hidden and once for `examples/europe` (no `[cash]`: the alpha panels read `No cash series is configured`, the residual map is present).

**Out of scope.** Cluster options on other tabs (13b, 13c, 13d, 13f, 13g).

---

## Slice 13b: Overview and Heatmap (parity)

**Goal.** The missing Overview tiles, hovers, signal snapshot and co-movement panel, and the Heatmap's extra measures, scale, sizes, cluster grouping and performance grid.

**Depends on.** Slices 13a and 13e.

**Read first.** `docs/reference/PARITY.md` sections 0 and 13b; `docs/PRODUCT.md` sections 4.1 and 4.2; `docs/reference/UI.md` sections 6, 8, 9 and 13.1.

**Files to create or change.** `web/tabs/overview.js`, `web/tabs/heatmap.js`, `web/app.js` (treemap hover and value line), `web/app.css`.

**Steps.**
1. KPI strip: reference, ahead of the reference, above 200-day, average correlation, filter rules (PARITY 13b.1).
2. ETF hover card on both treemaps (PARITY 13b.2) and leaderboard hover (PARITY 13b.3).
3. Signal snapshot (PARITY 13b.4) and How much moves together (PARITY 13b.5).
4. Top/Bottom z column (PARITY 13b.6) and the group-index legend (PARITY 13b.7).
5. Heatmap Colour by, Colour scale, Size by, Group by cluster (PARITY 13b.8 to 13b.11).
6. Tile value line, block header and title (PARITY 13b.12, 13b.13).
7. Groups across periods 1W, 5Y, 10Y (PARITY 13b.14).
8. Performance grid with Keep groups together and Filter (PARITY 13b.15).

**data.js additions.** None (reads 13a and 13e fields).

**Done.**
- [ ] With all groups visible the ten tiles, the snapshot and the co-movement panel show values; hiding one group turns the breadth and correlation tiles and the co-movement panel into the full-universe note.
- [ ] Every new Colour by, Size by and Group by option redraws the treemap only; Tight and Wide visibly change saturation.
- [ ] The grid shows every visible ETF in group blocks, filters as you type, and its sort follows the period until a header is clicked.
- [ ] `check=1` on both tabs reports `overlaps 0, clipped 0` at 1400 and 390 px.

**Verify.** Standard verify block; headless screenshots of `#tab=overview` and `#tab=heatmap` at 1400 and 390 px, once with a group hidden.

**Out of scope.** The market brief (13g).

---

## Slice 13c: Leadership and Rotation (parity)

**Goal.** Leadership scatter controls and legend, richer early-warning cards, the full participation panel; Rotation's stale heads, ETF rotation, period-following ranks and the monthly group heat.

**Depends on.** Slices 13a and 13e.

**Read first.** `docs/reference/PARITY.md` sections 0 and 13c; `docs/PRODUCT.md` sections 4.3 and 4.6; `docs/reference/METRICS.md` sections 6, 7.3, 7.4 and 11; `docs/reference/UI.md` sections 8 and 13.3; `docs/reference/DATA.md` sections 9 (`market`, `rotation`) and 12.

**Files to create or change.** `etf_arcade/constants.py` (`EW_BACK_LONG` 63, `EW_BACK_SHORT` 21), `etf_arcade/metrics.py`, `etf_arcade/build.py`, `web/tabs/leaders.js`, `web/tabs/rotation.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice13c.py`.

**Steps.**
1. Scatter: Labels, Group indices rings at the index position, Colour by, legend (PARITY 13c.1); label priority (PARITY 13c.2); hover (PARITY 13c.3).
2. `ew_dir ew_days ew_rel_3m` and the 63 / 21 row offsets (PARITY 13c.4).
3. Early-warning cards: sort, arrow, streak, dots, 3M relative, groups without usable history (PARITY 13c.5).
4. `market.ew_index`, `market.groups_above_200d` and the two linked participation charts with their legend (PARITY 13c.6).
5. Rotation: stale heads and head click (PARITY 13c.7), ETF rotation inside a group (PARITY 13c.8).
6. `rotation.rank_trail.by_period` and period-following ranks (PARITY 13c.9).
7. Monthly returns by group (PARITY 13c.10).
8. Tests: (a) `ew_days` sign and streak end (PARITY 13c.4); (b) `ew_index` and `groups_above_200d` (PARITY 13c.6).

**data.js additions.** `groups[]`: `ew_dir ew_days ew_rel_3m`. `market`: `ew_index groups_above_200d`. `rotation.rank_trail.by_period`.

**Done.**
- [ ] Labels All/Some/None, the ring toggle, Colour by and the legend change the scatter without a reload and survive a history change.
- [ ] Early-warning cards sort three ways and show arrow, streak, dots and 3M relative; groups without a ratio are listed once.
- [ ] The participation panel shows both charts on one x axis with the group line and the latest-value legend, and the full-universe note under a group filter.
- [ ] Clicking a group head switches the inside-group panel; switching the period to 1Y re-ranks the bump chart and table without resetting tail or end week; a missing month in the monthly heat is blank.
- [ ] `python -m unittest -q` passes (2 new tests).

**Verify.** Standard verify block; headless dumps of `#tab=leaders&check=1` and `#tab=rotation&check=1`.

**Out of scope.** The Market card that reuses `ew_index` (13g).

---

## Slice 13d: Chart wall and Groups (parity)

**Goal.** The wall's twelve chart kinds, grouping, ordering, scales, compare line, search, group cards, summary and collapse; the Groups log toggle, rolling statistic panel, z heat and four statistics sets.

**Depends on.** Slices 13a and 13e.

**Read first.** `docs/reference/PARITY.md` sections 0, 13a.11 and 13d; `docs/PRODUCT.md` sections 4.4 and 4.5; `docs/reference/METRICS.md` sections 8 and 9; `docs/reference/UI.md` sections 8, 9 and 13.2.

**Files to create or change.** `web/tabs/wall.js`, `web/tabs/groups.js`, `web/app.js`, `web/app.css`.

**Steps.**
1. Wall chart kinds with their reference lines (PARITY 13d.1).
2. Group by and Order groups by (PARITY 13d.2); Y scale and aligned dates (PARITY 13d.3).
3. Compare with (PARITY 13d.4), Search (PARITY 13d.5), Group index first (PARITY 13d.6).
4. Summary line, section header average, collapse (PARITY 13d.7); card header (PARITY 13d.8).
5. Groups: log toggle (PARITY 13d.9), rolling statistic (PARITY 13d.10), excess z heat (PARITY 13d.11).
6. Groups statistics sets Returns, Risk, Alpha, Breadth (PARITY 13d.12).

**data.js additions.** None (reads 13a and 13e fields).

**Done.**
- [ ] All twelve kinds draw on every card with their reference lines; Free, Per group and Shared, Compare with, Group by None, Order groups by, Search, Group index first, collapse and the summary line all work and survive a history or benchmark change; 520 cards stay lazy.
- [ ] Rolling Sharpe without `[cash]` shows `Not available` with the reason.
- [ ] The Groups log checkbox, the seven rolling kinds (nulls break the line) and the z heat (changes with the horizon) work; each statistics set shows its columns, and the Alpha set reads `—` with the reason without `[factors]`.
- [ ] `check=1` on both tabs reports `overlaps 0, clipped 0`.

**Verify.** Standard verify block; headless screenshots of `#tab=wall` and `#tab=groups` at 1400 and 390 px.

**Out of scope.** Python changes (none needed).

---

## Slice 13f: Risk & return and Screener (parity)

**Goal.** Risk & return gets the full metric catalogue, colour and size modes, label mode, legend, group rings, benchmark marks and richer hovers; the Screener gets column sets, heat cells, group rows, sparklines, the cluster column and header hovers.

**Depends on.** Slices 13a and 13e.

**Read first.** `docs/reference/PARITY.md` sections 0, 13a.10 and 13f; `docs/PRODUCT.md` sections 4.9 and 4.11; `docs/reference/UI.md` sections 8, 9, 10, 13.4 and 15.6.

**Files to create or change.** `web/tabs/risk.js`, `web/tabs/screener.js`, `web/app.js`, `web/app.css`.

**Steps.**
1. Risk: axis catalogue and title (PARITY 13f.1), Colour by (PARITY 13f.2), Dot size (PARITY 13f.3), label mode and legend (PARITY 13f.4).
2. Group-index rings (PARITY 13f.5), reference crosshair and strip line (PARITY 13f.6), hovers (PARITY 13f.7), capture parity lines (PARITY 13f.8).
3. Strip metric (PARITY 13f.9) and drawdown watch heat (PARITY 13f.10).
4. Screener: column sets (PARITY 13f.11), colour cells (PARITY 13f.12), group-index rows (PARITY 13f.13), sparkline (PARITY 13f.14), cluster column (PARITY 13f.15), header hover (PARITY 13f.16).

**data.js additions.** None (reads 13a and 13e fields).

**Done.**
- [ ] Choosing `sortino_3y` on Y titles the scatter `Sortino 3Y vs Volatility 1Y`, and a history change keeps axes, strip metric, colour, size and label mode.
- [ ] Rings sit at their `index_metrics`; switching the benchmark moves the crosshair lines and the diamond; the capture scatter shows three dashed lines meeting at (1, 1).
- [ ] Screener sets show their columns, the sort resets only on a set change, group rows interleave when sorting, sparklines load lazily and redraw on a history change.
- [ ] `check=1` on both tabs reports `overlaps 0, clipped 0`.

**Verify.** Standard verify block; headless screenshots of `#tab=risk` and `#tab=screener` at 1400 and 390 px.

**Out of scope.** Python changes (none needed).

---

## Slice 13g: Market, Fund and global shell (parity)

**Goal.** A fuller as-of line, the market brief, 10Y history, the collapsed settings summary and tab-change scrolling; the missing Market cards and rules; group indices and benchmarks as Fund subjects with the missing Fund panels, pills and tiles.

**Depends on.** Slices 13a, 13e (`funds[].cluster`) and 13c (`market.ew_index`).

**Read first.** `docs/reference/PARITY.md` sections 0 and 13g; `docs/PRODUCT.md` sections 2, 3, 4.13 and 4.14; `docs/reference/METRICS.md` section 11; `docs/reference/UI.md` sections 4, 5, 6, 12 and 13 items 13 and 14; `docs/reference/DATA.md` sections 9 (`market`) and 11.

**Files to create or change.** `etf_arcade/metrics.py`, `etf_arcade/build.py`, `web/app.js`, `web/tabs/market.js`, `web/tabs/fund.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice13g.py`.

**Steps.**
1. As-of line (PARITY 13g.1) and market brief (PARITY 13g.2) in `app.js`.
2. 10Y chart history (PARITY 13g.3), collapsed settings summary (PARITY 13g.4), tab-change scroll (PARITY 13g.5).
3. Market: large-move note (PARITY 13g.6), dispersion scale (PARITY 13g.7), Equal weight vs the reference (PARITY 13g.8), `market.pos_21d` and its card (PARITY 13g.9), squares hover (PARITY 13g.10), the full-universe rule for every market card (PARITY 13g.11).
4. Fund subjects `g:` and `b:` (PARITY 13g.12), flag chips (PARITY 13g.13), Trades like (PARITY 13g.14), KPI tiles (PARITY 13g.15).
5. Fund charts: group line (PARITY 13g.16), benchmark drawdown (PARITY 13g.17), rolling return and excess (PARITY 13g.18), rolling beta and correlation (PARITY 13g.19).
6. `ARCADE.monthGrid` and the Fund monthly panel (PARITY 13g.20), peers columns (PARITY 13g.21), Most correlated (PARITY 13g.22).
7. Test: `pos_21d` (PARITY 13g.9).

**data.js additions.** `market.pos_21d`.

**Done.**
- [ ] The as-of line shows the benchmark, priced and stale counts and the cutoff hover; the brief follows benchmark, period and group filter and never shows a full-universe correlation for a filtered view.
- [ ] 10Y works everywhere a history applies; at 390 px the collapsed summary reads `Settings · 3M · 3Y`; a tab change starts at the top.
- [ ] Hiding one group leaves exactly the drawdown, cash and squares cards on the Market tab; Auto switches the dispersion card to Log on an extreme day.
- [ ] `#tab=fund&fund=g:<first group>` and `#tab=fund&fund=b:<first id>` draw their subjects without opening the drawer; Back returns to the previous ETF.
- [ ] The Fund tab shows flag chips, the Trades like pill, the new tiles, both rolling panels, the monthly grid, the peers columns and 14 most-correlated bars.
- [ ] `python -m unittest -q` passes (1 new test).

**Verify.** Standard verify block; headless dumps of `#tab=market&check=1`, `#tab=fund&fund=<ticker>&check=1` and `#tab=fund&fund=g:<group>&check=1`; a 390 px iframe check of the settings summary.

**Out of scope.** Calendar and Data changes (13h).

---

## Slice 13h: Signal lab, Calendar, Data and review mode (parity)

**Goal.** Signal lab benchmarks, comparisons, pipeline, tiles, histogram and full cross-section; Calendar history scope, hovers, headers, seasonal paths and fund-by-fund heat; Data timeline and Method content; the review strip with marker mode on every tab.

**Depends on.** Slice 13g (and so every earlier sub-slice).

**Read first.** `docs/reference/PARITY.md` sections 0 and 13h; `docs/PRODUCT.md` sections 4.10, 4.12, 4.15 and 4.16; `docs/reference/METRICS.md` sections 7.5 and 8; `docs/reference/UI.md` sections 13.3, 13.6 and 14; `docs/reference/DATA.md` sections 9 (`params`), 11 and 14.

**Files to create or change.** `etf_arcade/constants.py` (`SEASON_PATH_MIN_CLOSES`), `etf_arcade/build.py` (`params.constants`), `web/tabs/lab.js`, `web/tabs/calendar.js`, `web/tabs/data.js`, `web/review.js`, `web/app.js`, `web/app.css`, `docs/reference/DATA.md`, `tests/test_slice13h.py`.

**Steps.**
1. Signal lab: benchmarks as series (PARITY 13h.1), Excess over (PARITY 13h.2), pipeline strip (PARITY 13h.3), stats tiles (PARITY 13h.4), histogram (PARITY 13h.5), percentile minimum (PARITY 13h.6), cross-section modes (PARITY 13h.7).
2. Calendar: history scope with `ARCADE.monthGrid` (PARITY 13h.8), seasonality hover (PARITY 13h.9), year-table headers (PARITY 13h.10), seasonal paths (PARITY 13h.11), fund-by-fund heat (PARITY 13h.12).
3. Data: timeline detail (PARITY 13h.13), `params.constants` and the Method content (PARITY 13h.14).
4. Review strip: marker, note, suspects only, discard pending, badges (PARITY 13h.15).
5. Test: `params.constants` reflects a `[constants]` override (PARITY 13h.14).

**data.js additions.** `params.constants`.

**Done.**
- [ ] `#tab=lab&series=b:<first id>` draws six panels; Own group gives different z values from Reference; the pipeline, tiles and histogram follow the settings; the cross-section has both modes and a recompute button for custom settings.
- [ ] History 3Y limits the Calendar to 4 years; seasonal paths and the fund-by-fund heat follow their controls and keep them across history and benchmark changes.
- [ ] The Data timeline is grouped and clipped to the history; Method shows live numbers, including a changed `STALE_DAYS`.
- [ ] With marker Exclude a Screener row click creates one pending mark without opening the drawer; Suspects only leaves exactly the flagged and marked ETFs; badges show on Screener rows, wall cards and treemap tiles.
- [ ] `python -m unittest -q` passes (1 new test); every tab passes `check=1`.

**Verify.** Standard verify block; headless dumps of `#tab=lab&check=1`, `#tab=calendar&check=1` and `#tab=data&check=1`; review mode exercised as in slice 12's Verify.

**Out of scope.** A separate review page or any server endpoint.
