# UI reference (browser side)

Build spec and contract for everything under `web/` (`STATUS.md` says which slices exist; sections 15 and 16 are the exact spec of slice 1). Product intent lives in `docs/PRODUCT.md`; the `data.js` schema in `docs/reference/DATA.md` (section 9) and metric definitions in `docs/reference/METRICS.md`. This file says how the browser code is built. Plain JavaScript, no bundler, no build step, no server: the page must work when `web/index.html` is opened from disk (`file://`).

## 1. Hard constraints

- Classic `<script>` tags only. ES modules and `fetch()` of local files fail on `file://`. Each file is an IIFE; `app.js` defines the single global `window.ARCADE` and tab files attach to it.
- Script order in `index.html`: Google Fonts link, `theme.css`, `app.css` (slice 2 on), `project.css` (slice 1b on), then `app.js` first, `vendor/d3.min.js` (slice 3 onward), `tabs/*.js`, `review.js` (slice 12), `data.js` last. `app.js` boots on `DOMContentLoaded`, which fires after every script has run, so every tab has registered and `window.ARCADE_DATA` is set. Extra data files (per-ticker series, slice 3) are loaded by injecting a `<script>` tag, never by `fetch`.
- `data.js` is the only data input: `window.ARCADE_DATA`. The browser never recomputes what `etf_arcade/metrics.py` publishes, except the filtered-scope aggregates in section 6.
- Colours, fonts, glows and spacing exist only as CSS custom properties in `theme.css`. JavaScript reads them through `ARCADE.css(name)` (a cached `getComputedStyle(document.documentElement).getPropertyValue`, cache cleared on each redraw). No colour literal appears in any `.js` file or inline style.
- Dark only. No scanlines, curvature, flicker or animation. The only motion is 150 ms hover/focus transitions, disabled under `prefers-reduced-motion`.
- Missing is `null`, shown as an em dash. A value is never filled, forward-filled across a gap, or defaulted to zero.

## 2. Files and responsibilities

| File | Responsibility |
|---|---|
| `index.html` | Static shell (section 16): header, settings slot (slice 2), tab bar, one `<main id="view">`, drawer slot (slice 2), tooltip node. |
| `theme.css` | Tokens on `:root` (including `--g1` … `--g12` for the group palette), base elements, component classes (`.panel`, `.row`, `.pill`, `.sw`, `table.data`, `.empty`; slice 2 adds `.strip`, `.picker`, `.drawer`; `.chip` arrives with slice 6), breakpoints. Single source for colour, type, spacing. |
| `app.js` | Tab registry and router, state, error and placeholder panels. Shared helpers: slice 1 `ARCADE.registerTab`, `.f`, `.css`, `.color`, `.swatch`, `.why`, `.statusTitle`, `.statusPill`, `.empty`, `.search`, `.table`, `.tabState`, `.openFund`; slice 2 persistence, hash codec, redraw scheduler, `.set`, `.scope`, `.fund`, `.tip`, drawer; slice 3 `.fit`, `.series`, `.lineChart`; slice 4 `.benchmark`, `.ref`; slice 6 `.legend`; slice 12 hooks into `.review`; slice 13 `.METRICS`, `.weekly`, `.wret`, `.roll` (13a), `.monthGrid` (13g) (`docs/reference/PARITY.md`). Exact slice-1 contract: section 15. |
| `review.js` | Slice 12: review mode (section 14). Attaches `ARCADE.review`; tabs call it, it never draws a tab. |
| `tabs/<id>.js` | One tab. Registers itself; owns no globals. |
| `data.js` | Generated, gitignored. Never edited by hand. |

`app.js` owns the canonical tab list (section 3), so order, labels and placeholders do not depend on which tab files are loaded.

## 3. Tabs and the module contract

Canonical list, in tab-bar order, with the slice that builds each:

| id | Label | Slice |
|---|---|---|
| overview | Overview | 5 |
| heatmap | Heatmap | 5 |
| leaders | Leadership | 7 |
| wall | Chart wall | 6 |
| groups | Groups | 6 |
| rotation | Rotation | 7 |
| correlation | Correlation | 9 |
| pca | Structure & PCA | 9 |
| risk | Risk & return | 8 |
| lab | Signal lab | 10 |
| screener | Screener | 1 |
| calendar | Calendar | 7 |
| market | Market | 4 |
| fund | Fund | 3 |
| data | Data | 11 |

A tab file calls `ARCADE.registerTab({id, label, slice, render})`. `id`, `label` and `slice` must equal the canonical row (`app.js` logs a console error and ignores a mismatch). Unregistered tabs render a placeholder panel: `Not built yet — slice N`.

`render(container, data, state)`:

- `container` is an empty `<section>` that `app.js` has just cleared; `data` is `ARCADE_DATA` already scoped to the visible groups (section 6); `state` is the shared state object (section 4).
- It builds the whole tab synchronously and is idempotent. There is no incremental update: any change redraws the active tab. Allocate nothing outside `container`; attach listeners to nodes inside it, so clearing the container is the cleanup.
- It must not throw for empty or partial data. Unexpected exceptions are caught by `app.js`, which shows a `Failed to render <label>` panel with the error text and keeps the tab bar usable.
- It reads settings from `state`, never from `localStorage` or the URL directly, and writes changes through `ARCADE.set(patch)`.

## 4. State, persistence and deep links

`state` is one plain object:

| Key | Values | Persisted in |
|---|---|---|
| `period` | `r_1w r_1m r_3m r_6m r_ytd r_1y r_3y_ann`, labelled 1W 1M 3M 6M YTD 1Y 3Y (default `r_3m`; `r_1w` from slice 13a) | localStorage |
| `history` | `YTD 1Y 2Y 3Y 5Y 10Y MAX` (default `3Y`; `10Y` from slice 13g) | localStorage |
| `names` | `short` (default) or `full` | localStorage |
| `benchmark` | id of a `data.benchmarks[]` entry; default the first one (slice 4; absent without benchmarks) | localStorage |
| `hidden` | array of hidden group names | localStorage |
| `review` | boolean, default false (slice 12) | localStorage |
| `tab` | tab id | hash |
| `fund` | ticker or null; from slice 13g also `g:<group>` or `b:<benchmark id>`, a Fund-tab subject that never opens the drawer | hash |
| `series` | calendar series id | hash |
| `q`, `sort`, `x`, `y`, `m` | active tab's search text, sort (`key:dir`), axis metrics, custom metric | hash (active tab) and in-memory per-tab state |

localStorage uses `arcade.settings.v1`, holding `{period, history, names, hidden}` (slice 4 adds `benchmark`, slice 12 adds `review`), and from slice 12 `arcade.review.v1:<dir_name>` for pending review marks (section 14). Nothing else is stored. Read and write inside try/catch; with storage blocked everything still works for the visit. Unknown or invalid saved values fall back to the defaults (a saved `benchmark` that is no longer in `data.benchmarks` becomes the first one). Hidden names that no longer exist in `data.groups` are ignored, never errors.

Hash format: `#tab=<id>&fund=<ticker>&series=<id>&q=<text>&sort=<key>:<asc|desc>&x=<metric>&y=<metric>&m=<metric>`. Slice 1 reads and writes `tab` and `fund` only; slice 2 adds the rest. Values use `encodeURIComponent`. Unknown keys are ignored; an unknown or missing tab falls back to the default tab (`DEFAULT_TAB`), `screener` until slice 5 and `overview` from slice 5; invalid values are dropped. Only keys that differ from the default are written. Settings (period, history, names, benchmark, hidden, review) are never in the hash, so a shared link does not overwrite a reader's preferences.

