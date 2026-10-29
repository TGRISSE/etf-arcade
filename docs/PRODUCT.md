# etf-arcade: what the product looks like and does

Read this before building anything. It describes the finished website, tab by tab, so you can picture it. Colour, font and glow values live in `docs/THEME.md`; metric definitions in `docs/reference/METRICS.md`; the build order, steps and checklists in `docs/BUILD_SLICES.md`. This file contains no formulas and no code. Every feature below is built by exactly one slice step.

## 1. Pitch and audience

etf-arcade is a single-page, local, open-source ETF screener with a neon-arcade look. It covers one universe chosen by the project folder (the default example lists about 520 US-listed ETFs, `examples/europe` 141 UCITS ETFs; a private project brings its own list), each fund tagged with a **family** (for example "Equity" or "Fixed Income") and a **group** inside that family. It is for a person screening the whole ETF shelf: spotting what leads or breaks down, comparing ETFs inside a group, and opening any one ETF. It has no accounts, orders or portfolios; the only thing a reader can write is review marks (section 4.16), which are exported as a file and applied by the next build. The page opens straight from disk and shows one generated data file; until real prices are fetched it runs on clearly labelled SYNTHETIC prices.

Terms:

- **Group**: the finest classification; the colour key and the grouping unit.
- **Benchmark**: one of the series configured in the project's `settings.toml` (the default project: SPY, AGG and a 60/40 blend of them). The first is the default.
- **Reference**: the benchmark currently selected with the settings strip's Benchmark select (slice 4); relative numbers are measured against it and labelled `vs <benchmark label>` wherever they appear. Switching it never changes the open fund, filters, searches, sorts or custom metrics.
- **Cash**: the project's short-rate series named in `settings.toml` and shown under its own label (the default project: a T-bill proxy), used for Sharpe (slice 4). Without one, Sharpe is blank, never computed with zero cash.
- **Return period**: the period used for headline returns, colours and rankings.
- **Chart history**: how far back time-series charts reach.
- **As of**: the latest date for which the universe has prices.

## 2. The page at a glance

```
+--------------------------------------------------------------------------+
| ETF ARCADE                                                               |
| As of 26 Dec 2025 · synthetic · 520 ETFs · 59 groups                     |
| Overview  Heatmap  Leadership  Chart wall  Groups  Rotation  ...         |
| Groups [18 of 59]  Period [3M]  History [3Y]  Names [Short|Full]  Bench  |
| +----------------------------------------------------------------------+ |
| | PANEL TITLE (kicker: what one mark means)                  controls  | |
| | chart or table, legend chips, footnote                               | |
| +----------------------------------------------------------------------+ |
+--------------------------------------------------------------------------+
```

**Header.** The title with a magenta glow, then `As of <date> · <source> · <n> <label> · <g> groups` where `<label>` is the project's `universe_label` (default `ETFs`; an em dash until the data file loads). With sample data the source reads `synthetic`; there is no warning banner. From slice 13g the line also names the selected benchmark, how many ETFs are priced and how many are stale, and ends with `groups to <date>` when group outputs end earlier, with a hover explaining that cutoff.

**Market brief** (slice 13g). One muted sentence under the settings strip on every tab: what the reference returned over the period, how many groups beat it, which two led and which lagged, how correlated the shelf is and how much the first component explains, and how many groups show four or more early-warning signs. A part whose data is missing is left out; under a group filter it speaks of the visible groups and drops the correlation part.

**Tab bar.** Fifteen tabs in a fixed order: Overview, Heatmap, Leadership, Chart wall, Groups, Rotation, Correlation, Structure & PCA, Risk & return, Signal lab, Screener, Calendar, Market, Fund, Data. It scrolls sideways inside itself on narrow screens; Left/Right arrow keys move between tabs (wrapping).

**Settings strip** (slice 2). Identical on every tab: group picker, return period, chart history, name style, from slice 4 the benchmark select (hidden when the project has no benchmark) and, from slice 12, a Review toggle. Below 900 px it collapses to `Settings · <period> · <history>` (slice 13g).

**Panels.** Raised dark-violet cards with a thin violet border; on hover the border turns magenta with a soft glow. Every panel has a title, a one-line cyan kicker saying what one mark means, optional controls, the chart or table, legend chips and a muted footnote. At most two panels per row; one per row below 900 px.

