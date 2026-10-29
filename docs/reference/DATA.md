# etf-arcade data reference

Lookup reference for the data side: `<project>/universe.csv` + `<project>/var/prices.csv` + `<project>/settings.toml` -> `etf_arcade` Python package -> `web/data.js`.
The browser never reads CSV; `web/data.js` is the only data input of the UI and must work from `file://`.
`STATUS.md` says which slices are built. Slice numbers refer to `docs/BUILD_SLICES.md` and say which slice builds an item (build order 1, 1b, 2, 3, 11, 12, 4 to 10). Sections 1 to 8 and 15 are the exact build spec of slices 1 and 1b.

## 1. Data flow and ownership

```
<project>/universe.csv (committed) ---\
<project>/settings.toml -----------------> python -m etf_arcade build --> web/data.js (generated, gitignored)
<project>/var/prices.csv (gitignored) -/
        ^
        |-- python -m etf_arcade sample   (SYNTHETIC, offline, deterministic)
        '-- python -m etf_arcade fetch    (yfinance adjusted closes, optional extra)
```

| Artifact | Writer | Reader | Committed |
|---|---|---|---|
| `<project>/universe.csv` (the repository root project keeps it in `data/universe.csv`) | humans | `universe.py` | yes |
| `<project>/settings.toml` | humans | `project.py` | yes |
| `<project>/var/prices.csv` | `sample` or `fetch` (or any other producer) | `prices.py` | no |
| `web/data.js`, `web/project.css`, `web/series/` | `build` | `web/app.js` and tabs | no |

Rules:
- The project folder is selected with `--project DIR` (section 2b); the default `DIR` is the repository root. `DIR` may be any path, including one outside the repository (a private project folder); the output is always `web/data.js` in the current directory.
- Import rules: `constants.py` imports nothing from the package. `metrics.py` imports only `constants.py`. `universe.py` imports `constants.py`; `prices.py` imports nothing from the package; `project.py` imports `constants.py` and `universe.py`. `build.py` imports `universe`, `prices`, `project`, `metrics`, `constants` and, once they exist, `quality.py` (slice 11) and `review.py` (slice 12). `quality.py` imports only `constants.py`; `review.py` imports only the standard library and `constants.py`. `metrics.py` never imports `quality.py` or `review.py`. Nothing but `cli.py` imports `fetch.py` or `sample.py`. `prices.csv` is the only price input contract.
- `fetch.py` is the only module that imports `yfinance`, inside the function. Without yfinance installed, `fetch` exits 2 with an install hint; `sample` and `build` still work.
- Python >= 3.11 (`tomllib` reads `settings.toml`); hard dependencies are `pandas` and `numpy` only.

## 2. `universe.csv`

UTF-8, header row exactly: `ticker,name,short_name,group,family,currency`. The default example `data/universe.csv` lists 520 US-listed ETFs in 59 groups and 5 families. Benchmarks and cash are not in this file (section 2b).

| Column | Type | Rule |
|---|---|---|
| `ticker` | string | Stable id, regex `[A-Za-z0-9^][A-Za-z0-9._^=-]{0,31}` (`universe.TICKER`; allows Yahoo suffixes such as `CSPX.DE`, index symbols such as `^IRX` and private fund ids). Unique. Share-class tickers use a hyphen (`BRK-B` style), matching the fetch provider. The same regex checks benchmark component and cash tickers in `settings.toml`. |
| `name` | string | Full legal-style name. Non-empty. Shown in hover/detail only. |
| `short_name` | string | Non-empty, <= `SHORT_NAME_MAX` (24) characters, no ellipsis. Default display name everywhere. Duplicates are reported as a warning. |
| `group` | string | Non-empty taxonomy group. Each group belongs to exactly one family. |
| `family` | string | Non-empty. Parent of the group. |
| `currency` | string | `[A-Z]{3}`, the listing currency, a label only: no FX conversion exists, so closes and returns stay in each fund's own currency. Any other value is a validation error naming the ticker. |

Validation (`universe.load(path)` returns a DataFrame of the six columns, all strings, and raises `ValueError("universe row <n>, ticker '<t>': <problem>")` for the first offending row; nothing is partially loaded):
- the header is not exactly the six columns, or there are zero data rows (`ValueError` naming the columns, or `universe has no data rows`);
- cells are stripped of leading/trailing whitespace first; rows that are blank in every column are skipped; the reported row number is the data row's position among the remaining non-blank rows plus 2 (the header is line 1), so a file without blank lines gives the real file line;
- checks per row, first match wins, with these problem texts: `bad ticker` (regex), `duplicate ticker`, `empty cell` (any column), `short_name longer than 24 characters`, `currency must be a 3-letter code`, `group '<g>' belongs to two families`.

Derived at load time:
- `group order` = order of first appearance in the file. It fixes the colour: `index % 12 + 1` selects `--g1`..`--g12` from `theme.css`. With ~60 groups colours repeat; the group name is always shown beside the colour, never colour alone.
- `family order` likewise.

### 2a. `data/universe_ucits.csv` (optional European mirror, not read by any slice)

One row per US ticker in `universe.csv`: `us_ticker, match, ucits_name, isin, yahoo_ticker, exchange, currency, index_or_exposure, figi, yahoo_first, note`.
- `match`: `same_index` (same index family and method), `similar` (same exposure, different index; the difference is in `note`), `none` (no UCITS equivalent; other columns blank).
- Listing preference: Xetra (`.DE`, EUR), then Amsterdam, Milan, Paris, Zurich, London. Accumulating, larger, older share classes are preferred.
- Every non-`none` row was checked: the ISIN resolves to an ETP on OpenFIGI, and `yahoo_ticker` has price history starting `yahoo_first`.
- Several US ETFs may map to the same UCITS ETF. Facts only (names, identifiers): re-check before relying on a row.
- `examples/europe/` is built from this file (section 2b).

### 2b. Project folder (slice 1b)

`python -m etf_arcade [--project DIR] {sample|fetch|build|review-status}` (`review-status` arrives with slice 12). Default `DIR` is the current directory (the repository root: its list stays in `data/universe.csv`). Every command honours `--project`; the output is always `web/data.js` (one active project at a time), so run from the repository root.

```
DIR/universe.csv      same columns as data/universe.csv (data/universe.csv if DIR has no universe.csv)
DIR/var/prices.csv    section 3, written by any producer (ignored by git)
DIR/settings.toml     optional
DIR/theme.css         optional: overrides token values only (same names as web/theme.css)
DIR/review/*.json     optional review marks (slice 12, section 14)
```

`settings.toml` (tomllib; unknown section or key, or a wrong type -> `build` exits 2 with the old `data.js` kept):

| Section | Content |
|---|---|
| `[project]` | `title` (default `ETF ARCADE`), `subtitle` (default empty), `currency` (default `USD`; label only, e.g. `EUR`), `universe_label` (default `ETFs`; the plural noun the UI uses wherever it names the funds: "N <label>", "<shown> of <total> <label>"). All strings; `title` non-empty; `currency` three capital letters. |
| `[constants]` | Any name from `etf_arcade/constants.py` (section 12), overriding it. Unknown name or wrong type (an int is accepted for a float; a bool is never an int) -> exit 2. |
| `[[benchmarks]]` | An ordered list, the first entry is the default benchmark. Each: `id` (`[A-Za-z0-9][A-Za-z0-9_-]{0,31}`, unique), `label` (non-empty, shown in the UI), `components = { "TICKER" = weight }` (tickers match `universe.TICKER`, weights numbers > 0 summing to 1 within 1e-9). A one-component benchmark is that ticker alone; several components make a weekly-rebalanced blend (METRICS section 4). Exactly the keys `id label components`. Shape is checked in slice 1b, consumed from slice 4. |
| `[cash]` | `ticker`, `label` (the name the UI shows, for example `T-bill proxy`; nothing in the UI hard-codes a cash name), `kind = "annual_yield_pct"` (the only kind: the series holds an annual yield in percent). Exactly these keys; an empty table counts as absent. Never defaulted: no `[cash]` means cash-dependent metrics are null with reason `cash_missing`, never zero-filled. Shape checked in slice 1b, consumed from slice 4. |
| `[factors]` | Slice 13a, optional: `equity` and `bond`, each a ticker (`universe.TICKER`) whose series is in `prices.csv` (a universe member, a benchmark component or an extra ticker; not the cash ticker). Exactly these two keys, both required when the table is present; an empty table counts as absent. Never defaulted: without it every factor output is null with reason `factors_missing`. Used by the factor fits and the equity-factor residuals (`docs/reference/PARITY.md` 13a.1, 13e). |