Rules that must survive any interaction:

- Changing chart history changes only `history`. It never resets `sort`, `x`, `y`, `m`, `q`, `fund` or `series`.
- Switching the benchmark (slice 4) changes only `benchmark`: the current fund, `series`, group filter, searches, sorts and custom metrics (`x`, `y`, `m`) all stay, and every reference-dependent number, line and label is redrawn against the new benchmark. A sort on a reference-dependent column stays on that column.
- After a rebuild the page resolves the latest publication: a reload re-reads `data.js`, `ARCADE.series` reloads series files whose `generated` differs, and a saved or linked `fund`, `series` or `benchmark` that no longer exists falls back as described above instead of failing.
- Changing the return period re-sorts a table only while its sort is the implicit default (the period column, descending). Once the reader clicks a header, that sort persists across period and history changes.
- Chart-history window: start = `as_of` minus N calendar months (1Y 12, 2Y 24, 3Y 36, 5Y 60, 10Y 120); `YTD` starts at the last close on or before 31 December of the prior year; `MAX` at the first close. Every fund uses the same window, so a stale fund simply ends early. Charts rebase to 100 at the first close inside it.
- Switching tab keeps the departing tab's `q/sort/x/y/m` in memory only and restores them on return (`ARCADE.tabState(id)`). Every other per-tab control (chart kind, window, the correlation selection `tabState.sel`) lives there too, never in the hash. `fund` and `series` are shared across tabs.
- A reload after a rebuild keeps what the hash and localStorage hold and shows the latest `data.js`.
- Hiding a group never clears `fund`: a deep-linked ETF still opens (looked up with `ARCADE.fund(ticker)`, which reads the unscoped data), with a pill `Outside current group filter`. Lists, charts and aggregates honour the filter.
- Tab changes and fund open/close push a history entry; sorting, searching, setting changes and redraws use `replaceState`, so Back moves between tabs and funds only. `hashchange` and `popstate` re-read the hash and redraw.

`ARCADE.set(patch)` merges, persists, updates the hash and schedules one redraw. `ARCADE.set` also takes the active tab's `q sort x y m` and `series`; `set(patch, {redraw: false})` updates state and hash without a redraw (a tab that updates itself in place, such as the Screener search). `ARCADE.openFund(ticker)` and `ARCADE.closeFund()` push a history entry. `ARCADE.table` rows carry `data-key` (the row key) so focus can return to the trigger, and accept `label(row)` (the row's `aria-label`) and `tip(row)` returning `{header, lines}` for the shared tooltip. Redraws are coalesced with `requestAnimationFrame`. Window resize triggers a redraw 150 ms after the last event, only when the width bucket (`<600`, `<900`, `>=900`) or the container width changes by more than 24 px. `document.fonts.ready` triggers one redraw so text measurement uses the loaded fonts.

## 5. Page structure and layout

- Order: header (title, as-of line, settings from slice 2), tab bar, view, drawer.
- No warning banners; the as-of line names the source (`synthetic` or `yfinance`). If `data.js` is missing or `schema` is not `1`, the view shows an error panel with the commands to run: `python -m etf_arcade sample && python -m etf_arcade build` (the build step names the project folder when one was given).
- As-of line: `<subtitle · >As of <as_of> · <source> · <n> <label> · <g> groups` (`<label>` is `project.universe_label`; `—` for unknown parts). When the coverage cutoff `C` (DATA.md section 11) is before `as_of`, the line ends with `· groups to <C>`; every group-level and cross-sectional panel ends at `C` and its footnote names that date. From slice 13g the line reads `<subtitle · >As of <as_of> · <source> · vs <benchmark label> · <priced> of <n> <label> priced, <stale> stale · <g> groups · groups to <C>` with an explanatory hover on the cutoff part, and a one-sentence market brief sits under the settings strip on every tab (PARITY 13g.1, 13g.2).
- `#view` has `max-width: 1800px`, centred, 16 px gutters. Panels sit in `.row` grids: `.row` is one column, `.row.two` is two columns at `>=900px` and one below. Never more than two analytical panels per row. Dense comparisons (matrices next to tables, several charts) stack as separate rows.
- Grid children get `min-width: 0`. The page never scrolls horizontally at 360 px. Wide tables and heat grids scroll inside their panel (`.scroll`, `overflow: auto`, sticky header and first column).
- Panel anatomy (see `docs/skeleton.html`): title (display font, accent, glow), kicker (cyan, small caps), optional controls line, legend, chart area, footnote. Every panel states its scope in the footnote when it is not the whole visible universe.
- Glow (`--glow-accent`, `--glow-cyan`) is allowed only on titles, the active tab, hovered or focused controls, and the panel border on hover. Body text, tables, chart text and numbers in cells never glow.

## 6. Group scope

The group filter applies to every tab through one function, `ARCADE.scope(data)`, called by `app.js` before `render`. It returns `data` with `funds` and `groups` reduced to visible groups.

- Lists, tables, charts, heatmaps and counts use the scoped data only.
- Aggregates that `build.py` published for the full universe (PCA shares, rolling absorption, fixed loadings, breadth series) are never relabelled as filtered. Under a filter the panel shows the standard note `Unavailable for a filtered group selection. Restore all groups to view this full-universe analysis.` Quantities the browser can compute exactly from visible rows (counts, medians, a sub-matrix of pairwise correlations) are recomputed, and the footnote says `Recomputed for N visible <label>`.
- Group colours come from the full group order: `ARCADE.color(group)` returns the token name `--g<n>` with `n = index % 12 + 1` using `groups[].index`, so hiding a group never recolours the others. Benchmarks and cash use `--chart-ref`.
- Empty scope (every group hidden): every panel shows `No <label> in the selected groups. Enable a group in settings.` and controls stay usable.

## 7. Names, formats and tooltips