**The look.** Dark only, neon accents used sparingly, glow only on titles, the active tab, hovered or focused controls and a hovered panel border; no scanlines, curvature, flicker or animation beyond 150 ms hovers. Readability beats style. Exact values are in `docs/THEME.md`.

## 3. Shared chrome and settings

### What persists

Settings are shared by every tab and saved in the browser, so they survive tab changes and reloads:

| Setting | Options | Default | Effect |
|---|---|---|---|
| Groups | any subset of groups | all | Which groups appear anywhere |
| Return period | 1W (slice 13a), 1M, 3M, 6M, YTD, 1Y, 3Y | 3M | Headline returns, heat colours, rankings, default sort |
| Chart history | YTD, 1Y, 2Y, 3Y, 5Y, 10Y (slice 13g), MAX | 3Y | How far back time-series charts reach |
| Names | Short, Full | Short | Fund label style |
| Benchmark | any configured benchmark (slice 4) | the first one | Which benchmark relative numbers, lines and rankings use |
| Review mode | Off, On | Off | Shows the review actions on the Fund and Data tabs (section 4.16) |

Chart history and the benchmark switch never reset a tab's own metric, sort or search, nor the open fund or calendar series. Search, sort and metrics chosen inside a tab are local to that tab, kept in memory while the page is open, and never change the shared settings.

### Group scope

The group picker opens a panel with search, "Show all", "Hide all" and a swatch checkbox per group, listed by family. Hidden groups disappear from every tab: table rows, chart series, selectors, rankings. The reference series is never hidden. The strip says which scope applies: "18 of 59 groups selected across all tabs" or "All 59 groups selected."

Full-universe analyses (Structure & PCA, every Market card built from market data, participation, co-movement and the correlation regime) never show their full-universe number as if recalculated for a filtered selection; they show: "Unavailable for a filtered group selection. Restore all groups to view this full-universe analysis." Tables, rankings and group indices are recomputed from visible ETFs.

### Deep links and history

The address hash holds the tab and the open ETF (on the Fund tab, from slice 13g, also a group index or a benchmark), for example `#tab=fund&fund=XYZ`, plus the active tab's search, sort and metric choices. Switching tab or opening or closing an ETF adds a history entry, so Back moves between tabs and ETFs; sorting, searching, settings and redraws replace the entry. Settings are never in the hash, so a shared link does not overwrite a reader's preferences.

### Hover and detail drawer

Every ETF mark (row, dot, tile, bar) shows on hover a compact card: full name, ticker, group and the value under the pointer. Clicking opens the **detail drawer**, a right-hand side panel (a bottom sheet on narrow screens) with name, ticker, group, family, currency, history range, status, every metric with the reason for each missing one, an "Open Fund tab" link and a close button (Esc closes). Group marks have hover text only. The drawer arrives in slice 2; before then, clicking an ETF sets it in the address hash.

Empty, insufficient and stale states are worded in `docs/reference/UI.md` section 11. Chart lines break at gaps and never drop to zero; two tickers are never joined into one series.

## 4. The tabs

Each tab is built by the slice named at the end of its section. Until a tab's slice is built, the tab shows a "Not built yet" placeholder.

---

### 4.1 Overview

**Purpose.** One-screen answer to "what is happening across the ETF shelf right now?"

**KPI strip.** Six tiles with display-face numbers: ETF count, share of ETFs up over the period, best group, worst group (name plus return), median ETF return, stale ETF count.

**Strategy heat.** A treemap with one block per group, titled with the group name, in family order. **One tile = one ETF; its colour = return over the period.** Tiles carry no text; the group title is the only label. Hover shows the ETF card; click opens the drawer.

**Group leaderboard.** Horizontal bars, **one bar = a group's equal-weight return over the period**, sorted, with a dashed vertical line at the reference return. Over 30 groups: top 15 and bottom 15 only.

**Group indices.** One line per visible group plus a heavier dashed reference, rebased to 100 at the start of the chart history; end labels (at most 18); hover lists groups sorted by value.

**Top 12 / Bottom 12.** Two tables of the best and worst ETFs over the period: ETF (short name), group swatch, period return with an in-cell bar, YTD, 1Y. They never share an ETF. Row click opens the ETF.