Benchmark and cash tickers are series in `prices.csv`. A benchmark component may also be a universe member: the series is then shared and nothing treats it specially. The cash ticker must not be a universe member (slice 4 checks this at build). `build` writes `web/project.css` (gitignored) as `DIR/theme.css` or the comment `/* no project theme */`; `index.html` links `theme.css` then `project.css`. `data.js` gains `project`: `{title, subtitle, currency, universe_label, dir_name}` (`dir_name` is the project folder's name; `default` for the repository-root project and when `build` runs without a project object).

Review marks (slice 12): `DIR/review/*.json`, specified in section 14. `review-status` prints them.

## 3. `<project>/var/prices.csv`

Long format, one row per ticker per trading date, adjusted closes only.

```
# source=synthetic
date,ticker,adj_close
2024-01-02,AAA,101.2345
```

| Field | Rule |
|---|---|
| Line 1 | `# source=<label>`, label free text `[a-z0-9_]+` (`yfinance`, `synthetic`, `csv_import`, ...); any producer may write the file. Missing or malformed -> `ValueError`. `sample` refuses to overwrite a file whose label is not `synthetic` unless `--force`. |
| `date` | ISO `YYYY-MM-DD` (exactly 10 characters), not in the future (local date). Exchange-local calendar date, no time zone. |
| `ticker` | Must be a universe ticker (from slice 4 also a benchmark component or the cash ticker named in `settings.toml`, from slice 13a a `[factors]` ticker). Other tickers are dropped and counted in the issues list. |
| `adj_close` | Finite float > 0, written with `repr`-precision. Split- and distribution-adjusted close (yfinance `auto_adjust=True`). Raw unadjusted closes are never stored. The one exception (slice 4): the cash ticker's value is an annual yield in percent and may be zero or negative; it only has to be finite. |
| Order | Sorted by `ticker`, then `date`. Writers sort; readers do not rely on it. |

Reader behaviour (`prices.read(path, universe) -> (frame, issues)`; slice 4 adds the extra tickers and the cash exemption as optional arguments):
- Columns after line 1 must be exactly `date,ticker,adj_close`, else `ValueError`.
- A bad row is dropped and added to `issues` as `(file line number, reason)`; it is never repaired. Reasons, applied in this order so a later one overrides an earlier one: `adj_close not a finite number > 0`; `date not YYYY-MM-DD` (unparseable or not 10 characters); `date in the future`; `ticker not in the universe` (only if no earlier reason applied). File line numbers count the `# source` line as 1 and the header as 2, so the first data row is line 3.
- Duplicate `(ticker, date)` among the remaining rows: all copies are dropped (the date becomes a gap) and each is recorded as `duplicate (ticker, date)`. No "last wins" guessing.
- Missing days are simply absent rows. No blank placeholder rows, no forward fill, no interpolation, anywhere.
- Tickers with no rows are legal and become `status: "no_data"`.
- The frame has columns `date` (datetime), `ticker`, `adj_close` (float), sorted by ticker then date, index reset.
- `prices.source(path)` returns the label (the `source` and `synthetic` fields of `data.js` come from it).

Writer behaviour (`prices.write(path, frame, source)`): label must match `[a-z0-9_]+`; create the folder; write `<path>.tmp` in the same directory (UTF-8, `\n` line endings: `# source=<label>` line, header, rows sorted by ticker then date, dates as `YYYY-MM-DD`), flush, then `prices.replace(tmp, path)` (`os.replace`, retried up to 5 times 0.2 s apart on `PermissionError`, because Windows can lock the target); the tmp file is removed on any failure.

## 4. `sample` (SYNTHETIC)

Offline generator for demos and tests: `sample.make_sample(tickers, groups, end=None, years=None)` returns `DataFrame[date, ticker, adj_close]` (`groups[i]` is the group of `tickers[i]`; `end` and `years` default to `END_DATE` and `YEARS`, read at call time so `settings.toml` overrides apply). Everything it produces is labelled SYNTHETIC: header `# source=synthetic`, `data.js` `synthetic: true`, and the source name in the header.

| Property | Rule |
|---|---|
| Determinism | Fixed constants (`SEED`, `END_DATE`, a Friday, `YEARS`). Same inputs give byte-identical `prices.csv` (two runs, same hash). Every random stream is `numpy.random.default_rng([SEED, crc32(k1), crc32(k2)])` with `crc32` of the UTF-8 text of each key: keys `("group", <group>)` for a group factor and `("ticker", <ticker>)` for a ticker, so adding a ticker or group does not change the others. |
| Calendar | `YEARS * BDAYS_PER_YEAR` (261) business days (Mon-Fri, `pandas.bdate_range(end=END_DATE, periods=...)`) ending at `END_DATE`. No holiday calendar. |
| Process | Group factor per group: `n` standard normals. Per ticker, drawing from its own stream in this order: `vol = uniform(0.08, 0.30) * lev / sqrt(261)`, `drift = sign * uniform(0.0, 0.10) / 261`, then `n` standard normals `z`; return `drift + vol * (0.6 * group_factor + 0.8 * z)`; start level `uniform(20, 200)`; close = `exp(log(start) + cumsum(return))` rounded to 4 decimals (stays > 0). `lev` is 2.5 for groups starting with `Leveraged` else 1; `sign` is -1 for groups starting with `Inverse` else 1. |
| Short history | About 1 in 25 tickers (`crc32("short:" + ticker) % 25 == 0`) keeps only its last `integers(130, 653)` business days (6 to 30 months; the draw comes after the start level) to exercise 1Y/3Y nulls. Committed universe: 27 tickers, 8 of them under 12 months (status `short`). |
| Gap | Exactly one ticker is missing `GAP_BDAYS = 60` consecutive business days: the gap starts at row `n - GAP_FROM_END` (127), moved back to the Monday on or before it (inside the last year, whole Mon-Fri weeks, 12 weekly buckets emptied). Its `vol_1y` is null, its returns still compute. |
| Stale | Exactly one ticker loses its last `STALE_BDAYS = 20` business days (28 calendar days): status `stale`. |
| No data | Exactly one ticker has no rows: status `no_data`. |
| Special tickers | The gap, stale and no-data tickers are the first three tickers ordered by `crc32("special:" + ticker)` (stable sort), in that order: first the gap ticker, second the stale ticker, third the no-data ticker, so the pick does not depend on file order. They are never also short-history. With the first 40 universe rows there is exactly one of each. |
| Slice 4 additions | `make_sample` also takes `extra` (benchmark tickers, and from slice 13a the `[factors]` tickers, that are not universe members: plain walks using group key `"_benchmark"`, never special or short) and `cash` (a ticker: annual yield in percent, `round(2.0 + cumsum(normal(0, 0.02, n)), 4)` from stream `("cash", ticker)`, may go below zero). The special picks stay among the universe tickers only and, from slice 4, never fall on a benchmark component, the cash ticker or (slice 13a) a `[factors]` ticker, so a gap or spike never removes a reference series. |
| Safety | `sample` refuses to overwrite a `prices.csv` whose header names another source (for example `yfinance`) unless `--force` is passed (exit 2 otherwise). |

For the committed `data/universe.csv` (520 tickers) the result has 779,599 rows from 2019-12-27 to 2025-12-26 and statuses: 510 `ok`, 8 `short`, 1 `no_data`, 1 `stale`. Use these as a sanity check of the build, not as a test.

## 5. `fetch`

`fetch.fetch_prices(tickers, years=FETCH_YEARS)` returns `(frame[date, ticker, adj_close], failed)`, `failed` being the requested tickers with no rows.

| Property | Rule |
|---|---|
| Source | yfinance, `yf.download(chunk, start=<start>, auto_adjust=True, progress=False, group_by="column")`, the `Close` column (a single ticker comes back as a Series: wrap it). Never `Adj Close` of an unadjusted download, never raw close. |
| Window | `FETCH_YEARS = 6` back from today (`start` = today minus `years` as `YYYY-MM-DD`; `--years N` overrides). Full window every run: adjusted history is restated whenever a distribution is paid, so appending to old rows would mix two adjustment bases. |
| Batching | `FETCH_BATCH = 100` tickers per `yf.download` call, one call after another, no pause and no retry. A batch that returns nothing adds no rows. NaN closes are dropped, as are values <= 0. |
| Per-ticker failure | A requested ticker with no rows is listed in the run summary and has no rows in the new file: nothing is carried over from the old `prices.csv`, whatever its source (no splicing across sources or fetches). It shows as `no_data` until a later fetch succeeds. |
| Whole-run failure | yfinance missing (`RuntimeError('yfinance is not installed: pip install "etf-arcade[fetch]"')`), nothing returned (`no prices returned`), more than `FETCH_MAX_FAIL_FRACTION = 0.5` of the requested tickers failed (`<n> of <m> tickers failed: <first 10>`), or a `--tickers` name outside the universe (`not in universe: ...`): exit 2 and leave `prices.csv` untouched. Otherwise exit 0 with the failed tickers listed. |
| Write | Through `prices.write` (atomic), header `source=yfinance`, replacing the whole file. `--tickers A,B` fetches and writes only those tickers, so `build` then stops under `MIN_PRICED_FRACTION`; use it for a trial and run a full `fetch` before `build`. A fetch never edits `web/data.js`. |
| Currency | Closes are in the listing currency. No conversion. |
| Slice 4 additions | The request also holds every benchmark component ticker, the cash ticker and (slice 13a) the `[factors]` tickers that are not universe members. The failure fraction counts universe tickers only; a missing benchmark or cash ticker is listed in the summary. The `> 0` filter does not apply to the cash ticker. `--tickers` may name any of these tickers. |

## 6. Normalisation rules (apply in `prices.read` and `metrics.py`)

- Series are keyed by `ticker`. A ticker is one series: no merging of tickers, share classes or sources.
- One value per `(ticker, date)`; dates strictly ascending after reading.
- `as_of` (universe) = the maximum `last_date` over the universe tickers. It is data-driven, never the wall clock, so builds are reproducible.
- `days_stale` = `as_of - last_date` in calendar days. A fund is stale when `days_stale > STALE_DAYS`.
- A return is never computed across a missing observation by filling it. Anchors use the last close on or before the anchor date, within tolerance; otherwise the result is null.
- Missing metric = `null` in `data.js`, em dash in the UI, with the reason in `why` and in hover text.
- Zero cash rate is never assumed. Slice 1 has no cash-dependent metric; slice 4 adds Sharpe with the project's labelled `[cash]` series and leaves it null (`cash_missing` or `cash_incomplete`) if that series is absent or incomplete.
- Every stored number is a decimal fraction (`0.052` = 5.2%): returns, `rel_*` differences, shares such as breadth. Ratios, index levels and correlations are plain numbers. Units per field: section 9 and METRICS.md. The UI formats; the data never carries percent signs or rounded strings.

## 7. Slice-1 metrics (`metrics.py`)

Input per ticker: ascending `adj_close` series `P`. `t_end` = the ticker's own last date. All windows end at `t_end`.

Anchor rule (shared): `L0` = last close with date <= `target`; valid only if `(target - date(L0)).days <= ANCHOR_TOL_DAYS` (7). Otherwise null, reason `no_history_to_anchor`. The tolerance stops a one-day holiday offset from nulling a metric.

| Field | Definition | Null when |
|---|---|---|
| `r_1m`, `r_3m`, `r_6m`, `r_1y` | `P_end / L0 - 1`, `target = t_end - DateOffset(months=1/3/6/12)` | anchor rule fails |
| `r_ytd` | `P_end / L0 - 1`, `target` = 31 Dec of the prior year of `t_end` | anchor rule fails |
| `r_3y_ann` | `(P_end / L0)^(1/3) - 1`, `target = t_end - DateOffset(months=36)` | anchor rule fails (history < 3y) |
| `vol_1y` | Weekly: `P.resample("W-FRI").last()`, drop buckets whose Friday label is after universe `as_of` (partial week), `pct_change(fill_method=None)`, take the last `VOL_WEEKS` (52) return slots, drop the NaN slots, sample SD (`ddof=1`) x `sqrt(ANNUAL_WEEKS)` | fewer than `VOL_MIN_WEEKS = 40` finite returns among the slots; reason `insufficient_weekly_returns` |
| `maxdd_1y` | On daily closes with date >= `t_end - DateOffset(months=12)`: `min(P / cummax(P) - 1)`, value <= 0 | fewer than `DD_MIN_CLOSES = 200` closes in the window, or first close of the window is more than 7 days after the target; reason `insufficient_daily_closes` |
| `first_date` | First date with a close | no rows |
| `last_date` | `t_end` | no rows |
| `days_stale` | See section 6 | no rows |

`metrics.compute(series, as_of)` takes the closes indexed by date (NaN dropped, sorted) and the universe `as_of`, and returns a dict with keys `status first_date last_date days_stale <FIELDS> why`, in that order. `metrics.FIELDS = ["r_1m","r_3m","r_6m","r_ytd","r_1y","r_3y_ann","vol_1y","maxdd_1y"]` and `metrics.RETURNS = {"r_1m":1,"r_3m":3,"r_6m":6,"r_1y":12}` (months back). An empty series returns `status: "no_data"`, `first_date`, `last_date`, `days_stale` null, every field null and `why` mapping every field to `no_data`. `why` otherwise holds `{field: code}` for exactly the null fields. Dates are ISO strings, `days_stale` an `int`, numbers `float` at full precision.

Slice 11 (section 11): `build` feeds these formulas the screened series, so a spike close is absent and, after a gap longer than `GAP_BREAK_DAYS`, `first_date`, the anchors, `r_si`, `maxdd` and the `short` test start at the segment after that gap. The sample's 60-business-day gap (84 calendar days) therefore starts that ticker's analysis after its gap, and its status is `short`. `metrics.compute` itself does not screen, so the tests that call it directly keep their gapped series.

Notes:
- A week with no close yields an empty bucket, so its return and the next one are NaN (no fill). A long gap therefore removes vol, not the point-to-point returns.
- Volatility uses weekly returns only; daily-return volatility is not used anywhere.
- `adj_close` returns are total returns only to the extent the provider adjusts for distributions; the Data tab (slice 11) states this once.

## 8. `status` per fund

Evaluated in this order; first match wins.

| `status` | Condition | UI |
|---|---|---|
| `review_excluded` | a review `exclude` mark (slice 12); metrics are still computed | row kept and dimmed, left out of every aggregate |
| `review_broken` | a whole-series `broken_price` mark (slice 12) | row kept, all metrics em dash |
| `no_data` | no price rows | row kept, all metrics em dash |
| `stale` | `days_stale > STALE_DAYS` | row kept, warning colour, metrics shown against the fund's own last date |
| `short` | history shorter than 12 months (`first_date > t_end - 12 months`) | row kept, 1Y/3Y metrics null |
| `ok` | otherwise | normal |

Funds are never removed for data or review reasons; they stay visible with the reason. Aggregates (group, market, correlation, PCA, rotation; slices 4 to 9) take their members from funds whose status is not `review_excluded`, `review_broken` or `no_data`; `build.py` applies this once, in one helper.

## 9. `web/data.js` schema v1

File content: the comment line `// GENERATED by etf-arcade build. Do not edit.`, then one statement `window.ARCADE_DATA = <JSON>;` and a newline. The JSON is produced with `json.dumps(data, separators=(",", ":"), allow_nan=False)`, so `NaN`/`Infinity` can never appear; missing numbers are `null`.

Top level (key order as listed):

| Field | Type | Slice | Meaning |
|---|---|---|---|
| `schema` | int | 1 | Always `1`. `app.js` refuses a different value and shows an error panel. |
| `generated` | string | 1 | UTC ISO-8601 build time, seconds precision (`2026-01-02T09:30:00Z`). The only non-deterministic field. |
| `as_of` | string | 1 | Universe `as_of` date (section 6). Header text "As of <date>". |
| `source` | string | 1 | The `prices.csv` header label (`yfinance`, `synthetic`, ...). |
| `synthetic` | bool | 1 | `source == "synthetic"`. The header shows `source`; no banner. |
| `project` | object | 1b | `{title, subtitle, currency, universe_label, dir_name}` (section 2b). The header shows `title` and `subtitle`; `universe_label` replaces "ETFs" in UI text. |
| `funds` | array | 1 | One object per universe row, universe-file order. |
| `groups` | array | 1 | One object per group, first-appearance order. |
| `params` | object | 3 | `{line_gap_days, ew_warn, signal}`: constants the browser needs (`ew_warn` from slice 7; `signal` from slice 10 is `{h, win, z_min, pct_min, sd_min, custom_floor, custom_share}`, the Signal lab constants of section 12). Slice 13 appends `roll` `{weeks, min}` (13a, from `VOL_WEEKS`, `VOL_MIN_WEEKS`), `factors` `{equity, bond}` or `null` (13a), `t_crit` (13e) and `constants` (13h, `{name: value}` of every constant after the project's overrides). |
| `quality` | object | 11 | Output of the data-quality checks (section 11). |
| `review` | object | 12 | `{files, marks, counts, warnings}` of the active review marks (section 14); always present. |
| `benchmarks` | array | 4 | The `[[benchmarks]]` of `settings.toml`, in order (the first is the UI default; empty without any): `{id, label, components, series_file}` plus `returns` (slice 5: the six period ids `r_1m r_3m r_6m r_ytd r_1y r_3y_ann`, each a decimal fraction or `null`, from the benchmark's level series at its own last date with the anchor rule of METRICS 3.1; the benchmark may end before the universe `as_of`). `series_file` is `series/_bench_<id>.js`. Slice 13a: `returns` is evaluated at the coverage cutoff `C` (the last level on or before `C`, so it compares with `group_return` on the same dates) and holds nine ids (`r_1w` and `r_5y_ann r_10y_ann` added); each entry appends `metrics` and `why` (table below). Key order `id label components series_file returns metrics why`. |
| `cash` | object or null | 4 | `{ticker, label, kind, series_file}` from `[cash]` (`series_file` is `series/_cash.js`), or `null` without one. |
| `market` | object | 4 | As built: `{dates, above_50d, above_200d, new_hi, new_lo, dispersion_21d, impl_corr_63d}`: `dates` is the market axis (the dates on which at least `MARKET_MIN_COVERAGE` of the member funds have a close), every other key an array of the same length, a number or `null` (shares and dispersion are decimal fractions, `impl_corr_63d` a plain number). Full-universe, never recomputed for a group filter. Definitions: METRICS section 11. Slice 13 appends `ew_index`, `groups_above_200d` (13c) and `pos_21d` (13g) (table below). |
| `rotation` | object | 7 | As built: `{dates, cutoff, benchmarks, rank_trail}`. `dates`: the last `RRG_TAIL_GROUPS` (156) W-FRI weeks up to the last Friday on or before `cutoff` (the slice-11 cutoff date, null without prices). `benchmarks`: one key per benchmark id (empty without benchmarks), each `{groups: {name: E}, funds: {ticker: E}}` with `E = {rs, rrg_ratio, rrg_mom, rrg_quadrant}`: arrays on `dates` for every group (all-null arrays when no ratio exists), and arrays on the last `RRG_TAIL_FUNDS` (52) entries of `dates` for the member funds that have a ratio there. `rs rrg_ratio rrg_mom` are stored rounded to `ROT_DECIMALS` (4) decimals (size; a second stored rounding besides correlation), `rrg_quadrant` is one of `Leading Weakening Improving Lagging` or null. `rank_trail`: `{months, horizon_months, ret, rank}`: `months` the last date on or before the cutoff in each calendar month, `ret` and `rank` map every group name to arrays on `months` (trailing 3-month return; rank 1 = best among the groups with a return, ties by group order; null when the group has none). The browser re-ranks `ret` among the visible groups under a filter. Slice 13c appends `rank_trail.by_period` (table below). METRICS 7.3, 7.4. |
| `calendar` | object | 7 | As built: `{benchmarks, groups, funds}`, each a map (benchmark id / group name / member-fund ticker) to `null` (benchmark without levels) or `{start, monthly_return, year_return, season_mean, season_median, season_hit}`: `start` the first month `YYYY-MM`, `monthly_return` consecutive months from there (null where a month has no return), `year_return` maps a year to `{ret, n}` (n months with a return), the three `season_*` arrays hold 12 values (January first), null without a full year. Groups without any index level and funds without closes are absent. METRICS 7.5. |
| `correlation` | object | 9 | As built: `{cutoff, corr_1y_daily, corr_3y_weekly}`; `cutoff` is the slice-11 cutoff date (null without prices). Each window is `{end, rows, min_pairs, n, status, order, upper}`: `end` the last row's date (the last axis date for the daily window, the last Friday on or before `cutoff` for the weekly one), `rows` the number of rows used (`CORR_DAILY_ROWS` 252 or `CORR_WEEKLY_ROWS` 156 at most), `min_pairs` the pair minimum (`CORR_*_MIN`), `n` the funds with at least `min_pairs` returns in the window, `status` `ok` or `insufficient_data` (n under `PCA_MIN_FUNDS`; then `order` and `upper` are empty). `order` lists the `n` tickers in display (average-linkage) order; `upper` is the upper triangle without the diagonal of the matrix in that order, row-major (entry of rows `i < j` at `i*(2n-i-1)/2 + (j-i-1)`), `n*(n-1)/2` numbers rounded to `CORR_DECIMALS` (3) or `null` for a blank pair (fewer than `min_pairs` overlapping returns, or zero variance). Only member funds (section 8) appear. The daily axis is the market-style axis (dates up to the cutoff on which `MARKET_MIN_COVERAGE` of the funds have a close); a daily return needs the fund's previous close at most 7 calendar days earlier. Slice 13e appends `corr_3y_residual` and `groups` (table below). METRICS 7.1. |
| `pca` | object | 9 | As built: `{pca_status, pca_n, pca_orient, pca_fixed, pca_rolling}`. `pca_status` `ok`, `insufficient_data`, `zero_dispersion` or `indefinite_correlation`; `pca_n` `{used, excluded}` (member funds in and out of the 104-week complete-case sample); `pca_orient` `mean_loading` or `largest_element` (the rule that fixed the PC1 sign), null unless ok; `pca_fixed` null unless ok, else `{end, weeks, shares}` with `shares` the variance shares of components 1 to `PCA_COMPONENTS` (10); `pca_rolling` `{dates, n, pc1, pc2, pc3}`: one entry per weekly return date up to the cutoff, `n` the complete funds in the 52-week window ending there, `pc1..pc3` the shares, all null where fewer than `PCA_MIN_FUNDS` funds are complete. Full-universe, never recomputed under a group filter. Slice 13 appends `pca_rolling.avg_corr within across corr_ref` (13a), `pca_rolling.ar5 pc1_resid`, `pca_resid_status pca_resid_n pca_resid_fixed mds clusters` (13e) (table below). METRICS 7.2. |
| `group_index` | object | 5 | As built: `{dates, series, breaks}`: `dates` is the market axis, `series` maps every group name (`groups[]` order) to an array of the same length (index level, 100 at the group's first member close, `null` on dates where no member has a close), `breaks` maps each group name to its chain-break count (METRICS 6). Members come from the one member filter (section 8). Rebasing to the chart history is done in the browser. |

`funds[]`:

| Field | Type | Slice | Notes |
|---|---|---|---|
| `ticker`, `name`, `short_name`, `group`, `family`, `currency` | string | 1 | Straight from `universe.csv`. |
| `status` | string | 1 | Section 8. |
| `first_date`, `last_date` | string or null | 1 | ISO dates. |
| `days_stale` | int or null | 1 | |
| `r_1m`, `r_3m`, `r_6m`, `r_ytd`, `r_1y`, `r_3y_ann` | number or null | 1 | Decimal fractions. |
| `vol_1y` | number or null | 1 | Annualised, decimal fraction. |
| `maxdd_1y` | number or null | 1 | Decimal fraction <= 0. |
| `r_si` | number or null | 3 | Annualised return since the first close (METRICS 3.2). Null with `insufficient_history` when the history is 0.5 years or less. |
| `dd_now`, `maxdd` | number or null | 3 | Current and worst full-history drawdown, decimal fractions <= 0 (METRICS 3.2). `maxdd` is null (`insufficient_history`) with fewer than 2 closes; both are `no_data` for a fund without closes. |
| `why` | object | 1 | `{field: reason_code}` for each null plain metric; `{}` if none. Reference-dependent fields use the object shape below. Codes below. |
| `rel_3m`, `rel_1y` | object | 4 | Reference-dependent: `{benchmark id: number or null}` with one key per `benchmarks[]` entry. Fraction: fund return minus the benchmark's return over the same dates. |
| `sharpe_3y` | number or null | 4 | Plain ratio, depends on cash only. Null (`cash_missing`, `cash_incomplete`) when the cash series is absent or incomplete. |
| `beta_3y` | object | 4 | Reference-dependent, plain ratios keyed by benchmark id. |
| `up_cap`, `down_cap` | object | 8 | Reference-dependent capture ratios keyed by benchmark id (METRICS 5): plain ratios, 1.0 = as much as the reference. Null with `insufficient_pairs` (under 104 pairs, or an empty up or down regime) or `reference_missing`. |
| `vs_200d` | number or null | 5 | Distance from the mean of the last 200 closes, decimal fraction (METRICS 3.2). Null with `insufficient_history` under 200 closes. |
| `mom_12_1` | number or null | 5 | 12-1 momentum, decimal fraction (METRICS 3.2). Null with `no_history_to_anchor`. |
| `from_52wh`, `from_52wl` | number or null | 7 | `P / max(closes) - 1` (at most 0) and `P / min(closes) - 1` (at least 0) over the closes dated on or after `t_end - 12 months`; null with `insufficient_history` under `HILO_MIN_CLOSES` (100) closes (METRICS 3.2). |
| `pc1`, `pc2` | number or null | 9 | PCA loadings (plain numbers); subject to the `PCA_WEEKS` complete-case rule. As built: null for a fund outside the sample, and they have no `why` entry and are not in `metrics.FIELDS` (the reason is the `pca_n` count, shown on the PCA tab). PC1 is oriented by `pca_orient`; PC2 and PC3 so that the largest-magnitude loading is positive. |
| `rel_13w`, `z_13w`, `pct_13w`, `z_rank` | object | 10 | Reference-dependent (`{benchmark id: number or null}`), the canonical signal of METRICS 8 at the fund's final weekly slot (its own last close, the slot of `weekly_arrays`): `rel_13w` the 13-week excess return (decimal fraction), `z_13w` its z-score in the trailing 156 weekly values, `pct_13w` its percentile in that window (a fraction), `z_rank` the share of the fund's finite weekly `z_13w` over its full history at or below today's. Null reasons: `reference_missing` (the reference has no value in the final week or its 13-week window, or no levels), `insufficient_history` (own window incomplete, fewer than 104 finite values for z, 52 for the percentile), `zero_variance` (sample SD at or below `SIGNAL_SD_MIN`); `z_rank` carries the reason of `z_13w`. Stored at full precision. |
| `group_original` | string | 12 | Present only on a fund moved by a review mark: its `universe.csv` group. `group` and `family` hold the target group's values. |
| `r_1w`, `r_5y_ann`, `r_10y_ann` | number or null | 13a | 1-week, 5-year annualised and 10-year annualised returns (PARITY 13a.3), decimal fractions; null `no_history_to_anchor`. Appended to `metrics.FIELDS` with the next four in this order. |
| `years` | number or null | 13a | Years from `first_date` to `t_end`; null `no_data`. |
| `vol_3y` | number or null | 13a | 156-slot weekly volatility, decimal fraction; null `insufficient_weekly_returns`. |
| `sortino_3y` | number or null | 13a | Plain ratio, cash-dependent; null `cash_missing`, `cash_incomplete`, `insufficient_pairs`, `zero_variance`. |
| `hit_1y` | number or null | 13a | Share of the last 52 weekly returns above 0 (fraction); null `insufficient_weekly_returns`. |
| `corr_3y`, `rel_1w`, `rel_1m`, `rel_6m`, `rel_ytd`, `rel_3y_ann` | object | 13a | Reference-dependent, appended to `metrics.REF_FIELDS` in this order: 156-slot weekly correlation with the benchmark (plain), and the ETF return minus the benchmark return over the same dates for the other periods (decimal fractions; `rel_3y_ann` annualised legs). PARITY 13a.3, 13a.4. |
| `alpha_3y`, `alpha_t_3y`, `beta_equity_3y`, `beta_bond_3y`, `res_vol_3y`, `appraisal_3y`, `alpha_full`, `alpha_t_full` | number or null | 13e | Two-factor fit on weekly excess returns with Newey-West t (PARITY 13e.2), plain fields appended to `metrics.FIELDS` in this order; alpha and `res_vol` decimal fractions a year. Null `factors_missing`, `cash_missing`, `cash_incomplete`, `insufficient_pairs`, `zero_variance`. |
| `rpc1`, `rpc2` | number or null | 13e | Residual-PCA loadings (PARITY 13e.3); like `pc1 pc2`: no `why` entry, not in `metrics.FIELDS`. |
| `mds_x`, `mds_y` | number or null | 13e | Residual-map coordinates (PARITY 13e.4); null when the ETF is not placed; no `why` entry. |
| `cluster` | string or null | 13e | Computed cluster name (PARITY 13e.4); null when not placed; no `why` entry. |
| `series_file` | string | 3 | Path of the fund's series file, e.g. `series/SPY.js`; absent for funds without closes. |

Reference-dependent fields (rule for every slice): a field computed against a benchmark is stored as an object keyed by benchmark id, one key per entry of `benchmarks[]`, value a number or `null`. With no benchmark configured the object is `{}`. A field counts as null when any of its entries is null; its `why` entry is then an object `{benchmark id: code}` holding exactly the null entries, and is absent when none is null. The list of reference-dependent fields is in METRICS section 4. The UI reads them through `ARCADE.ref(row, field)` for the benchmark selected in the settings strip (UI.md sections 4 and 7).

Series files (slice 3): `web/series/<key>.js` assigns `window.ARCADE_SERIES[key] = {generated, dates, adj_close}`. `key` is a fund ticker (file name `<ticker>.js`), `_bench_<id>` for a benchmark (`dates` and `adj_close` hold its level, base 100) or `_cash` (`dates` and `rate`, the annual yield as a decimal fraction; may be zero or negative; slice 13a adds `wdates`, the W-FRI Fridays, and `cash_w`, the weekly cash return of METRICS 4 on them, null where incomplete). Only existing values are written, so a gap is a missing date. `generated` equals the `data.js` stamp, and `ARCADE.series` shows "Reload: data was rebuilt" on a mismatch. Later slices add weekly arrays on their own date list `wdates`: `roll_ret`, `drawdown_t` (slice 6; as built: fund files only, not the benchmark or cash files; `wdates` holds one ISO date per `W-FRI` week from the fund's first to its last close, each the Friday except the final slot, which carries the date of the last close; the three arrays have the length of `wdates`, `null` for a week without a close or without a value; written from the cleaned series, so review marks and the quality screen apply), `rel_13w`, `z_13w` (slice 10; reference-dependent, so each is an object keyed by benchmark id; as built: fund files only, on the `wdates` slots, null where the value is undefined, rounded to `SIGNAL_DECIMALS` (4) as the third stored rounding after correlation and rotation, because the arrays add about 15 MB for 466 funds x 5 benchmarks; the percentile is not stored, the browser derives it from `rel_13w`). Daily drawdown is computed in the browser from `adj_close`. Files load by script tag, never `fetch()`.

`groups[]`: `{name, family, index, n, ...}` where `index` is the 0-based group order (colour = `index % 12`) and `n` is the fund count (from slice 12 without `review_excluded` funds, after moves). Slice 5 adds `group_return` (object keyed by the six period ids, each the return of the group's `group_index` ending at the coverage cutoff `C`, METRICS 6; `null` when the index has no level at an anchor), `med_r_1m`, `med_r_3m`, `med_r_1y` (member medians) and `breadth_200d` (share of finite `vs_200d` above 0), all `null` when no member qualifies; a group without member funds holds all-null aggregates. Slice 6 appends `beat_ref_3m` (object keyed by benchmark id: share of members with a finite `rel_3m` for that benchmark whose value is above 0, `null` when none is finite; `{}` without benchmarks), `dispersion_1y` (sample standard deviation of members' `r_1y`, `null` under 2 members), `med_vol_1y` and `med_maxdd_1y` (member medians). Slice 7 appends `ew_ratio` (object keyed by benchmark id: the last value of `group_index / reference level` on the dates where both exist, null under 200 observations) and `ew_tells` (object keyed by benchmark id: count 0 to 6 of the early-warning tests, null under 220 observations; METRICS 7.4). Slice 13 appends the fields of the table below. Key order: `name family index n group_return med_r_1m med_r_3m med_r_1y breadth_200d beat_ref_3m dispersion_1y med_vol_1y med_maxdd_1y ew_ratio ew_tells med_z_13w index_metrics index_why factor_fit ew_dir ew_days ew_rel_3m`. Top-level order: `... group_index rotation calendar correlation pca`. Under a group filter any full-universe aggregate must read "unavailable", never a stale full-universe number.

Slice 13 fields inside existing objects (specs in `docs/reference/PARITY.md`; reference-dependent ones are objects keyed by benchmark id):

| Field | Type | Slice | Notes |
|---|---|---|---|
| `groups[].group_return` keys `r_1w`, `r_5y_ann`, `r_10y_ann` | number or null | 13a | Nine period ids in all, same rule (index return at `C`). `r_ytd` is null when `as_of` is in a later calendar year than `C` (13a.6). |
| `groups[].med_z_13w` | object | 13a | Median of members' `z_13w` at `C` (13a.7). |
| `groups[].index_metrics` | object | 13a | Every `metrics.FIELDS` and `REF_FIELDS` id computed on the group's `group_index` cut at `C` (13a.6). |
| `groups[].index_why` | object | 13a | Reasons for the null `index_metrics` entries, shape of `funds[].why`. |
| `groups[].factor_fit` | object | 13e | `{"3y", "12m", "ytd", "full"}`, each `{alpha, alpha_se, alpha_t, beta_equity, beta_bond, res_vol, ar, n, start, end, status}` (13e.2). |
| `groups[].ew_dir`, `ew_days`, `ew_rel_3m` | object | 13c | Direction (-1, 0, 1), signed streak (int) and 3M ratio change (fraction) of the early-warning ratio (13c.4). |
| `benchmarks[].metrics` | object | 13a | Every `metrics.FIELDS` and `REF_FIELDS` id on the benchmark level at its own last level (13a.8). |
| `benchmarks[].why` | object | 13a | Reasons for the null `metrics` entries. |
| `market.ew_index` | array | 13c | Equal-weight index of all member funds on `market.dates`, null where no member qualifies (13c.6). |
| `market.groups_above_200d` | array | 13c | Share of groups above their 200-level mean (fraction or null) (13c.6). |
| `market.pos_21d` | array | 13g | Share of members with a positive 21-row return (fraction or null) (13g.9). |
| `pca.pca_rolling.avg_corr`, `within`, `across` | array | 13a | Mean pairwise correlation of each 52-week window: all pairs, same-group, cross-group (13a.9). |
| `pca.pca_rolling.corr_ref` | object | 13a | `{benchmark id: array}`, mean ETF-to-benchmark correlation per window (13a.9). |
| `pca.pca_rolling.ar5`, `pc1_resid` | array | 13e | Top-5 absorption ratio and PC1 share of residual returns per window (13e.3). |
| `pca.pca_resid_status`, `pca_resid_n`, `pca_resid_fixed` | string, object, object or null | 13e | Residual PCA on 104 weeks, shapes of `pca_status`, `pca_n`, `pca_fixed` (13e.3). |
| `pca.mds` | object | 13e | `{status, n, dropped}` of the residual map (13e.4). |
| `pca.clusters` | array | 13e | Cluster names in column order (13e.4). |
| `correlation.corr_3y_residual` | object | 13e | Window shape of `corr_3y_weekly` on equity-factor residuals; status also `factors_missing` (13e.5). |
| `correlation.groups` | object | 13e | `{corr_3y_weekly, corr_1y_daily}`, each `{end, rows, min_pairs, ids, upper}`; `ids` group names then `bench:<id>` (13e.8). |
| `rotation.rank_trail.by_period` | object | 13c | `{period id: {ret, rank}}` on `rank_trail.months` for the seven period ids (13c.9). |
| `params.roll`, `params.factors`, `params.t_crit`, `params.constants` | object, object or null, number, object | 13a, 13a, 13e, 13h | See the `params` row above. |
| Series file `_cash.js` `wdates`, `cash_w` | arrays | 13a | Weekly cash return on W-FRI Fridays (13a.2). |

As built (slice 13a): `group_return` is the return of the group index cut at `C` for the nine period ids and equals `index_metrics[p]` (slice 5 had published the mean of the members' own returns, contradicting METRICS 6; `group_aggregates` no longer returns `group_return`). Member values for `med_* breadth_200d beat_ref_3m dispersion_1y med_z_13w` come from `compute(series cut at C, C)` for a member with a later close. `benchmarks[].returns` is `period_returns` of the level series cut at `C` (no YTD rule for benchmarks). `rel_*` read the reference with the alignment carry (`_level_at`, `CASH_CARRY_DAYS`; the doc constant `CARRY_DAYS` is named `CASH_CARRY_DAYS` in `constants.py`), which also changes `rel_3m` and `rel_1y` where the reference has no level on the exact date. `pca_rolling` key order `dates n pc1 pc2 pc3 avg_corr within across corr_ref`; `avg_corr within across` are null exactly where `pc1` is null, `corr_ref` needs only 8 complete funds. `params.factors` is the `[factors]` table whenever it is set (also when a factor ticker has no rows). `_cash.js` `wdates` run from the first Friday after the first fixing to the last Friday on or before `as_of`.

As built (slice 13e): `funds[]` key order ends `... pc1 pc2 rpc1 rpc2 mds_x mds_y cluster`; `groups[]` gains `factor_fit` after `index_why`; `correlation` key order `cutoff corr_1y_daily corr_3y_weekly corr_3y_residual groups`; `pca` key order `pca_status pca_n pca_orient pca_fixed pca_rolling pca_resid_status pca_resid_n pca_resid_fixed mds clusters`; `pca_rolling` ends `... corr_ref ar5 pc1_resid`. The factor fits use the weekly slots of `compute` (`wret` up to `as_of`, the Sharpe window for `3y`) and the weekly return series of the factor tickers' cleaned closes (`ref_weekly`); a configured factor whose ticker has no usable closes counts as no factors (`factors_missing`), though `params.factors` still lists it. A fit whose design has rank 3 but whose residual (or its standard error) is exactly zero is `zero_variance`; an almost perfect fit still reports its huge t. `full` fits need cash on every slot with finite `r`, `e`, `b`, so a fund older than the cash series has `alpha_full` null `cash_incomplete`. Residuals (`resid`) come from `weekly_returns` of the member closes and of the equity factor (rows after the last Friday on or before `C` are dropped); `pca_resid_n.excluded` counts the members without a complete residual window (including those without residuals). `ar5` and `pc1_resid` are rounded to `CORR_DECIMALS` and null exactly where the window is not ok (`pc1_resid` also when fewer than 8 funds have complete residuals). `loadings` of `pca()` return `[pc1, pc2, rpc1, rpc2]` per ticker. `mds` drops, one at a time, the ETF with the most blank residual-correlation pairs (ties: later in universe order) until no pair is blank; `mds.n` is the placed count and `mds.dropped` the ETFs with residuals that were left out (`insufficient_data` gives `n` 0). Clusters: average linkage on the residual-map distance until `min(groups with a placed member, n)` clusters remain, named as PARITY 13e.4 says; only a (cluster, group) pair with at least one member can name a cluster. `correlation.groups` windows hold `{end, rows, min_pairs, ids, upper}` with `rows` the number of return rows used (156 or 252 at most) and `end` null when no row exists (all `upper` null). `groups[].factor_fit` windows `12m` and `ytd` use the group's last weekly slot as `t_end`. `params.t_crit` is `T_CRIT`. `build.py` validates all of these (a mismatch exits 2).

`why` reason codes and the text `app.js` shows for them (the `WHY` table there; slice 1 holds every code below except `review_broken_price`, which slice 12 adds):

| Code | Meaning | UI text |
|---|---|---|
| `no_data` | the fund has no price rows | No price data for this fund |
| `no_history_to_anchor` | no close within `ANCHOR_TOL_DAYS` before the anchor date | No close near the start of this window (history too short or a gap) |
| `insufficient_weekly_returns` | fewer than `VOL_MIN_WEEKS` finite weekly returns | Too few weekly returns in the volatility window |
| `insufficient_daily_closes` | fewer than `DD_MIN_CLOSES` closes in the window | Too few daily closes in the drawdown window |
| `insufficient_history` | later slices: history shorter than the metric needs (for example `r_si`) | History is shorter than this metric needs |
| `cash_missing` | `settings.toml` has no `[cash]` (or the cash ticker has no rows) | No cash series is configured |
| `cash_incomplete` | a cash-rate week is missing in the window | A cash-rate week is missing in this window |
| `reference_missing` | no benchmark configured, or the benchmark has no value in the window | The benchmark has no value in this window |
| `insufficient_pairs` | fewer overlapping observations than the metric's minimum | Too few overlapping observations |
| `zero_variance` | zero variance in the window | No variation in this window |
| `review_broken_price` | every close was removed by a review mark | Prices removed by a review mark |
| `factors_missing` | slice 13a: `settings.toml` has no `[factors]`, or a factor ticker has no rows | No factor series is configured |
| `cutoff_prior_year` | slice 13a: a group's YTD at `C` when `as_of` is in a later calendar year | The coverage cutoff is in the previous calendar year |

Compatibility rules:
- Additive changes within `schema: 1` are fine (new optional fields). Renaming, removing or changing the unit or shape of an existing field bumps `schema`.
- JS must treat a missing optional field the same as `null`.

## 10. Build and publish safety (`build.py`)

Order: load project -> load universe -> read prices -> apply review marks (`review.py`, slice 12) -> screen (`quality.py`, slice 11) -> compute metrics -> serialise in memory -> validate -> print warnings -> write series into `web/series.tmp` (slice 3) and swap it in as `web/series/` -> write `web/project.css` -> write `web/data.js.tmp` -> `os.replace` (`data.js` last).

| Rule | Behaviour |
|---|---|
| Atomic | Write tmp in `web/`, then `prices.replace` (`os.replace`, retried on Windows `PermissionError`). Readers see the old or the new file, never a partial one. |
| Failure keeps the old file | Any exception before the replace removes the tmp file and directory, leaves `web/data.js` byte-identical, prints the reason (`error: <message>` on stderr), exit code 2. |
| Validation before replace | Re-parse the serialised JSON (text between `window.ARCADE_DATA = ` and the last `;`); check `schema == 1`, `len(funds) == len(universe)`, unique tickers, no `NaN` text, `project` has exactly the keys `title subtitle currency universe_label dir_name`, for every fund the `why` keys are exactly the null plain metric fields (`metrics.FIELDS`; later slices extend the list and add the object-field rule above) and `first_date` is null exactly when `status == "no_data"` (slice 11 and 12 widen this to: when the fund has no closes left, which covers `review_broken` and an excluded fund with none). |
| Hard stop | Fewer than `MIN_PRICED_FRACTION = 0.5` of universe tickers have any price row in `prices.csv` (counted before review marks): `ValueError("only N of M universe tickers have prices (minimum 50%)")`, no publish (a broken fetch must not replace a good page). |
| No fabrication | No default prices, no carried-forward values, no synthetic fallback when `prices.csv` is missing: missing input is an error. |
| Synthetic labelling | `synthetic` is always set from the `prices.csv` header, never inferred or defaulted to false. |
| Warnings | Dropped rows (one line per reason: `warning: N price rows dropped: <reason> (first at line L)`; unknown tickers are one reason) and duplicate short names (`warning: N duplicate short names, e.g. '<name>'`): printed to stdout, exit 0. The same list seeds the slice-11 `quality` object. |
| Exit codes | `0` published, `2` error (previous output kept). Every command uses these two codes. |
| Ignored files | `var/`, `examples/*/var/`, `web/data.js`, `web/project.css`, `web/series/`, `*.tmp` stay gitignored. |

## 11. Data-quality checks (slice 11)

`quality.py` owns the screen: `screen(prices, issues=(), tickers=None) -> (cleaned prices, flags)` (`issues` are the dropped rows from `prices.read`, counted per reason into `flags["dropped"]`; `tickers` is the universe in file order, which `flags["funds"]` and the `unpriced` count need, default the tickers present). The screen runs before `metrics.compute`, so it cannot know `status`, `first_date` or `last_date`: `build` copies those three from the metrics of the cleaned series into each `quality.funds[]` entry (key order as in the example) and validates that `stale` is a flag exactly when `days_stale > STALE_DAYS` (from slice 12 the status of an excluded fund is `review_excluded`, so the check reads `days_stale`, not `status`). `build` calls it before `metrics.compute()`, so metrics and series files see the cleaned prices (`prices.csv` is never edited), and stores the flags as `quality` in `data.js` for the Data tab. `metrics.py` never imports `quality.py`. Thresholds are constants in section 12 (all added to `constants.py` by slice 11, including `CUTOFF_LAG_BDAYS`).

| Check | Rule | Severity |
|---|---|---|
| Missing prices | ticker in universe, no rows | error row, fund `no_data` |
| Unknown tickers | rows for tickers outside the universe | warning |
| Dropped rows | bad value/date, future date, duplicates (section 3) | warning, counts by reason |
| Staleness | `days_stale > STALE_DAYS` | warning per fund |
| Gap | consecutive closes more than `GAP_DAYS` calendar days apart | warning with dates |
| Large move | absolute close-to-close change > `MOVE_WARN` | warning with date (`moves`) |
| Spike | interior point with `abs(v/prev - 1) > SPIKE_MOVE` and next point back within `SPIKE_REVERT` of prev: excluded from analysis (stored data unchanged, no interpolation), reported | warning, counted |
| Gap break | a gap > `GAP_BREAK_DAYS`: analysis uses only the segment after the last such gap; reported with the break date (no splicing across it) | warning |
| Short history | under 12 months (`short`), under 3 years (`short_3y`) | info |
| Coverage | number of funds with a close on `as_of`, and on the cutoff date | info |
| Cutoff | the date `flags["coverage"]["cutoff"]`: the last date with any observation minus `CUTOFF_LAG_BDAYS` business days, moved back to the last observed date on or before it. Computed once here; `build` passes it to the cross-sectional outputs of slices 7 and 9 (correlation, PCA, rotation), which end there; the header still says "As of <as_of>" | info |
| Source label | `source` and `synthetic` shown verbatim | info |

Per-ETF `quality.funds[]` entries follow the example: one per universe ETF in file order; `n_obs` is the number of stored closes; `flags` holds any of `stale`, `gap`, `spike`, `break` (the four chips shown everywhere; `stale` equals `status: stale`); `gaps`, `spikes` and `moves` list the dates behind them; `break_date` is the latest gap-break date or null; `counts` counts ETFs, not events; `params` holds the live constants used.

Details the table leaves open (as built): gaps, large moves and `n_obs` describe the stored closes before the screen's own cleaning (from slice 12: after the review marks, section 14); a gap break is found on the stored closes, then spikes are searched only inside the segment after the last break (a jump across a break is a break, not a spike), so `spikes` and the cleaned series never mention data before it. `spikes` and `moves` entries are `{date, change}` (`change` = close over previous close minus 1; a spike's previous close is the previous stored close); `gaps` entries are `{from, to}` (the two closes either side). `short` counts funds whose first cleaned close is within 12 months of their last, `short_3y` within 36 months (so `short` is part of `short_3y`); `unpriced` funds are in neither. `first_date` and `last_date` are those of the cleaned series (the segment start after a break). `coverage.on_as_of` and `on_cutoff` count stored closes on those dates; `cutoff` and both counts are null for an empty price file. The cutoff is the last observed date over all rows on or before `as_of - CUTOFF_LAG_BDAYS` business days (`pandas.offsets.BDay`, so public holidays are not skipped). `screen` raises `ValueError` when `SPIKE_REVERT` is not below `SPIKE_MOVE`.

```json
"quality": {
  "counts": {"unpriced": 1, "stale": 1, "gap": 1, "spike": 0, "break": 1, "move": 2, "short": 8, "short_3y": 40},
  "params": {"stale_days": 14, "gap_days": 10, "move_warn": 0.25, "spike_move": 0.3, "spike_revert": 0.05,
             "gap_break_days": 60, "cutoff_lag_bdays": 10},
  "coverage": {"as_of": "2025-12-26", "on_as_of": 508, "cutoff": "2025-12-12", "on_cutoff": 508},
  "dropped": [{"reason": "duplicate (ticker, date)", "n": 2}],
  "funds": [{"ticker": "AAA", "status": "ok", "first_date": "2019-12-27", "last_date": "2025-12-26",
             "n_obs": 1500, "flags": ["gap"], "gaps": [{"from": "2025-05-02", "to": "2025-07-25"}],
             "spikes": [], "moves": [{"date": "2025-07-25", "change": 0.27}], "break_date": null}]
}
```

Slice 11 changes slice-1 behaviour where a metric changes: the sample's 60-business-day gap exceeds `GAP_BREAK_DAYS`, so after slice 11 that ticker's analysis starts after its gap (`first_date` is the segment start) and the gap test in section 13 (item 4 of the slice-1 list) keeps its own gap-free-vs-gapped series by calling `metrics.compute` directly, which does not screen. Update section 7 accordingly. From slice 4 the cash series is not screened (build removes its rows before calling `screen`); benchmark-only tickers are screened but not listed in `quality.funds`.

## 12. Constants (no config system)

Every tunable constant is defined once in `etf_arcade/constants.py` and imported by name (`from .constants import X`), never copied. A later slice adds its constants there (column Slice). Names are upper case; values are plain `int`, `float` or `str` (`project.apply` type-checks `settings.toml` overrides against them).

| Constant | Value | Slice | Meaning |
|---|---|---|---|
| `STALE_DAYS` | 14 | 1 | stale threshold, calendar days |
| `ANCHOR_TOL_DAYS` | 7 | 1 | anchor tolerance |
| `VOL_WEEKS` / `VOL_MIN_WEEKS` | 52 / 40 | 1 | vol window / minimum finite weekly returns |
| `DD_MIN_CLOSES` | 200 | 1 | minimum daily closes in the 1Y drawdown window |
| `ANNUAL_WEEKS` | 52 | 1 | annualisation factor |
| `SHORT_NAME_MAX` | 24 | 1 | longest `short_name` |
| `MIN_PRICED_FRACTION` | 0.5 | 1 | hard-stop coverage |
| `FETCH_YEARS`, `FETCH_BATCH`, `FETCH_MAX_FAIL_FRACTION` | 6, 100, 0.5 | 1 | download behaviour |
| `SEED`, `END_DATE`, `YEARS`, `BDAYS_PER_YEAR` | 20240229, "2025-12-26", 6, 261 | 1 | sample determinism |
| `GAP_BDAYS`, `STALE_BDAYS`, `GAP_FROM_END` | 60, 20, 127 | 1 | the sample's gap and stale tickers |
| `LINE_GAP_DAYS` | 7 | 3 | charts break a line beyond this many days; published as `params.line_gap_days` |
| `GAP_DAYS` / `MOVE_WARN` | 10 / 0.25 | 11 | gap and large-move warnings |
| `SPIKE_MOVE` / `SPIKE_REVERT` | 0.3 / 0.05 | 11 | spike candidate; `SPIKE_REVERT < SPIKE_MOVE` |
| `GAP_BREAK_DAYS` | 60 | 11 | analysis uses only the latest segment |
| `CUTOFF_LAG_BDAYS` | 10 | 11 | cutoff lag of every group-level and cross-sectional output (slices 4 to 10, METRICS 1 and 10) |
| `PCA_WEEKS` / `PCA_MIN_FUNDS` | 104 / 8 | 9 | complete-case sample; fewer funds -> `insufficient_data` |
| `CORR_DAILY_ROWS` / `CORR_DAILY_MIN` | 252 / 200 | 9 | daily correlation window / minimum pairs |
| `CORR_WEEKLY_ROWS` / `CORR_WEEKLY_MIN` | 156 / 104 | 9 | weekly correlation window / minimum pairs |
| `CORR_DECIMALS` | 3 | 9 | decimals stored for correlation matrices (as built; the plan named the rounding, not a constant) |
| `PCA_ROLL_WEEKS` / `PCA_COMPONENTS` / `PCA_STD_MIN` | 52 / 10 / 1e-8 | 9 | rolling window / scree components published / minimum standard deviation of a complete-case series (as built; the plan named 52, 10 and 1e-8 only in prose) |
| `REF_WEEKS` / `REF_MIN_PAIRS` | 156 / 104 | 4 | window and minimum finite weekly pairs of `sharpe_3y` and `beta_3y` (as built; METRICS 5) |
| `CARRY_DAYS` | 7 | 4 | alignment carry: a cash fixing or a close is carried to at most this many following calendar days, never past the series' last value (METRICS 1, Carry) |
| `MARKET_MIN_COVERAGE` | 0.5 | 4 | a date enters the market axis when this share of the member funds has a close |
| `MARKET_MA_SHORT` / `MARKET_MA_LONG` | 50 / 200 | 4 | breadth averages, observation rows; slice 5 reuses `MARKET_MA_LONG` for `vs_200d` |
| `MARKET_HILO_ROWS` / `MARKET_DISP_ROWS` | 252 / 21 | 4 | new-high / new-low window; dispersion return horizon, observation rows |
| `MARKET_CORR_ROWS` / `MARKET_CORR_MIN` | 63 / 50 | 4 | implied-correlation window and the common dates required in it |
| `HILO_MIN_CLOSES` | 100 | 7 | closes required in the last year for `from_52wh` / `from_52wl` |
| `EW_LAG` / `EW_BACK_LONG` / `EW_BACK_SHORT` / `EW_WARN` | 20 / 63 / 21 / 4 | 7 | early-warning comparison offsets (observation rows before the last row) and the flag threshold (published as `params.ew_warn`); the 50 and 200 averages are `MARKET_MA_SHORT` / `MARKET_MA_LONG` |
| `RRG_FAST` / `RRG_SLOW` / `RRG_LAG` / `RRG_MIN_WEEKS` | 10 / 40 / 4 / 60 | 7 | rotation EMA spans, momentum lag, finite weekly levels required |
| `RRG_TAIL_GROUPS` / `RRG_TAIL_FUNDS` / `ROT_DECIMALS` | 156 / 52 / 4 | 7 | weekly points published per group / fund; decimals stored |
| `SIGNAL_H` / `SIGNAL_WIN` | 13 / 156 | 10 | canonical signal horizon (weeks of excess return) and z / percentile window (weekly values); published in `params.signal` |
| `SIGNAL_Z_MIN` / `SIGNAL_PCT_MIN` / `SIGNAL_SD_MIN` | 104 / 52 / 1e-12 | 10 | finite values required for z and for the percentile; sample SD at or below the last has no z (as built: METRICS 8 said "sd <= 0", which floating-point noise defeats) |
| `SIGNAL_CUSTOM_FLOOR` / `SIGNAL_CUSTOM_SHARE` | 26 / 0.67 | 10 | custom window minimum `max(floor, round(share * window))` (METRICS 8, was literals) |
| `SIGNAL_DECIMALS` | 4 | 10 | decimals stored in the weekly signal arrays of the series files |
| `RANK_MONTHS` | 3 | 7 | trailing horizon of the leadership ranks, calendar months |
| `RESID_MIN_WEEKS` | 104 | 13e | weekly rows required to fit an ETF's equity-factor residual, and overlaps per residual correlation pair of the residual map |
| `PCA_AR_COMPONENTS` | 5 | 13e | components in the absorption ratio `ar5` |
| `FACTOR_SHORT_MIN` | 26 | 13e | complete weekly rows for the 12M and YTD group factor fits |
| `HAC_LAGS` | 4 | 13e | Newey-West lags of the factor fits |
| `T_CRIT` | 1.96 | 13e | two-sided 95 % critical value for alpha intervals and filled markers; published as `params.t_crit` |
| `SEASON_PATH_MIN_CLOSES` | 5 | 13h | closes required in a seasonal-path window |

Display constants (slice 13): pixel sizes, counts and scale factors used only for drawing. They live in the named browser file, not in `constants.py`, and change no published number.

| Constant | Value | Slice | File | Meaning |
|---|---|---|---|---|
| `TIP_MAX_LINES` | 16 | 13a | `web/app.js` | series listed in a crosshair tooltip before `+N more` |
| `SNAPSHOT_BARS` | 24 | 13b | `web/tabs/overview.js` | signal-snapshot bars; above it the top and bottom halves |
| `HM_SCALE` | `{tight: 0.6, auto: 1, wide: 1.8}` | 13b | `web/tabs/heatmap.js` | colour-cap multipliers of the Colour scale control |
| `HM_YEARS_FLOOR` / `HM_INV_VOL_FLOOR` / `HM_MOVE_FLOOR` | 0.5 / 0.02 / 0.003 | 13b | `web/tabs/heatmap.js` | size floors for years, inverse volatility and size of the move |
| `GRID_MAX_HEIGHT` | 720 | 13b | `web/tabs/heatmap.js` | performance-grid scroll height, px |
| `LEAD_RING_LABEL_MAX` | 16 | 13c | `web/tabs/leaders.js` | most visible groups whose rings carry a label |
| `LEAD_PRIO_NEAR_HIGH` | -0.03 | 13c | `web/tabs/leaders.js` | distance from the high above which a dot gets the label bonus |
| `Z_HEAT_CAP` | 2.5 | 13d | `web/tabs/groups.js` | colour cap of the excess z heat |
| `DISP_LOG_RATIO` | 50 | 13g | `web/tabs/market.js` | Auto dispersion scale switches to log above this multiple |
| `FUND_PEERS` | 14 | 13g | `web/tabs/fund.js` | bars in Most correlated |
| `LAB_HIST_BIN` | 0.25 | 13h | `web/tabs/lab.js` | z histogram bin width |


There is no benchmark constant: benchmarks come from `settings.toml` (section 2b). `GROUP_PALETTE_SIZE` (12) stays in `web/app.js` because it must match `--g1`..`--g12` in `theme.css`. The slice-1 `constants.py` is laid out in comment groups: `metrics` (STALE_DAYS to ANNUAL_WEEKS), `universe` (SHORT_NAME_MAX), `build` (MIN_PRICED_FRACTION), `fetch`, `sample`; each constant carries a one-line comment.

## 13. Invariants worth a test

Slice 1 has exactly six tests (`tests/test_slice1.py`, class `Slice1`, stdlib `unittest`, synthetic data only); slice 1b adds two (`tests/test_slice1b.py`, class `Slice1b`). Fixtures: `ROOT` is the repository root, `UNIVERSE` is `ROOT/data/universe.csv`; `daily(start, end)` is a business-day series `100.0 + 0.01 * calendar days since start`; `small_universe(dir, n=40)` writes the first 40 universe rows to `dir/u.csv` and returns `(universe path, prices path, data.js path, frame)` with prices at `dir/var/prices.csv` and data.js at `dir/web/data.js`. A test that runs the CLI copies `u.csv` to `dir/data/universe.csv` and runs `cli.main` with `dir` as the working directory (slice 1) or with `--project dir` and `dir/universe.csv` (from slice 1b).

1. `test_universe_rejects_bad_rows`: a duplicate ticker, a 25-character `short_name`, a lower-case currency and a group with two families each raise `ValueError` (one CSV per case); the committed universe loads with more than 400 rows.
2. `test_sample_is_deterministic_and_labelled`: `make_sample` on the first 40 universe rows, written twice with `prices.write(..., "synthetic")`, gives byte-identical files and `prices.source` is `synthetic`.
3. `test_anchor_rule_nulls_only_what_history_cannot_reach`: `compute(daily("2025-02-03", "2025-12-26"), "2025-12-26")` (about 10.5 months) has `r_1y` null with `why["r_1y"] == "no_history_to_anchor"`, `r_6m` not null and status `short`; an empty series gives `r_3m` null.
4. `test_weekly_gap_removes_volatility_but_not_returns`: `daily("2022-01-03", "2025-12-26")` with `2025-04-07` to `2025-06-27` dropped (60 business days, Monday to Friday) has `vol_1y` null (`insufficient_weekly_returns`) and `r_1y` not null; the gap-free series has `vol_1y`.
5. `test_failed_build_keeps_published_file`: in a temp folder holding `data/universe.csv` (40 rows) and sample prices, `cli.main(["build"])` run with that folder as the working directory returns 0; after overwriting `var/prices.csv` with `not a prices file`, it returns 2, `web/data.js` is byte-identical and no `web/data.js.tmp` remains.
6. `test_output_contract`: built from 40 rows with `build(upath, ppath, out)`: line 1 is `// GENERATED by etf-arcade build. Do not edit.`; `schema == 1`, `synthetic is True`, `source == "synthetic"`; no `NaN` in the text; `groups[0]` has `name family index n`; exactly one fund is `no_data` and one `stale`; for every fund `set(why)` equals the set of null `FIELDS`.

Slice 1b tests:

7. `test_constants_override_and_bad_settings_fail`: a project folder with 40 universe rows and a `settings.toml`: `[constants] STALE_DAYS = 3` loads and, after `project.apply`, `metrics.STALE_DAYS == 3` (the override reaches modules that imported the name); an unknown constant, a string and a bool for an int constant each raise `ValueError` from `project.load`; `project.apply({})` restores 14 (also called in `tearDown`).
8. `test_project_build_writes_theme_and_project_object`: a project with `[project] title = "MINE"`, `subtitle = "sub"`, `currency = "EUR"` and a `theme.css`; in a temp working directory `cli.main(["--project", "proj", "sample"])` and `... "build"` return 0; `web/project.css` equals the theme text; `data.js` `project` equals `{"title": "MINE", "subtitle": "sub", "currency": "EUR", "universe_label": "ETFs", "dir_name": "proj"}`; `proj/var/prices.csv` exists.

## 14. Review marks (slice 12)

Human corrections applied at build time only; `prices.csv` and `universe.csv` are never edited. Order: read prices, apply marks, screen, metrics.

File `DIR/review/<name>.json`, written by "Export marks" (`review_<YYYYMMDDTHHMMSS>.json`, UTC) or by hand:

```json
{"schema": "arcade.review/1", "created": "2025-11-02T09:15:00Z", "marks": [
  {"ticker": "AAA", "mode": "broken_price", "from": "2025-03-03", "to": "2025-03-07", "note": "bad print", "t": "2025-11-02T09:14:10Z"},
  {"ticker": "BBB", "mode": "exclude"},
  {"ticker": "CCC", "mode": "move_group", "group": "Gold"},
  {"ticker": "AAA", "mode": "clear", "clears": ["broken_price"]}]}
```

`ticker`, `mode` required. `from`, `to` (`YYYY-MM-DD` or null = open end; both null = whole series) only on `broken_price`; `group` only, and required, on `move_group`; `clears` only on `clear`; `note`, `t` optional on every mode.

**Precedence.** Files sort by name (export names sort by time), marks in file order; a later mark replaces an earlier one for the same `(ticker, mode)`, different modes combine. A `move_group` to the fund's own group undoes a move. A `clear` mark removes the earlier effective marks of its ticker for the modes in `clears` (a non-empty list of `broken_price`, `exclude`, `move_group`; absent means all three) and is not itself an effective mark; marks after it apply again.

| Mode | Effect | `data.js` |
|---|---|---|
| `broken_price` | Closes from `from` to `to` inclusive are dropped before the screen and metrics, never filled (METRICS section 12). | Whole series: status `review_broken`, metrics null, `why` `review_broken_price`. Range: status unchanged. |
| `exclude` | Own metrics kept; left out of every aggregate, market statistic, correlation, PCA and rotation. | Status `review_excluded`; `groups[].n` counts the rest. |
| `move_group` | `group` and `family` become the target's. `groups[]` order and colours come from the unmoved universe. | `group_original`. |
| `clear` | Cancels earlier marks of the ticker (see precedence). | Counted in `review.counts.cleared`. |

With `exclude` and a whole-series `broken_price`, the status is `review_excluded`, metrics null.

`review` is always present: `files`, `marks` (effective marks, each with `file` and, for `broken_price`, `closes_dropped`), `counts` `{marks, broken_price, exclude, move_group, closes_dropped, overridden, cleared}` (zeros without a folder; `overridden` counts marks replaced by a later mark of the same ticker and mode, `cleared` the effective marks removed by `clear` marks) and `warnings` (a list of `{file, ticker, reason}`, empty without any).

**Validation.** Each error exits 2, names file and mark, keeps the old `data.js`:
1. File unreadable, not JSON, or top level not an object.
2. `schema` is not `arcade.review/1`, or `created` is not a string.
3. `marks` missing or not a list; a mark not an object or with an unknown key.
4. `mode` not one of the four.
5. `from`/`to` not null and not a real date, `from` after `to`, or present on another mode.
6. `move_group` without a `group` in the universe; `group` on another mode.
7. `note` or `t` not a string; `clears` not a non-empty list of distinct modes from `broken_price`, `exclude`, `move_group`, or present on another mode.

**Warnings, not errors.** A mark whose `ticker` is not in `universe.csv` (the universe may have changed since the file was written): the mark is ignored, listed in `review.warnings` with its file, shown in the Data tab and printed by `review-status`; the build still publishes. Also not errors: a range with no closes, a fund without prices, empty `marks`, no `review/` folder.

As built (slice 12), details the text above leaves open:
- `review.load(DIR/review, universe)` returns the published `review` object itself; `review.apply` returns `(prices, funds)` (funds = the universe frame, with a `group_original` column only when a fund moved) and fills `closes_dropped` on each broken mark and `counts.closes_dropped`. `review.py` imports only the standard library (frames are used by duck typing). `build` takes the review folder from the project (`DIR/review`); `build()` without a project object has no review (empty object).
- Effective marks keep only `ticker, mode`, `from`/`to` (broken_price, both always present, null = open), `group` (move_group), `note`, `t`, `file`, and `closes_dropped` (broken_price). A `move_group` to the fund's own group removes the earlier move and is not kept. There is one effective `broken_price` mark per ticker: a later one replaces the earlier (two ranges for one fund cannot coexist).
- `status`: `review_broken` when a `broken_price` mark leaves the fund without closes (a whole-series mark, even for a fund with no rows, or a range that took every close); `why` is then `review_broken_price` for every metric. `review_excluded` wins over it; its `why` stays whatever the metrics gave. `as_of` is the latest close after the marks and before the screen, so the screen and the metrics agree on it; the hard stop still counts tickers before the marks; `quality.counts.unpriced` counts universe tickers without rows in `prices.csv` itself (a whole-series mark does not make a fund unpriced); `quality.funds[].n_obs`, gaps and flags describe the closes left after the marks (a removed range shows as a gap or break, METRICS 12). A review that removes every close in the file fails the build.
- `groups[].n` counts funds that are not `review_excluded` (after moves); `build.members(funds)` is the one member filter (status not `review_excluded`, `review_broken`, `no_data`) that the aggregates of slices 4 to 9 must use.
- `data.js` validation adds: `review` keys `files marks counts warnings`, marks only for universe tickers, `sum(groups[].n)` equals the funds that are not `review_excluded`, and `first_date` null only for `no_data`, `review_broken` or a `review_excluded` fund without closes (`series_file` exactly when closes exist).

`review-status` runs the same loader and prints each effective mark with its effect (`dropped 5 closes`, `fund excluded`, `moved from X to Y`), the warnings, and the counts (no close counts without `prices.csv`). It writes nothing; exit 2 on an invalid file, 0 otherwise.

## 15. Package, modules and CLI (slices 1 and 1b)

Build exactly this layout. Python names listed here are relied on by later slices and by the tests.

```
pyproject.toml            setuptools project, see below
etf_arcade/
  __init__.py             one-line docstring
  __main__.py             import sys; from .cli import main; sys.exit(main())
  constants.py            section 12
  universe.py             load(path) -> DataFrame; COLUMNS; TICKER (compiled regex)
  prices.py               source(path) -> str; read(path, universe) -> (frame, issues); write(path, frame, src); replace(tmp, path)
  project.py              load(directory=".") -> namespace; apply(overrides); DEFAULT_META; DEFAULTS
  sample.py               make_sample(tickers, groups, end=None, years=None)
  fetch.py                fetch_prices(tickers, years=FETCH_YEARS) -> (frame, failed)
  metrics.py              compute(series, as_of) -> dict; FIELDS; RETURNS
  quality.py              screen(prices, issues=(), tickers=None) -> (cleaned prices, flags); FLAGS (slice 11)
  review.py               load(review_dir, universe) -> review object; apply(prices, funds, review) -> (prices, funds); status_lines(review, universe, counted) (slice 12)
  build.py                build(universe_path, prices_path, out_path, proj=None) -> data dict
  cli.py                  main(argv=None) -> int; OUT = "web/data.js"
tests/__init__.py (empty), tests/test_slice1.py, tests/test_slice1b.py
```

`pyproject.toml`: build backend `setuptools.build_meta` (`setuptools>=61`); project `etf-arcade`, version `0.1.0`, description "A neon-arcade ETF screener that runs from static files.", `requires-python = ">=3.11"`, license `Apache-2.0`, dependencies `pandas` and `numpy`; optional dependency group `fetch = ["yfinance"]`; `[tool.setuptools] packages = ["etf_arcade"]`. Installing is optional: every command runs from the repository root with `python -m etf_arcade`.

Module contracts:
- `universe.load(path)`: section 2. Returns the stripped string DataFrame with `COLUMNS` in file order, index reset. Reads with pandas (`dtype=str`, `keep_default_na=False`, UTF-8).
- `prices.read`, `write`, `source`, `replace`: section 3. `read` calls `source` first, so a bad header raises before anything else.
- `project.load(directory)`: resolves paths with forward slashes (`os.path.normpath`, then `/`): universe `DIR/universe.csv`, else `DIR/data/universe.csv`; prices `DIR/var/prices.csv`; settings `DIR/settings.toml` (absent = all defaults; a TOML syntax error becomes `ValueError("<path>: <message>")`); theme `DIR/theme.css` or `None`. Checks, each raising `ValueError`: unknown top-level section (allowed: `project constants benchmarks cash`, from slice 13a also `factors`), `[project]` keys and types and the `title`/`currency` rules (section 2b), unknown or wrongly typed constant, benchmark, cash and (slice 13a) factors shape (section 2b, one error naming the entry). Returns `SimpleNamespace(dir, universe, prices, theme, meta, constants, benchmarks, cash)` (slice 13a appends `factors`, `{}` if absent) where `meta` is the `[project]` values over `DEFAULT_META` plus `dir_name` (`os.path.basename(os.path.abspath(DIR))`), `constants` the validated overrides, `benchmarks` the list ([] if absent) and `cash` the table ({} if absent). `DEFAULTS` is `{name: value}` for every upper-case name in `constants.py` as shipped; type checks are `type(value) is type(default)`, with an int accepted and converted for a float default.
- `project.apply(overrides)`: sets `{**DEFAULTS, **overrides}` on `constants` and on every loaded `etf_arcade.*` module that has the same attribute (so `from .constants import X` copies see the override, and calling `apply({})` restores the shipped values). The CLI calls it right after `project.load`, before any module that reads a constant at call time is used.
- `metrics.compute`, `FIELDS`, `RETURNS`: section 7. Pure functions of the series and `as_of`; no I/O, no module state.
- `build.build(...)`: section 10. Steps: `uni = universe.load`; `src = prices.source`; `frame, issues = prices.read`; hard stop; `by_ticker = {ticker: Series(adj_close indexed by date)}`; `as_of = frame.date.max()`; per universe row the fund object is the six universe columns followed by `compute(...)` (or `compute(empty Series)` for a ticker with no rows); `groups` from first appearance with `n` = member count; assemble the top level in the order of section 9; serialise; validate; print warnings; write `project.css` (the project's `theme.css` text or `/* no project theme */\n`) then `data.js`. `proj=None` means the default meta with `dir_name` `default`.
- `cli.main(argv)`: `argparse` program `etf_arcade`; global `--project DIR` (default `.`) placed before the subcommand; subcommands (required): `sample [--force]`, `fetch [--tickers A,B] [--years N]`, `build`. The whole body runs inside one `try`; any exception prints `error: <message>` to stderr and returns 2 (an argparse usage error exits 2 by itself); success returns 0. Order: `proj = project.load(DIR)`, `project.apply(proj.constants)`, then the command. Imports of `sample` and `fetch` happen inside their branches (after `apply`, so they see the overrides).
  - `sample`: unless `--force`, if the prices file exists and `prices.source(file) != "synthetic"` raise `ValueError("<file> holds <source> prices; pass --force to overwrite them with SYNTHETIC data")` (a malformed existing header also fails here); load the universe; `prices.write(file, make_sample(tickers, groups), "synthetic")`; print `wrote <file> (SYNTHETIC)`.
  - `fetch`: tickers = the `--tickers` list (comma-separated, stripped) or the whole universe; any name outside the universe raises `ValueError("not in universe: A, B")`; `fetch_prices(tickers, --years or FETCH_YEARS)`; `prices.write(file, frame, "yfinance")`; print `wrote <file> (yfinance): <ok> of <requested> tickers` plus `; no data for <failed, comma-separated>` when any failed.
  - `build`: `build(proj.universe, proj.prices, OUT, proj)`; print `wrote web/data.js: <n> funds, as_of <date>, source <source>, project '<title>'`.
  - `review-status` (slice 12) is added to the same parser.
- All commands run from the repository root; `OUT` is relative to the working directory.

Slice 1 builds everything above except the items that belong to slice 1b: `project.py`, the `--project` option (slice 1 reads `data/universe.csv` and `var/prices.csv` directly), the `proj` parameter of `build`, `web/project.css`, the `project` field of `data.js` and its validation, and `, project '<title>'` in the build message. Slice 1b adds exactly those.