- Universe label: UI text never hard-codes "ETFs". `ARCADE.label` (written `<label>` throughout these docs) is `data.project.universe_label` (default `ETFs`) and every plural noun for the funds in headers, count lines, footnotes, empty states, search labels and ARIA text uses it; singular wording is the neutral word "fund" (`Pick a fund`).
- Names: `names === 'short'` shows `short_name`; `full` shows `name`. Hover or `title` text is always `<name> (<ticker>)`. The overview heatmap labels each taxonomy group; individual ETF tiles carry no text.
- Number formats (`ARCADE.f`): non-finite or null gives `—`; the minus sign is U+2212. Return and volatility fields are decimal fractions in `data.js`; `pct(v, d=1)` multiplies by 100, is signed and adds `%`; `pctu` is unsigned; `num(v, d=2)`; `int` with thousands separators; `date` as `2 Jan 2026`; `idx` as `1,234`, `123`, or `12.3` by magnitude.
- Benchmark (slice 4): the settings strip holds a `Benchmark` select listing `data.benchmarks[].label` (hidden when there are none); `ARCADE.benchmark()` returns the selected entry (state `benchmark`, else the first). "The reference" in these docs means that selected benchmark. Every reference-dependent number, tile or chart is labelled `vs <label>`, and `ARCADE.ref(row, field)` returns `row[field][state.benchmark]` (`null` when the object or key is missing). With no benchmark configured, reference-dependent items show `—` with the `reference_missing` text.
- Cash (slice 4): wherever the cash series or a number depending on it is shown, the name comes from `data.cash.label`; the UI hard-codes no cash name. Without `data.cash` a cash-dependent number shows `—` with the `cash_missing` text and a cash card says `No cash series`.
- Null reasons: `funds[i].why` maps each null field to a reason code. `app.js` holds one code-to-text table (`WHY`; codes and texts are listed in DATA.md section 9). Any cell or tile showing `—` carries that text as `title`; with no code it reads `Not available`. A `why` entry that is an object (a reference-dependent field) is read with `[state.benchmark]`.
- Status: `funds[i].status` drives pills labelled `ok`, `short`, `stale`, `no data` (`stale`: warning colour, with `last_date` and `days_stale` in the title; `short`: info; `no_data`: negative); slice 12 adds `review_excluded` (`Excluded`), `review_broken` (`Prices removed`) and `Moved from <group>` (from `group_original`). The UI holds no staleness threshold of its own; `build` decides.
- Tooltip: one `#tip` node; `ARCADE.tip.show(evt, header, lines[{name, value, color?, bold?}])`, `.hide()`. Offset 14 px from the pointer, flipped at the right and bottom edges, never leaving the viewport, `pointer-events: none`, hidden on scroll, tab change and Escape. Text uses `--text` on `--raised`, no glow. Touch: tap shows, tap elsewhere hides.
- Anything shown by a tooltip is also reachable by keyboard: focusable rows and tiles (`tabindex="0"`) carry the same text in `title`/`aria-label`, and Enter opens the detail drawer.
- Detail drawer (slice 2): activating an ETF anywhere opens a side panel (bottom sheet below 900 px) with name, ticker, group, family, currency, history range, status, all metrics with null reasons, and a link to the Fund tab. Escape or the close button closes it and returns focus to the trigger.

## 8. Legends, labels and fitting

- Legend = row of chips (`button.chip` with swatch and label), 32 px high on desktop, 44 px under `(pointer: coarse)`. Click toggles a series; hover or focus previews it; `Reset` restores all (disabled when all active). Inactive chips are muted and struck through, never removed; their marks fade to 12 % opacity instead of disappearing. `aria-pressed` on every chip; the row is `role="group"` with a label.
- Legend selection is per panel, kept across redraws in `tabState` (every redraw builds a new container, so a WeakMap keyed by container would lose it), never persisted, and never changes data or statistics.
- `ARCADE.fit(text, maxPx, font)` measures with a canvas context using the element's computed font and returns the longest prefix plus `…` that fits, or `''` if even `…` does not fit. Every truncated label keeps the full text in `<title>` (SVG) or `title` (HTML). Chart axis and end labels must fit inside their panel; reserve margin from measured label width, capped at 40 % of the panel width.
- Every chart coloured by group carries the shared chip legend (`A.legend`): Leadership, Risk & return, PCA loadings, Groups, Overview group indices. Scatters add a label mode `All / Some / None` (default `Some` = the budget) kept in `tabState`. With more groups than palette colours the legend is the only way to decode a colour, so it is never omitted.
- Bars and tiles that show a signed value are coloured by sign (`--positive` / `--negative`), never by group colour; the group shows as a swatch beside the label.
- Axes: x ticks at most `floor(width / 80)`, formatted by span (years above 2 years, `Mon YY` below); y ticks share one format per axis (d3 `tickFormat` of the scale). Axis titles sit outside the tick labels with their own margin and never overlap them.
- Name cells show the short name with a group swatch; the ticker is in hover text (the Screener has its own ticker column).
- Self-check: with `check=1` in the hash, `app.js` runs `ARCADE.checkLabels()` after each render. It compares the bounding boxes of every SVG `text` in the view with each other and with their panel, and writes `<pre id="check">overlaps N, clipped M</pre>` at the end of the view. Every headless dump uses `&check=1` and must show `overlaps 0, clipped 0`.
- As built (13a): `ARCADE.checkLabels()` judges overlap on the ink box (text box shrunk to font size + 1 px, because a line box is taller than the font and labels are spaced by font size + 2), clipping on the full box against the panel and the own svg, 1 px tolerance; with `check=1` it runs 300, 1200 and 2600 ms after each render so lazily drawn charts are included, and the flag survives hash rewrites. Compact line charts (Chart wall cards) have no end labels and no hover highlight.
- End labels read `<label> <last finite value>` and hovering or focusing a line or its end label fades the other series to 12 % (from slice 13a, PARITY 13a.12).
- Wide tables and heat tables: a table or heat map that does not fit a half-width panel is placed full width; number columns come right after the name column, long text columns (group names) last, truncated with the full text in hover; headers are short (`t`, `AR`, `Res vol`). A heat table draws row and column names only when a cell is at least 13 px; otherwise names are in hover only.
- Return periods are defined once in `app.js` (`A.PERIODS`: id, label, field); every tab reads that list, none keeps its own, so adding a period changes one place.
- No label may overlap another: end labels are sorted by y and pushed apart by `fontSize + 2`; scatter labels follow a budget (about `W·H/18000`, between 8 and 28) with priority to selected, then largest, and skip a label that would collide.

## 9. Charts (d3 v7, from slice 3)

- d3 is vendored at `web/vendor/d3.min.js` with its licence file beside it. Charts are inline SVG built inside `render`, sized from `container.clientWidth` (minimum 240) at call time.
- Colours only from tokens through `ARCADE.css`: grid `--chart-grid`, axis lines and tick marks `--chart-axis`, tick and axis label text `--muted`, reference lines `--chart-ref` (dashed `3 3`), series from `--g1` to `--g12`, `--accent`, `--accent-2`, positive and negative from `--positive`/`--negative`. Diverging scale: `--negative` to `--panel` to `--positive`, scaled by the 95th percentile of absolute values, never by an outlier. Text on a coloured cell is `--on-accent` or `--text`, chosen by luminance for 4.5:1.
- Never rely on colour alone: signed values show `+`/`−`; bars also differ in direction.
- Line charts: slice 3 builds the one shared `ARCADE.lineChart` (axes, crosshair, end labels with dodge, at most 18); every later line chart reuses it. It uses `d3.line().defined(d => d != null)` and also breaks a line where consecutive dates are more than `LINE_GAP_DAYS` (`data.params.line_gap_days`) calendar days apart, so a gap stays a gap (never interpolated, never bridged). Rebase to 100 at the first close inside the history window (section 4). Linear y by default; log only for growth of 100 over 5Y, 10Y or MAX. Axes: 3–6 y ticks, x ticks about every 90 px, edge labels anchored inward. A reference line at 0 or 100 where meaningful. Series width 1.5–2 px; no glow, shadow filters or gradients on marks.
- Crosshair: one vertical line, tooltip lists the series at that date sorted by value, nulls shown as `—`; from slice 13a at most `TIP_MAX_LINES` (16) lines, then `+N more`, and series switched off in the legend are not listed.
- Bars start at zero with labels outside the bar end. Heatmaps and matrices: SVG up to about 2,000 cells; canvas above that (device-pixel-ratio scaled) with the same hover contract.
- Every chart container has `role="img"` and an `aria-label` summarising title and range.
- Redraw is a full rebuild, so a chart must be cheap enough (under about 100 ms for 520 ETFs). Tabs with many panels build off-screen panels lazily with `IntersectionObserver` (`rootMargin: 600px`).

## 10. Tables

`ARCADE.table(container, {columns, rows, sort, key, selected, onActivate})` returns `{setRows}`:

- Column: `{key, label, render(row, td), value(row), title(row), cls(row), align, sortable=true, narrow=true, sticky}`. Numeric columns use the mono font with `font-variant-numeric: tabular-nums`, right-aligned; text columns left. Sign-coloured cells use the `pos` and `neg` classes; diverging heat cells arrive with slice 5.
- Sort: the header is a `<button>` inside `<th aria-sort>`. First click on a text column sorts ascending, on a numeric column descending; second click flips. Null and empty values sort last in both directions. Ties break by ticker. Sort state is the `{key, dir, user}` object the tab keeps in `ARCADE.tabState`, not in the DOM; `user` is true once a header was clicked.
- Search: `ARCADE.search(rows, q)`, case-insensitive substring over ticker, name, short name, group and family; the tab debounces input 150 ms; the count line reads `<shown> of <total> <label>`.
- Rows are keyed by ticker, `tabindex="0"`, activated by click, Enter or Space. First column and header are sticky inside the scroll area.
- Cells: the ETF cell shows the name per `state.names` plus a small muted ticker; null cells show `—` with the reason title. Columns marked `narrow: false` hide under 600 px. Table body text never glows.

## 11. States

| Situation | Treatment |
|---|---|
| No `data.js`, wrong `schema` | Error panel with the exact commands; tab bar still works. |
| Tab not built | `Not built yet — slice N` placeholder panel (muted, dashed border; text from section 15). |
| No rows for the selection | `.empty` text in the panel: `No data for this selection`. |
| Series with no point in the range | `No data in this range`. |
| Sample too short for a statistic | `Insufficient data: needs N, has M` plus the rule (for example `2Y PCA needs 104 complete weeks across at least 8 <label>`). Never a zero or a partial estimate. |
| Stale ETF | Pill from `status === 'stale'`; metrics are shown against the ETF's own last date, never rolled forward. |
| Series file from another build | `Reload: data was rebuilt` (`ARCADE.series`). |
| Series file loading (slice 13a) | `Loading <ticker>…` at the panel's final height. |
| Series file failed to load (slice 13a) | `Failed to load series/<file>. Rebuild, then reload.`; other panels still draw. |
| Short history | Windows longer than the history are `null` with the reason; the ETF stays listed. |
| Group filter makes an aggregate invalid | Standard unavailable note (section 6). |

## 12. Narrow screens and accessibility

- Below 900 px: all rows are one column; settings collapse into a disclosure; the drawer becomes a bottom sheet; charts keep a minimum height of 220 px.
- Tab bar: at 1400 px and wider all fifteen tabs fit one row; below that the bar scrolls horizontally inside itself and never wraps to a second row. The Benchmark select is wide enough for its longest label (full label also in `title`).
- Below 600 px: the tab bar scrolls horizontally inside itself and keeps the active tab in view; `narrow: false` table columns hide; legend chips use touch size.
- Tab bar: `role="tablist"`, buttons `role="tab"` with `aria-selected` and roving `tabindex`. ArrowLeft/ArrowRight move and activate with wrap, Home/End jump to the ends. The view has `role="tabpanel"` labelled by the active tab.
- Focus: every interactive element has a visible `:focus-visible` ring (2 px `--accent-2` plus `--glow-cyan`). Sort headers, chips, rows and tiles are keyboard operable. Escape closes tooltip, drawer and settings panels and returns focus.
- Contrast of `--text` and `--muted` on `--panel` stays at least 4.5:1; do not introduce lower-contrast text colours for convenience. Hit targets are at least 32 px (44 px on coarse pointers).

## 13. Per-tab implementation notes

Common to all: scoped data in; `period`, `history`, `names` and (slice 4) `benchmark` honoured; ETF activation opens the drawer (slice 2 onward) or sets `fund` and opens the Fund tab; hover shows `<name> (<ticker>)` plus the tab's metrics. Field names are those of `data.funds[i]` and `data.groups[j]` in DATA.md. Fields added in later slices (`series_file`, `benchmarks`, `cash`, `quality`, correlation and PCA fields) are specified in DATA.md and are added by the slice that builds them. Fund and reference-dependent wording follows section 7 (`<label>`, "fund", `vs <benchmark label>`). Slice 13 adds controls and panels to every tab; its per-tab spec is `docs/reference/PARITY.md`, which takes precedence where a note below or an as-built note differs.

1. **overview (5).** KPI strip (ETF count, share positive over the period, best and worst group, median return, stale count; display-font numbers). Strategy heatmap: treemap of equal tiles per ETF coloured by the period return, taxonomy group labels only, tiles without text, tooltip on hover, click opens the drawer. Group leaderboard (horizontal bars with a benchmark reference line). Group index lines rebased to 100 over `history`. One or two panels per row.
2. **heatmap (5).** Treemap of ETFs with a colour metric (the six period returns, relative return 3M or 1Y, 12-1 momentum, distance from the 200-day average, current drawdown) and a size metric (equal, volatility); group-by `group` or `family`; tile text only when the tile is large enough, full name in hover. Below it a groups-by-horizons table with diverging cells.
3. **leaders (7).** Scatter of gain from 52-week low against distance from 52-week high (log x), group rings, priority labels. Early-warning cards per group (ratio to benchmark with its 200-day average). Participation lines (published share above 200-day average) with linked x-axes; unavailable under a group filter. Needs series and benchmarks.
4. **wall (6).** Small-multiple line cards, one per ETF, sectioned by group; controls for chart kind (growth, drawdown, rolling return), sort metric and direction, card size; lazy-built; shared or per-group y-scales; click opens the drawer.
5. **groups (6).** Equal-weight group indices (growth, relative to benchmark, drawdown); group statistics table with switchable column sets (returns, risk, breadth); per-group panels with member lines in the group colour. Legend chips route to every chart on the tab.
6. **rotation (7).** Relative-rotation scatter of groups (ratio x, momentum y, quadrant names, tails, end-week slider); rank bump chart over months; leadership table with rank change. Rows stack; tail length is an input.
7. **correlation (9).** Pairwise matrix (windows 1Y daily, 3Y weekly) as canvas heat with clustered or group ordering; selecting an ETF (`tabState.sel`) shows closest and furthest peers as bars; pairs table (highest, lowest). Under a group filter the matrix is a sub-matrix, keeping the clustering order.
8. **pca (9).** Explained-variance bars, rolling PC1 share line, loadings scatter coloured by group with outlines for at most 16 groups. The 2Y fixed fit uses a 104-week complete-case sample and needs at least 8 complete ETFs, otherwise `Insufficient data`. Full-universe fit: unavailable under a group filter.
9. **risk (8).** Scatter with x and y metric selectors (volatility, return, drawdown, beta, Sharpe) and colour by group; up-versus-down capture scatter with a diagonal; per-group dot strip with medians; drawdown watch table. Labels follow the budget in section 8.
10. **lab (10).** Choose an ETF or group, horizon (4, 13, 26, 52 weeks) and window. Six stacked panels: indexed price, excess return, rolling mean with bands, z-score, percentile, distribution of z with today's marker. Then a cross-section bar chart of today's z for every ETF. Statistics use full history before slicing to `history`.
11. **screener (1).** Slice-1 tab and the first user of the table of section 10 (exact spec: section 15.6). Control: search box (slice 1 also has a group select that slice 2 replaces with the shared picker). Columns: fund, group, family, `r_1m r_3m r_6m r_ytd r_1y r_3y_ann vol_1y maxdd_1y`, `first_date`, `last_date`, status pill. Default sort is the current period (`state.period`, `r_3m` until slice 2 adds the setting), descending. Returns are sign-coloured; nulls `—` with reasons; count line `<shown> of <total> <label>`; row activation sets `fund` in the hash (drawer from slice 2).
12. **calendar (7).** Month-by-year return grid for the selected ETF or group (`series`); seasonality bars (average, share positive); calendar-year table by group with `*` where a year has fewer than 12 months. Monthly returns come from consecutive month-end closes only; a missing month stays blank.
13. **market (4).** Six line cards over `history`: breadth (above 50-day and 200-day averages), new 52-week highs and lows, implied correlation, dispersion, reference drawdown (of the selected benchmark, computed in the browser from its series file), cash rate (the `[cash]` series, labelled with `cash.label`, never group-filtered, gaps kept). Below them non-clickable member squares, best to worst. Under a group filter breadth and implied correlation show the filtered-selection note (nothing is recomputed); from slice 13g every card drawn from `market` does. No zero cash assumption: without `[cash]` the cash card says `No cash series`.
14. **fund (3).** Fund select grouped by group (visible groups plus the current ETF), header (name, ticker, group, family, status pills), KPI tiles (slice-1 metrics plus since inception and drawdowns; relative return, Sharpe and beta from slice 4), price and drawdown charts over the history window (daily drawdown computed in the browser; the selected benchmark as a dashed line from slice 4), peers table of the visible same-group members sorted by the period. No fund in the hash gives `Pick a fund`.
15. **data (11).** History timeline (one bar per ETF from first to last date, coloured by group), coverage and quality table (status pill, four flag chips `stale` `gap` `spike` `break`, first, last, days stale, observations) with search, method notes showing the live windows, publication record (generated, source, synthetic). It shows exactly what `build` flagged.