**Slice 13b additions.** Four more KPI tiles: the reference's return over the period, the share of ETFs ahead of the reference, the share above their 200-day average and the average 52-week correlation between ETFs (the last two are full-universe and read "Unavailable" under a group filter). The ETF hover card adds 1M, YTD, 1Y, 1Y volatility and the 13-week z-score; the leaderboard hover adds the reference return, the median ETF and the ETF count. Under the leaderboard a **signal snapshot**: **one bar = a group's 13-week excess return over the reference as a z-score against its own three years**, coloured by sign. Beside Group indices, **How much moves together**: the first component's share of 52-week variance and the average correlation over time (full-universe). Group indices get the chip legend, and hovering a line or its end label fades the others. Top 12 and Bottom 12 add a z 13W column.

**Empty behaviour.** No finite returns: "No data for this selection". No groups selected: one message.

*Built by slice 5; slice 13b adds the parity panels (`docs/reference/PARITY.md` 13b).*

---

### 4.2 Heatmap

**Purpose.** Look at the same universe through different measures.

**Controls.** Three dropdowns that affect the treemap only: **Colour by** (the six return periods, relative return over 3M or 1Y, 12-1 momentum, distance from the 200-day average, current drawdown), **Size by** (equal or 1Y volatility), **Group by** (group or family).

**ETF heatmap.** A treemap like the Overview's, but **tiles are labelled** with the short name when the tile is large enough; tiny tiles stay blank. Block headers show the group and its ETF count. **One tile = one ETF; colour = the chosen measure.** Hover: full name, ticker, group, the measure.

**Groups across periods.** A matrix: rows are groups (with ETF count); columns are the six return periods and the share of ETFs above their 200-day average. **One cell = the group's equal-weight return over that period**, coloured per column (green/red around zero). Default sort is the shared return period, descending, and follows period changes until the reader sorts something else.

**Slice 13b additions.** Colour by adds 1W, 5Y and 10Y, relative returns over every period and 13 weeks, the 13-week z-score, distance from the 52-week high, gain from the 52-week low, Sharpe and (with factors configured) alpha; a **Colour scale** `Tight | Auto | Wide`; Size by adds years of history, inverse volatility and size of the move; Group by adds the computed cluster. Large tiles show the value under the name, block headers the group's median, and the title names the measure. Groups across periods adds 1W, 5Y and 10Y. **Performance grid**: a full-width table, **one row = one ETF**, sixteen return, relative, signal and trend columns each with its own green-red scale, rows kept in group blocks (a checkbox) and a filter box.

*Built by slice 5; slice 13b adds the parity controls and the grid (`docs/reference/PARITY.md` 13b).*

---

### 4.3 Leadership

**Purpose.** Show which groups are leading, which are rolling over, and how broad participation is.

**Leaders and laggards.** A scatter. Horizontally, how far each ETF is above its 52-week low (log scale); vertically, how far below its 52-week high (0 at top). **One dot = one ETF**, coloured by group. Top-right is the leaders corner (green), bottom-left the laggards corner (red). Labels follow the label budget; the rest are in hover.

**Early-warning groups.** One small card per group: name, "n of 6 warnings" and a sparkline of the group index over the reference. Cards with four or more warnings are red, others green.

**Participation.** The share of ETFs above their own 200-day average over time, with bands at 20 % and 80 %; unavailable under a group filter.

**Slice 13c additions.** The scatter gains a label mode (All, Some, None), a **Group indices** checkbox (one ring per group at the group index's own position), Colour by group or computed cluster, a chip legend, and 3M and 1Y in the hover; labels favour the leaders corner. Early-warning cards gain a sort (most warnings, weakest 3M, name), an arrow for the direction of the ratio's 200-day average, a streak line, six warning dots and the 3-month relative return; groups without usable history are listed once. Participation becomes two linked charts: the equal-weight index of all ETFs over the reference (with its 200-day average), and the share of ETFs and the share of groups above their 200-day average with a latest-value legend; both full-universe.

*Built by slice 7; slice 13c adds the parity controls and charts (`docs/reference/PARITY.md` 13c).*

---

### 4.4 Chart wall

**Purpose.** Scan hundreds of small charts at once, grouped, to catch odd shapes by eye.

**Controls.** **Chart** (growth of 100, drawdown, rolling 52-week return), **Sort ETFs by** (name or any metric, with a direction toggle), **Card height** (four steps), **Y scale** (per group or shared). The range follows the shared chart history.

**Sections and cards.** One section per group with a sticky header (swatch, name, ETF count). **One card = one ETF**: rank, short name, the value of the sort metric coloured by sign (so the order on screen always matches the numbers), and a small chart of the chosen kind in one neutral line colour; the group colour marks the card's edge. Reference lines sit at 100 for growth and 0 for drawdown. Click or Enter opens the drawer. Cards render lazily.

**Slice 13d additions.** Chart adds relative to the reference, relative to the own group, rolling 52-week excess, volatility, beta, correlation and Sharpe, and the 13-week excess z-score and percentile, each with its reference lines. **Group by** group, family, computed cluster or none; **Order groups by** average of the sort metric, name, count or group order; **Y scale** free (default), per group or shared; **Compare with** the reference, the own group index or nothing (a grey line behind each card); a **Search** box; **Group index first** (a dashed card per section). A summary line names the count, chart, history, sort and grey line; section headers show the section average and a Hide/Show toggle; cards show the group when not grouped by group, and every card spans the same dates.

**Empty behaviour.** A card without data says "Not available" or "No data in this range".

*Built by slice 6; slice 13d adds the parity controls and kinds (`docs/reference/PARITY.md` 13d).*

---

### 4.5 Groups

**Purpose.** Compare groups against each other, then look inside each one.

**Controls.** One shared legend of group chips toggles a group in every panel at once (click hides or shows it, hover previews it, Reset restores all); a table-set selector (Returns, Risk, Breadth).

**Panels.**

- *Growth of 100*: one line per group index (equal-weight of members), dashed reference, rebased to 100, end labels (at most 18).
- *Relative to the reference* and *Drawdown from peak*: the same group lines.
- *Group statistics*: one row per group: ETF count plus returns over the standard periods, risk numbers (median volatility, median 1Y drawdown, dispersion) or breadth numbers (share above 200-day, share beating the reference). Switching the set resets the sort; switching the period does not.
- *Inside each group*: a small panel per visible group titled with its name and a note ("23 <label>, index +12.4 % over 1Y", with the project's universe label). Members are thin translucent lines, the equal-weight index bold, the reference dashed. Lazy.

**Slice 13d additions.** A **Log scale** checkbox. *Rolling statistic*: **one line = one group index's 52-week return, excess, volatility, beta, correlation, Sharpe or 13-week z-score** (a select). *Excess z-score over time*: a heat grid, **one row = one group, one column = one week, one cell = the z-score of its h-week excess return** (4, 13, 26 or 52 weeks). The statistics table gains an **Alpha** set (alpha beyond equity and bond factors, its t-statistic, factor betas, residual volatility, appraisal ratio, full-history alpha); Returns adds 5Y, 10Y and the 1Y relative return; Risk shows the group index's own volatility, Sharpe, Sortino, drawdowns, beta, correlation and capture before the member medians; Breadth adds the median 13-week z-score.

*Built by slice 6; slice 13d adds the parity panels and sets (`docs/reference/PARITY.md` 13d).*

---

### 4.6 Rotation

**Purpose.** Show how groups move through relative-strength cycles over time.

**Group rotation.** A weekly relative-rotation quadrant chart. Horizontal: strength against the reference; vertical: whether that strength is speeding up. Four translucent corner-labelled quadrants: Leading (green), Weakening (amber), Lagging (red), Improving (blue). **One head dot = a group at the chosen week**, trailing a smoothed tail of earlier weeks with fading dots. Controls: **Tail** length in weeks (default 8) and an **End week** slider that replays rotation through time.

**Leadership ranks.** A bump chart: months across, rank down; **one line = one group's rank by trailing 3-month return at each month end**. End labels show "rank. group". Hover fades others.

**Leadership table.** Rank, group, 3-month return, rank a month and three months ago, change arrow, quadrant pill; sortable.

**Slice 13c additions.** A group whose latest point is before the chosen week is drawn hollow. Clicking a group head selects it in **ETF rotation inside a group**, a second quadrant chart, **one head dot = one ETF of that group at its latest week**, with its own group select and tail (1 to 20 weeks). Leadership ranks and the table follow the shared return period instead of a fixed 3 months, over the months inside the chart history. **Monthly returns by group**: a heat grid, **one cell = a group's return in one month**, coloured by its rank among the visible groups that month.

**Empty behaviour.** Groups with fewer than 60 valid weeks are omitted and named once. Every footnote says the actual end date (the publication cutoff).

*Built by slice 7; slice 13c adds the parity panels (`docs/reference/PARITY.md` 13c).*

---

### 4.7 Correlation

**Purpose.** See which ETFs move together, and which genuinely diversify.

**ETF correlation matrix.** A full-width heat matrix. **One cell = the correlation of two ETFs' returns**, from minus one (red) through zero (dark) to plus one (green); a colour band along each edge shows group. **Window**: one year of daily returns, or three years of weekly returns (default). **Order**: clustered so similar ETFs sit together (default) or by group. Pairs without enough overlapping observations are blank, never zero. Click selects the row ETF for the next panel. Under a group filter the matrix is the sub-matrix of visible ETFs, keeping the clustering order.

**Closest and furthest.** For the chosen ETF, bars for the 12 most and 8 least correlated peers, coloured by peer group. Click switches to that peer.

**Pairs.** Two lists of 15 pairs, highest and lowest correlation. Row click selects the first ETF.

**Slice 13e additions.** The matrix Window adds **3Y residual** (correlations after removing each ETF's equity-factor exposure; needs factors configured). An ETF select drives Closest and furthest; Pairs gains **Across groups** (default) / **All pairs** and Group A / Group B columns. **Group correlation**: a heat table, **one cell = the correlation of two group indices** (and of each with the reference), 3Y weekly or 1Y daily. **Correlation regime**: average correlation over time for all pairs, pairs within groups, pairs across groups and ETFs to the reference (full-universe). **Rolling correlation to the reference**: **one line = one group index's 52-week correlation with the reference**.

**Empty behaviour.** Fewer than 8 ETFs in the window: "Insufficient data" with the count.

*Built by slice 9; slice 13e adds the parity panels (`docs/reference/PARITY.md` 13e).*

---

### 4.8 Structure & PCA

**Purpose.** Show how much of the universe moves as one block.

The sample is two years (104 weekly returns, complete-case); with fewer than 8 complete ETFs the panels say "Insufficient data".

**Rolling components.** Lines over time: variance share of the first, second and third components, over 52-week windows. A high first share means everything moves together.

**Scree.** Bars for components one to ten: variance share on the two-year sample. The footnote counts ETFs included and excluded. If the fit cannot be made the panel says "Fixed PCA unavailable" and why; it is never repaired silently.

**Loadings.** A scatter of each ETF's loading on the first two components, **one dot = one ETF**, coloured by group, with hull outlines when 16 or fewer groups are visible. ETFs without complete history are omitted and counted, never zero. Click opens the ETF.

All three panels are full-universe analyses; under a group filter each shows the filtered-selection message.

**Slice 13e additions.** Rolling components adds the top-5 absorption ratio and **PC1 ex-market** (the first component of returns after removing equity-factor exposure); the scree shows raw and market-residual bars. A controls row sets **Colour by** group or computed cluster, **Group outlines** and one legend for both scatters. **Residual map**: **one dot = one ETF**, placed so that close dots trade alike beyond the market. Loadings switch between raw and market-residual. **Alpha forest**: **one row = one group's annualised alpha beyond equity and bond factors** with its 95 % interval, for 3Y, 12M, YTD or full history. **Alpha against residual risk**: each group's alpha against its residual volatility with appraisal-ratio rays, and a table below. **Group vs computed cluster**: **one cell = the ETFs of a group placed in a computed cluster**. The map, residual loadings and scree are full-universe; the alpha panels hide hidden groups. Without factors configured the factor panels say "No factor series is configured".

*Built by slice 9; slice 13e adds the residual structure, clusters and alpha panels (`docs/reference/PARITY.md` 13e).*

---

### 4.9 Risk & return

**Purpose.** Plot ETFs on risk against reward and find the outliers.

**Controls.** X and Y axis (volatility, return, drawdown, beta, Sharpe; from slice 13f every metric of the shared catalogue; defaults 1Y volatility and 3Y annualised return; the title reads "<Y> vs <X>"). Colour is the group until slice 13f adds **Colour by** group, family or computed cluster, with **Dot size**, **Labels** (All, Some, None), a chip legend and **Show group indices** (one ring per group at the group index's values).

**Risk vs return.** **One dot = one ETF**, coloured by group. Click opens the ETF.

**Up vs down capture.** **One dot = one ETF**: how much of the reference's up moves it captured (vertical) against its down moves (horizontal), weekly over three years. Above the diagonal is better.

**Distribution by group.** A strip plot of the Y metric: one row per group, a dot per ETF, a median tick.

**Drawdown watch.** Table, deepest first: ETF, group, current drawdown (bar), worst ever (bar), distance from 52-week high and low, 1M return, 1Y volatility. Row click opens the ETF.

**Slice 13f additions.** Dashed lines and a diamond mark the selected reference's values on both axes and in the strip; the hover adds 1Y return and Sharpe. The capture scatter adds dashed parity lines at 1.0 and beta in the hover. The strip gets its own metric select. The drawdown watch colours distance from the 52-week high and low and 1M return as heat cells.

**Empty behaviour.** ETFs with a null on either axis are left off and counted in the footnote.

*Built by slice 8; slice 13f adds the parity controls (`docs/reference/PARITY.md` 13f).*

---

### 4.10 Signal lab

**Purpose.** Examine how unusual an ETF's or group's recent excess return is relative to its own history.

**Controls.** **Series** (an ETF or a group index; from slice 13h also a benchmark), **Horizon** (4, 13, 26, 52 weeks), **z window** (1, 2, 3, 5 years). Excess is over the reference until slice 13h adds **Excess over** reference, own group or nothing.

**Six stacked panels.** Index (log, rebased); excess return (area around zero); rolling mean with one and two spread bands; z-score (dotted at plus and minus one, labelled at two); percentile (guides at 10, 50, 90); histogram of z with the current bin filled and a "today z, percentile" label.

**Same signal, every group today.** Sorted bars of each visible group's latest z (one bar per group, short group name beside it); the selected group is outlined and the selected ETF is a marker at its own z. Click loads the group into the controls.

**Slice 13h additions.** A numbered pipeline line explains the six steps for the current settings; tiles add the series name, window mean and window spread; the histogram uses quarter-unit bins with the share of weeks on the y axis. The cross-section can also show **one bar per ETF** plus one outlined bar per group index, with a button that recomputes every bar for non-canonical settings.

**Empty behaviour.** Not enough history for the chosen window: "Insufficient data" with the minimum.

*Built by slice 10; slice 13h adds the parity controls (`docs/reference/PARITY.md` 13h).*

---

### 4.11 Screener

**Purpose.** Find ETFs: filter, sort and compare in one wide table. First tab to build; it proves the data path.

**Controls.** **Search** (ticker, name, short name, group, family; debounced). Groups are filtered with the shared group picker.

**Table.** One row per ETF: short name (full name and ticker on hover), group swatch, family, returns 1M, 3M, 6M, YTD, 1Y and 3Y (annualised), 1Y volatility, 1Y max drawdown, first and last date, and a status pill (`ok`, `short`, `stale`, `no data`; the hover text gives the last date and days stale). From slice 2 the shared return period's column comes first among the returns. Clicking a header sorts, with a visible arrow; the default sort is the shared return period, descending, and follows period changes until the reader sorts something else. The count line reads "<shown> of <total> <label>" with the project's universe label. The first column stays pinned.

**Slice 13f additions.** **Columns** sets Overview, Returns, Risk, Signals, Factor (with factors configured; adds the computed cluster) and All; **Colour cells** turns signed columns into green-red heat; **Include group indices** adds one row per group with the group index's own metrics; a sparkline column shows each ETF over the chart history; header hover gives the full metric name.

**Empty behaviour.** Nulls render as an em dash with the reason on hover and always sort to the bottom. A search with no match shows "No data for this selection".

*Built by slice 1; slice 13f adds column sets, heat cells, group rows and sparklines (`docs/reference/PARITY.md` 13f).*

---

### 4.12 Calendar

**Purpose.** Seasonality and year-by-year patterns.

**Monthly returns.** A grid, **one cell = one calendar month's return** for the chosen series (an ETF or a group; default the reference), years descending, January to December, plus a compounded year column. One decimal, green or red; a missing month is blank.

**Seasonality.** Twelve bars, **one bar = the average return for that calendar month across full years**, with the percentage of years that were up in the label.

**Calendar-year returns by group.** Years down, visible groups across (full names on hover), plus the reference. A star and "n of 12" mark years with fewer than 12 months of data.

**Slice 13h additions.** The grid, seasonality and year table show only the years inside the chart history; seasonality hover gives average, median, share up and years; year-table columns use initials with a key. **Seasonal paths**: **one card = one year's path over a chosen window** (start month, length, a year filter by the four-year election cycle) rebased to 100 with its low marked, plus summary statistics. **Monthly returns, fund by fund**: **one cell = one month's return** for the reference, a group's index and each of its ETFs.

*Built by slice 7; slice 13h adds history scope, seasonal paths and the fund-by-fund heat (`docs/reference/PARITY.md` 13h).*

---

### 4.13 Market

**Purpose.** Gauge the health of the whole shelf and the reference at a glance.

**Cards.** Six line charts over the shared chart history: breadth (share of ETFs above their 50-day and 200-day averages, 50 % line); new 52-week highs (green) and lows (red); implied correlation; dispersion (cross-sectional spread of 21-day returns); reference drawdown; cash rate (annual percent, labelled with the project's cash label, gaps kept, never filtered by group).

**Members by return.** Squares, one per ETF with a finite period return, best to worst, short name and value inside, coloured green or red. Not clickable.

**Slice 13g additions.** Two more cards: **equal weight vs the reference** (the equal-weight index of all ETFs over the reference) and **share up over 21 days**. A status line above the cards counts ETFs with large moves in the shown range. The dispersion card gets a scale switch (Auto, Linear, Log). Square hover shows the rank and four returns.

**Empty behaviour.** Under a group filter, every card built from market data (breadth, highs and lows, implied correlation, dispersion, equal weight vs the reference, share up; until slice 13g breadth and implied correlation) shows the filtered-selection message; reference drawdown, cash and the squares stay. Missing cash data leaves a gap and a project without a cash series makes the card say "No cash series", never zero.

*Built by slice 4; slice 13g adds the parity cards and rules (`docs/reference/PARITY.md` 13g).*

---

### 4.14 Fund

**Purpose.** Everything about one ETF on one screen.

**Controls.** An ETF select grouped by group (visible groups plus the current ETF) and the shared chart history. Selecting an ETF adds a history entry.

**Header.** Full name large; ticker; pills: group (swatch), family, data status, first to last date.

**KPI tiles.** Returns 1M, 3M, 6M, YTD, 1Y, 3Y (annualised) and since inception (annualised), 1Y volatility, 1Y and full-history max drawdown, current drawdown; from slice 4 also relative return 3M and 1Y, Sharpe and beta (3Y). Sign-coloured; nulls as an em dash with the reason.

**Charts.** Growth of 100 (the ETF in its group colour; from slice 4 the reference dashed), log at long ranges, and drawdown from the full-history peak. History changes only the displayed dates.

**Peers.** The visible members of the ETF's group as a table sorted by the shared return period, the current ETF underlined; row click switches ETF.

**Slice 13g additions.** The select also offers every group index and every benchmark as the subject. The header adds flag chips and, for an ETF, `Trades like: <cluster>`. Tiles add 5Y, alpha 3Y and its t-statistic, the 13-week z-score and the distances from the 52-week high and low. The growth chart adds the ETF's group index, the drawdown chart the reference's drawdown. New panels: rolling 52-week return and excess; rolling beta and correlation; monthly returns; **Most correlated <label>** (the 14 most correlated ETFs over three years). The peers table shows period, 1M, YTD, 1Y, volatility, Sharpe and z columns.

**Empty behaviour.** An ETF with no usable history shows "No usable history" in the header, empty tiles with em dashes, and "No data" panels. No fund in the hash gives "Pick a fund".

*Built by slice 3 (header, tiles, charts, select, peers), slice 4 (reference line and tiles) and slice 13g (subjects, panels, pills; `docs/reference/PARITY.md` 13g).*

---

### 4.15 Data

**Purpose.** Show what data the site holds and whether to trust it.

**History per ETF.** A timeline. **One bar = one ETF's first to last observation**, in universe order, group colours, dimmed when status is not `ok`, year ticks. Click opens the ETF.

**Coverage and quality.** A table with instant search and sort: ETF, group, status pill (exactly the `status` of the data file: `ok`, `short`, `stale`, `no data`) with four flag chips (`stale`, `gap`, `spike`, `break`) beside it, first date, last date, days stale, observations. Above it, counts of stale, gap, spike, break, large-move, short-history (under one and three years) and unpriced ETFs, the coverage on the latest date and the publication cutoff.

**Method.** Short prose with live values: price source, adjusted closes, missing stays missing, the windows in use, and the publication record (generated time, source, synthetic). A failed build leaves the previous data file in place, so an old generation time is visible here.

**Slice 13h additions.** The timeline groups ETFs by group, spans the chart history and names each bar's dates, years and status on hover. Method covers returns and missing data, benchmarks, cash, factors, group indices, shared settings, every metric window, adding ETFs and the publication record, all with live values.

*Built by slice 11; the review queue panel is added by slice 12 (section 4.16); slice 13h completes the timeline and Method (`docs/reference/PARITY.md` 13h).*

---

### 4.16 Review mode

**Purpose.** The monitor is only useful on clean data. Review mode lets the reader correct what the build cannot know. It is a mode of the Fund and Data tabs (from slice 13h also a strip on every tab), not a tab.

**Switch.** The Review toggle ends the settings strip; off by default, remembered. Off hides the actions; built marks stay visible.

**Actions** (one fund at a time):
- *Flag broken price*: whole series, or a range dragged on the price chart or typed; those closes are dropped, never filled.
- *Exclude*: the fund stays listed, dimmed, but leaves group, market, correlation and PCA numbers.
- *Move to group*: the fund joins the picked group on every tab.
- *Undo*: removes a pending mark.
- *Clear*: cancels the marks already built for the fund (a pending `clear` mark, applied by the next build).

**Loop.** Actions make pending marks in the browser: no number changes, and the fund shows a `Pending` pill. *Export marks* saves `review_<timestamp>.json`; put it in the project's `review/` folder and run `build`.

**Review queue (Data tab).** One row per fund the build flagged (`stale`, `gap`, `spike`, `break`) or that has a mark: status, flags, marks with file, actions in review mode. Pending first. With review off it is a read-only record.

**Reader view.** Pills `Excluded`, `Prices removed`, `Moved from <group>`, `Pending`; hover gives mode, range, note and file. No banner. Marks for a ticker that is no longer in the universe are ignored by the build and listed as warnings under the queue.

**Review strip** (slice 13h). In review mode a strip under the settings holds **Mark on click** (Off, Broken price, Exclude, Move), a note box, **Suspects only** (lists only flagged or marked ETFs) and **Discard pending**. With a marker on, activating an ETF anywhere on any tab marks it instead of opening it; marks show as pills on Screener rows and wall cards and as outlines on treemap tiles.

```
Fund tab, review mode on (under the header)
+--------------------------------------------------------------------------+
| REVIEW · AAA            2 pending                     [Export marks (2)]  |
| [Whole series]  Range [2025-03-03] to [2025-03-07] [Add]  or drag chart   |
| [Exclude]   Move to [group ........ v] [Move]   [Undo]                   |
+--------------------------------------------------------------------------+
Data tab, review queue
+--------------------------------------------------------------------------+
| REVIEW QUEUE                                          [Export marks (2)]  |
| ETF  Status  Flags      Marks                          Actions            |
| AAA  ok      gap spike  broken 03-03..03-07 (file)     [Undo]             |
| CCC  ok      break      none                [Broken] [Exclude] [Move]     |
+--------------------------------------------------------------------------+
```

*Built by slice 12; slice 13h adds the review strip (`docs/reference/PARITY.md` 13h.15).*

---

## 5. Global rules

**Names.** Short names are the default in table cells, tiles, chart labels and legends. Full name and ticker are always in hover text and drawers. A setting switches to full names; long labels are truncated with an ellipsis and complete on hover.

**Legends.** Chips start active. Click toggles a series; hover or focus previews it; Reset restores all. Legend state survives redraws, is not persisted, and never filters the data.

**Layout.** Maximum width 1800 px; one or two panels per row, dense comparisons stacked; one column below 900 px, where low-priority table columns hide. No horizontal page scroll at 390 px; wide tables scroll inside their panel. Chart and legend labels fit inside their panel.

**Nothing fabricated.** No forward-fill into returns, no splicing of two tickers, no assumed zero cash rate, no invented history. Missing is null, drawn as an em dash with a reason. Synthetic data is marked in the data file and named as the source in the header.

**Colour and motion.** A group keeps its palette colour on every tab. Colour never carries a value alone. Only 150 ms hover transitions.

**Fixed and free.** Tabs, panels, controls, defaults and empty-state wording are fixed here; pixel sizes and spacing are free within `docs/THEME.md`.