### 13.6 As built (slice 10)

- Signal lab: series (`series` hash value `f:<ticker>` or `g:<group>`; a bare name is a ticker, else a group; empty falls back to `fund`, then to the first visible group), horizon in the hash `x` (4, 13, 26, 52 weeks, default 13) and z window in `y` (`1`, `2`, `3`, `5` years or `all` for an expanding window, default `3`), all kept in `tabState("lab")`, so a history or benchmark change keeps them. `fund=` in the hash opens the drawer as on every tab; `series=` shows the lab without it.
- One controls panel with five tiles (excess, z, percentile, z rank, as-of date), then five stacked time panels (index on a log scale with the reference dashed, excess return as a green and red area, rolling mean with one and two SD bands, z with guides at 0, +-1 and +-2, percentile with guides at 10, 50 and 90), the z histogram (0.5 bins over the chart history, today's bin filled, label `today z .., percentile ..`) and the cross-section. The time panels use one x range, one left margin and one crosshair with a tooltip of all five values; the histogram has its own x axis (z).
- Statistics run over the full history of the series before the chart history cuts the display. For a fund at the canonical horizon and window the panels read the published `rel_13w` and `z_13w` arrays; otherwise, and for groups, the browser computes them (`ARCADE.lab`). A series that cannot give a z shows `Insufficient data: needs N, has M` in the z-based panels (N = finite values required, M = the most the series ever had in a window).
- Cross-section: one bar per visible group (the group's canonical z from its `group_index` at `C`, computed in the browser), sorted high to low, labelled with the short group name, the selected group (or the selected fund's group) outlined and the selected fund drawn as a marker at its own z; click loads the group into the series control. One bar per fund is never drawn (hundreds of 3 px bars are unreadable). It reads `data.funds`, so it honours the group filter; funds without a z are counted in the footnote. It always shows the canonical signal (see METRICS 8).
- `app.css` gains `.chart-area.lab-chart { min-height: 0 }` (the 240 px minimum of `.chart-area` left blank space under 170 px panels).

### 13.5 As built (slice 9)

- Correlation: window in the hash key `m` (`corr_3y_weekly` default, `corr_1y_daily`), order (`cluster` or `group`) and the selection `sel` in `tabState("correlation")`. A history or benchmark change keeps all three. A panel row holds the two selects, then the matrix as a full-width panel (a canvas of at most 900 px beside a side note with the colour key; below 1000 px of panel width the note drops under the canvas), then a two-panel row (peer bars, pair lists). The selected fund defaults to `fund` when it is in the matrix, else the first fund in display order. Peer bars: 12 highest and then 8 lowest of the remaining peers (fewer than 20 peers: the rest of them, no overlap), domain fixed to -1..1. Pair lists use `ARCADE.table` with their own sort objects (`sortHi`, `sortLo`), default value descending / ascending, ties by ticker; clicking a row selects the first fund of the pair. Blank pairs are drawn in `--raised` and say so on hover. Under a group filter the matrix is the sub-matrix of the visible funds in the published (or group) order, pair lists and peers are recomputed from it and the footnote says `Recomputed for N visible <label>`; fewer than 8 visible funds shows `Insufficient data: needs 8, has M`. Colour: `--negative` to `--panel` to `--positive` over -1..1 (`ARCADE.scaleKey` for the key); the group band is 8 px.
- PCA: rolling lines (`ARCADE.lineChart`, PC1 to PC3 shares over the chart history) and scree bars share a row, the loadings scatter (PC1 on x, PC2 on y, domain from the data, label budget as in section 8, Delaunay hover) is full width. Hull outlines are drawn when `data.groups.length` is 16 or fewer; with every group visible a project with more than 16 groups never gets outlines (a group filter makes the whole tab unavailable), and the footnote says so. Omitted funds (no complete sample) are named in the footnote and counted. Under a group filter every panel shows `Unavailable for a filtered group selection...`. A non-ok status shows `Insufficient data: needs 8, has M` or `Fixed PCA unavailable` with the reason.

As built (13e): Correlation rows are controls, matrix, (Group correlation, Correlation regime), (Rolling correlation to the reference, Peers with the ETF select), Pairs; a window without a matrix (3Y residual without `[factors]`, too few ETFs) shows its message in the matrix and peer panels while the group panels still draw. Pairs is one table (Set, Pair, Group A, Group B, rho) of the 15 highest and 15 lowest pairs, Across groups by default (`tabState("correlation").pairs`). The group heat table is an SVG with names only when a cell is at least 13 px wide. Structure & PCA rows are (Rolling components, Scree), Colour and labels, (Residual map, Loadings), (Alpha forest, Alpha against residual risk with its table), Group vs computed cluster; state in `tabState("pca")`: `color labels hulls off load fw aoff asort`. The first legend (`off`) drives both scatters, the second (`aoff`) both alpha panels.

### 13.4 As built (slice 8)

- Risk & return: axis ids in the hash `x` and `y` (defaults `vol_1y` and `r_3y_ann`; the choices are `vol_1y r_3y_ann maxdd_1y beta_3y sharpe_3y`), kept in `tabState("risk")` so a history change or benchmark switch does not reset them. The two selectors sit above a two-panel row (risk scatter, capture scatter); the group strip and the drawdown watch table are full-width rows below.
- Scatters: linear axes from the 1st to 99th percentile padded by 6 %; a mark beyond is held at the edge and counted in the footnote, hover shows the true value. The capture scatter uses one shared domain on both axes, so its dashed diagonal is `y = x` (x down capture, y up capture). Labels: the selected fund, then the marks farthest from the median, within the budget of section 8. Funds with a null on an axis are not plotted and are named (first six) in the footnote, all in its hover text.
- Strip: one row per visible group in family order (row height 18 px), dots with a fixed vertical jitter, white median tick, the Y metric of the scatter; rows with no value keep their label.
- Drawdown watch: `ARCADE.table`, default sort `dd_now` ascending (deepest first) until a header is clicked, bars (`cellbar neg`) scaled to the deepest value of the column.

### 13.3 As built (slice 7)

- Tab switches keep the shared `fund` and `series` (`app.js` `tabHash`): a click or arrow key writes `#tab=<id>&fund=..&series=..`. The drawer therefore stays open across tabs while a fund is set; Escape or its close button clears it.
- Leadership: scatter with log x (gain from the 52-week low, ticks at 1.05 to 20), y 0 at the top, translucent corner zones, one ring per group with three or more funds (median position), labels for the selected fund and then rings by size within the budget, nearest-fund hover (14 px), click opens the fund. Early-warning cards (one per visible group, most warnings first, red from `params.ew_warn`) draw the ratio of the group index to the selected benchmark's series over the history window with its 200-row mean; the count is the published `ew_tells`. Participation is `market.above_200d` with dashed 20% / 80% lines.
- Rotation: `tabState('rotation')` holds `tail` (0 to 26 weeks, default 8) and `end` (index into `rotation.dates`); changing either redraws the chart and the table in place, not the tab. Axes are symmetric around 100 and fixed from the 99th percentile of every shown week (a point beyond is held at the edge). The bump chart re-ranks `rank_trail.ret` among the visible groups; the rank columns of the table are the last point and the points one and three entries earlier (month ends, labelled with their month). The pill is the quadrant at the chosen end week.
- Calendar: `series` values are `b:<benchmark id>`, `g:<group>`, `f:<ticker>`; empty is the selected reference (so a benchmark switch moves it). The Year column compounds the months with data; part years carry `*` and `n of 12` in the hover.

### 13.1 As built (slice 5)

- `app.js` gains `A.textOn(bg)` (text colour by contrast), `A.diverging(values, {center})` (`{scale, cap, center}`, 95th percentile of absolute distance from the centre), `A.treemap(container, o)` (one tile per fund in blocks, null colour gives a neutral tile, tile text only when `o.labels` and the tile is at least 64 x 26 px, returns `{omitted}`), `A.scaleKey(container, ds, fmt)` and an optional `width` on a `lineChart` series. `DEFAULT_TAB` is `overview`.
- Overview: review-excluded funds are left out of the treemap and every panel; funds with no return for the period stay as grey tiles. Leaderboard is an SVG bar chart (group colour, value label outside the bar, dashed reference line from `benchmarks[].returns`). Group indices are rebased in the browser to the first value inside `history`; the heavier dashed reference line comes from the benchmark series file; end labels go to the reference and the highest and lowest groups. Top 12 / Bottom 12 keep their sort in `tabState("overview").sortTop|sortBottom`.
- Heatmap: colour in hash/tab state `m` (absent = the shared return period), size in `x` (absent = equal), group-by in `y` (absent = group). The table sort follows the period until a header is clicked (`tabState("heatmap").sort`). Funds without a size value (1Y volatility) are left out of the treemap and counted in the footnote.

### 13.2 As built (slice 6)

- `lineChart` gains options `compact` (small card: no end labels, 3 y ticks, margins 4/6/16, any height from 60 px), `yDomain [lo, hi]` (fixed scale, lines clipped to the plot) and per series `key` (written as `data-k` on the line and its end label), `opacity` and `off` (drawn at 12 %, no end label, left out of the y scale and the tooltip; the scale uses them only when every series is off).
- `A.legend(container, {items:[{key,label,color,title}], off, onChange, scope, label})`: chips `button.ctl.chip.lg` with `aria-pressed`, a `Reset` button (disabled when nothing is off). The switched-off keys are the caller's array. Because every redraw builds a new container, the selection is kept in `tabState` (Groups: `tabState("groups").off`), not in a WeakMap keyed by container; it is never persisted. Hover or focus previews one key through the `pv` / `pv-on` classes on `scope` (all `[data-k]` marks fade to 15 %, the hovered one shows in full, also when switched off). `A.lazy(node, build)` calls `build` once when the node is within 600 px of the viewport.
- Wall: controls kept in `tabState("wall")` (`kind growth|drawdown|roll`, `size S|M|L|XL`, `yscale group|shared`, `sort {key,dir,user}`, which is also the hash `sort`); the default sort is the shared period, descending, until a control is used. Each group is a section with a sticky header; its grid is a placeholder of the estimated height until it nears the viewport, then all its cards are created. Card chart height by size 70 / 100 / 140 / 190 px. Growth rebases the daily closes to 100 at the first close inside the history; drawdown and rolling slice the weekly arrays (the drawdown peak predates the window). Card value: the value of the current sort metric, formatted with its label (default the shared return period), so the badge and the sort always agree. The card line is drawn in `--accent-2` (never in a group colour, which could read as a loss or a gain); the group colour marks the card's left edge. Y scale `group`: min and max over the section's cards, `shared`: 1st to 99th percentile over every card (lines beyond are clipped), which loads every series file once. Funds without a series show the null reason in the card.
- Groups: legend row, then three full-width rows (growth, relative to the reference, drawdown) of the group lines, then the statistics table (column sets in `tabState("groups").set`, switching resets the sort), then one lazy panel per visible group (member lines at 45 % opacity, bold index, dashed reference). Groups switched off in the legend stay listed in the table (dimmed) and their panels are dimmed, never removed.

## 14. Review mode (slice 12)

`web/review.js` (classic IIFE after the tab scripts) attaches `ARCADE.review` from `ARCADE_DATA.review`; `app.js` owns the toggle and `state.review`.

- **Toggle.** A `role="switch"` labelled `Review` ending the settings strip, set through `ARCADE.set({review})`, never in the hash. Off: no action renders; built marks still show as pills and in the queue.
- **Pending marks.** A JSON array of file-format marks under `localStorage` key `arcade.review.v1:<project.dir_name>` (try/catch; blocked storage keeps them in memory). On load a pending mark equal to an active one is dropped. They change no number, only add a `Pending` pill and a queue row.
- **API.** `.pending()`, `.active(ticker)`, `.add(mark)` (replaces a pending mark of the same ticker and mode), `.undo(ticker, mode)` (pending only; for a built mark the button is disabled and its `title` names the file and says to use `Clear`), `.clear(ticker, clears)` (adds a pending `clear` mark; `clears` optional, default all modes), `.actions(container, ticker, {pickRange})`, `.exportFile()`, `.pills(fund)`, `.title(fund)`.
- **Actions.** `.review-bar` (`role="group"`): `Whole series`, two `input type="date"` with `Add` (disabled while `from` is after `to`), `Exclude`, an unscoped group `select` with `Move`, `Undo` (pending marks) and `Clear` (enabled when the ticker has a built mark: adds a pending `clear` mark that cancels the built marks at the next build; its `title` names the files it cancels).
- **Placement.** Fund tab: the bar under the header; `pickRange` is a `d3.brushX` on the price chart filling the date inputs (snapped to closes); typed inputs are the keyboard path. Data tab: the queue uses `ARCADE.table`; row actions omit `pickRange`; a `Review warnings` list under it shows `review.warnings` (marks for tickers not in the universe, ignored by the build).
- **Export.** `Export marks (N)` downloads `review_<YYYYMMDDTHHMMSS>.json` (UTC; `{schema, created, marks}` of pending marks whose ticker is in `data.js`, others skipped and reported) via Blob and `a[download]`, which works from `file://`. Disabled with none pending.
- **Keyboard.** Native `button`, `select` and `input` only; focus stays on the control after an action; a polite `aria-live` line reports `Pending: exclude AAA` or `Undone`; queue rows are focusable and Enter opens the Fund tab.
- **As built.** `.pills(fund)` returns `Moved from <group>` and `Pending`, plus `Prices removed` for a built range mark; a fund whose status is `review_excluded` or `review_broken` shows `Excluded` / `Prices removed` as its status pill (`A.statusPill`, hover = `.title(fund)`), so they are not repeated. `.exportButton()` returns the `Export marks (N)` button (disabled with none pending), `.texts(ticker)` the queue's mark texts, `.version()` a counter that changes with the pending marks (the drawer re-renders on it). `.actions(...)` returns `{setRange(from, to)}`; `pickRange: true` only adds the hint text, the Fund tab passes `onBrush: setRange` to `ARCADE.lineChart`, which adds a `d3.brushX` over the plot (the overlay takes over the crosshair hover). Add is enabled when at least one date is set and `from` is not after `to` (an empty end is open). After an action the tab redraws and focus returns to the same control (the first enabled one when that control is now disabled). Pending marks and the drafts of the date inputs survive redraws. `ARCADE.table` ignores a click inside a `button, input, select, label` and a key whose target is not the row, so row actions never open the fund; queue rows go to `#tab=fund&fund=...`. `tr.is-excluded` is added by `ARCADE.table` to every row with status `review_excluded`.
- **Styling.** `.review-bar`, `.pill.review`, `.pill.pending`, `tr.is-excluded` (muted); existing tokens only.

## 15. `app.js` contract (slice 1)

One classic script, one IIFE, `"use strict"`. It defines `window.ARCADE` (written `A` below, created with `window.ARCADE = window.ARCADE || {}`) and boots on `DOMContentLoaded`. Slice 1 builds everything in this section; later slices extend it where their steps say.

### 15.1 Module state and constants

- Constants: `DASH = "—"` (U+2014), `MINUS = "−"` (U+2212), `GROUP_PALETTE_SIZE = 12` (must match `--g1`..`--g12`), `COMMANDS = "python -m etf_arcade sample && python -m etf_arcade build"`, `DEFAULT_TAB = "screener"`.
- `TABS`: the canonical array of section 3 (`{id, label, slice}` in tab-bar order). `STATUS = {ok: "ok", short: "short", stale: "stale", no_data: "no data"}`. `WHY`: code to text exactly as in DATA.md section 9 (all codes except `review_broken_price`).
- Private state: `data` (the `window.ARCADE_DATA` object, read at boot because `data.js` loads after `app.js`), `impl` (registered tab modules by id), `mem` (per-tab memory), `cssCache`, `groupIndex` (group name to `groups[].index`), `state = {period: "r_3m", history: "3Y", names: "short", hidden: [], tab: DEFAULT_TAB, fund: null}` (slice 1 keeps the settings at their defaults in memory; slice 2 persists them).

### 15.2 Public helpers

| Helper | Behaviour |
|---|---|
| `A.f.pctu(v, d=1)` | Not a finite number: `—`. Else `(v * 100).toFixed(d)`; if the rounded value is below zero prefix `−` and drop the ASCII minus; append `%`. |
| `A.f.pct(v, d)` | `pctu`, with a leading `+` unless the text is `—`, starts with `−` or rounds to zero. |
| `A.f.num(v, d=2)` | `toFixed(d)` with `−` for negatives; `—` for non-finite. |
| `A.f.int(v)` | Rounded, `en-US` thousands separators, `−` for negatives; `—` for non-finite. |
| `A.f.date(iso)` | `YYYY-MM-DD` to `2 Jan 2026` (day without leading zero, three-letter English month); anything else `—`. |
| `A.f.idx(v)` | `1,234` at 1000 or more, `123` at 100 or more, else `12.3`; `—` for non-finite. |
| `A.label` | The plural noun for the funds: `data.project.universe_label`, else `ETFs`; set at boot. Every UI string that names the funds in the plural uses it (section 7). |
| `A.css(name)` | `getComputedStyle(document.documentElement).getPropertyValue(name).trim()`, cached in `cssCache` (cleared at the start of every render). |
| `A.color(group)` | Token name `--g<n>`, `n = (groupIndex[group] or 0) % 12 + 1`. |
| `A.swatch(group)` | `<span class="sw g<n>">`. |
| `A.why(row, key)` | `WHY[row.why[key]]`, else `WHY.no_data` when `row.status === "no_data"`, else `Not available`. |
| `A.statusTitle(f)` | `stale`: `Last close <date>, <days_stale> days behind the latest date`; `short`: `Less than 12 months of history`; `no_data`: `WHY.no_data`; else `Enough history, up to date`. |
| `A.statusPill(f)` | `<span class="pill <status>">` with the `STATUS` text and `title = statusTitle(f)`. |
| `A.empty(title, message)` | `<div class="empty">` with `<strong>title</strong>` and, when given, `<span>message</span>`. |
| `A.search(rows, q)` | Trim and lower-case `q`; empty returns `rows`; else keep rows whose ticker, name, short name, group and family (joined with newlines, lower-cased) contain it. |
| `A.tabState(id)` | The per-tab memory object (`mem[id]`, created on first use). |
| `A.openFund(ticker)` | `location.hash = "#tab=" + state.tab + "&fund=" + encodeURIComponent(ticker)` (adds a history entry; the `hashchange` listener redraws). |
| `A.registerTab(t)` | Accepts `{id, label, slice, render}` only when `id`, `label` and `slice` equal the canonical `TABS` row and `render` is a function; otherwise `console.error("registerTab: not the canonical tab", t)` and the tab is ignored. |
| `A.table(container, opts)` | Section 15.4. |

### 15.3 Shell, router and error panels

- `renderHeader()`: `title = data.project.title` (else `ETF ARCADE`); sets `#brand` text and `document.title`; `#asof` text is `<subtitle · >As of <A.f.date(as_of)> · <source> · <funds.length> <A.label> · <groups.length> groups`, with `—` for each unknown part (no data or wrong schema); the subtitle part is absent when `project` is missing or its subtitle is empty.
- `buildTabs()`: `#tabs` gets `role="tablist"` and one `<button type="button" id="tab-<id>" role="tab">` per `TABS` entry. Click sets `location.hash = "#tab=<id>"` plus the current `fund` and `series` (as built, slice 7: both are shared across tabs, UI 4). ArrowRight/ArrowLeft wrap, Home/End jump; the key moves focus to the target tab and sets the hash.
- `markTabs(active)`: `aria-selected`, roving `tabIndex` (0 on the active tab, -1 elsewhere) and `scrollIntoView({block: "nearest", inline: "nearest"})` on the active one.
- `panel(kicker, title, node)`: `<div class="row"><section class="panel">` with `<p class="kicker">`, `<h2>` (each only when given) and `node`.
- `render()` (runs at boot and on every `hashchange`): clear `cssCache`; parse the hash (`key=value` pairs split on `&`, values `decodeURIComponent` inside try/catch, invalid values dropped); the tab is the canonical entry for `tab`, else `DEFAULT_TAB`; `state.tab` and `state.fund` (`fund` or `null`) are set; `markTabs`; `#view` is emptied and gets `aria-labelledby="tab-<id>"`. Then, first match wins:
  1. no `data`: panel `Error` / `Cannot show data` holding `A.empty("No data yet", "Run this from the repository root, then reload:")`, a line break and `<code>COMMANDS</code>`;
  2. `data.schema !== 1`: the same panel with `A.empty("Data schema mismatch", "This page reads schema 1, data.js has schema <n>. Rebuild:")` and the commands;
  3. no registered module for the tab: panel titled with the tab label holding `<div class="placeholder"><strong>Not built yet — slice N</strong><span>This tab is planned; see docs/BUILD_SLICES.md.</span></div>`;
  4. otherwise a `<section>` is appended to `#view` and `module.render(section, data, state)` runs inside try/catch; on an exception log it, empty the section and show panel `Error` / `Failed to render <tab label>` with `A.empty(<error message>)`.
- Boot (`DOMContentLoaded`): `data = window.ARCADE_DATA || null`; with schema 1 fill `groupIndex` from `data.groups`; `renderHeader()`; `buildTabs()`; add the `hashchange` listener; `render()`.

### 15.4 `ARCADE.table`

`A.table(container, {columns, rows, sort, key, selected, onActivate})` appends `<div class="scroll"><table class="data">` to `container` and returns `{setRows(rows)}` (redraws the body only).

- Column fields: `key`, `label`, `render(row, td)`, `value(row)` (default `row[key]`, used for sorting and the null test), `title(row)`, `cls(row)`, `align` (`"right"` for numeric columns), `sortable` (default true), `narrow` (false hides the column under 600 px), `sticky` (the pinned first column).
- Header cell: `<th scope="col">` with classes `num` (right-aligned), `hide-narrow` and `first`; a sortable header holds a `<button type="button">`. Clicking the active sort column flips `sort.dir`; clicking another sets `sort.key` and the direction `-1` (descending) for a right-aligned column and `1` for a text column; both set `sort.user = true` and redraw. The active header carries `aria-sort="ascending|descending"`, the others none.
- Sorting: empty (`null`, `undefined`, `""`) values sort last in both directions; strings compare with `localeCompare` times `sort.dir`, numbers by difference times `sort.dir`; ties break by `String(row[opts.key]).localeCompare(...)`. A `sort.key` that matches no column leaves every value empty (ties by key).
- Body: one `<tr tabindex="0">` per row (class `sel` when `row[key] === selected`); cells carry the same `num`/`hide-narrow`/`first` classes; `render` fills the cell, else `String(value)` or `—`; a null value adds the class `nil` and the title `A.why(row, column key)`; `cls(row)` is appended for non-null values; `title(row)` applies when no title is set yet. With `onActivate`, click and Enter/Space (default prevented) call it with the row.

### 15.5 Registering a tab

A tab file is `(function(){ "use strict"; var A = window.ARCADE; ... A.registerTab({id, label, slice, render}); })();` with exactly the canonical `id`, `label` and `slice`. It adds its `<script>` tag to `index.html` between `app.js` and `data.js`.

### 15.6 Screener (`web/tabs/screener.js`)

`render(container, data, state)` builds one `div.row > section.panel`, in this order:
- `<p class="kicker">One row per fund</p>` and `<h2>Screener</h2>`.
- `div.controls` with `input type="search"` (placeholder `Search ticker, name, group`, `aria-label` `Search <A.label>`, value from memory) and a `select` (`aria-label` `Group`; the first option `All groups` with value empty, then one option per `data.groups[]` as `<name> (<n>)`). Slice 2 removes the select.
- The table box, then `<p class="footnote">`.

Memory (`A.tabState("screener")`): `q` (default empty), `group` (default empty), `sort` (default `{key: null, dir: -1, user: false}`); while `sort.user` is false the sort key is `state.period` and the direction `-1` on every render.

Rows: `data.funds` filtered by the select (`group` equal) and then `A.search(rows, q)`. The table uses `key: "ticker"`, `selected: state.fund` and `onActivate: f => A.openFund(f.ticker)`. After the table, a hidden `A.empty("No data for this selection")` is shown when no row remains.

Columns, in order (label, key, notes):
1. `Fund`, `name`, sticky; value and text are `short_name` or `name` per `state.names`, followed by a space and `<small>` ticker; the cell `title` is `<name> (<ticker>)`.
2. `Group`, `group`, `narrow: false`; swatch then the group name.
3. `Family`, `family`, `narrow: false`.
4. Returns `1M 3M 6M YTD 1Y 3Y ann.` for `r_1m r_3m r_6m r_ytd r_1y r_3y_ann`: right-aligned, `A.f.pct`, class `pos` above zero and `neg` below.
5. `Vol 1Y` (`vol_1y`) and `Max DD 1Y` (`maxdd_1y`): right-aligned, `A.f.pctu`.
6. `First date` (`first_date`, `narrow: false`) and `Last date` (`last_date`), `A.f.date`.
7. `Status` (`status`): `A.statusPill`.

Count line (footnote): `<shown> of <total> <A.label> · as of <date> · source: <source> · adjusted closes in listing currency`, with `total = data.funds.length`. The search input debounces 150 ms; the select applies at once; both update the table and the line without a full redraw and store `q` and `group` in memory.

## 16. `index.html` and `theme.css` (slice 1)

`web/index.html` is a static file, exactly:
- `<!doctype html>`, `<html lang="en">`, `<meta charset="utf-8">`, viewport `width=device-width, initial-scale=1`, `<title>ETF Arcade</title>` (`app.js` replaces it with the project title).
- Fonts (optional, the page falls back to system fonts offline): `preconnect` to `https://fonts.googleapis.com` and `https://fonts.gstatic.com` (crossorigin) and the stylesheet `https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@400;600&family=JetBrains+Mono:wght@400;600&family=Orbitron:wght@600;700;800&display=swap`.
- `<link rel="stylesheet" href="theme.css">`, then `<link rel="stylesheet" href="project.css">` (written by `build`; an HTML comment says so). Slice 1 writes the `theme.css` link only; slice 1b adds the `project.css` link; slice 2 adds `<link rel="stylesheet" href="app.css">` between them.
- Body: `div.wrap` holding `header.top` (`h1.brand#brand` text `ETF ARCADE`, `p.tagline#asof` text `As of —`), `nav.tabs#tabs[aria-label="Sections"]` and `main#view[role="tabpanel"]`.
- Scripts at the end of the body, in this order: `app.js`, `tabs/screener.js`, `data.js`. A missing `data.js` fails to load silently and the app shows the error panel. Later slices add their script tags in the order of section 1.

`web/theme.css` ships with this repository (tokens in `docs/THEME.md`, plus the base and component rules that slice 1 needs). Classes used by slice 1 and later: `.wrap`, `header.top`, `.brand`, `.tagline`, `nav.tabs` (active tab `[aria-selected="true"]`), `.row`, `.row.two`, `.panel`, `.kicker`, panel `h2`, `.legend`, `.chart-area`, `.footnote`, `.empty`, `.placeholder`, `.controls`, `input[type="search"]`, `select`, `.scroll`, `table.data` (`.num`, `.first`, `.hide-narrow`, header buttons, `tr.sel`), `.pos`, `.neg`, `.nil`, `.sw` with `.g1`..`.g12`, `.pill` with `.stale`, `.short`, `.no_data`, `.mono`. Later slices append their component classes (`.strip`, `.picker`, `.drawer`, `.chip`, `.review-bar`, ...) to the same file using tokens only.
