# Parity spec (slice 13)

Slice 13 completes the dashboard: every control, panel and metric that slices 1 to 12 leave out. It is split into eight sub-slices. **Build order: 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h** (letters are names, not the order, as for slices 11 and 12): 13e publishes the residual structure, factor fits and computed clusters that 13b, 13c, 13d, 13f and 13g read. `docs/BUILD_SLICES.md` holds each sub-slice's steps and checklist; every step points to a numbered item here. The coverage table at the end lists every feature item with the slice that builds it.

## 0. Conventions

- `C` is the coverage cutoff (METRICS 10); "own last close" is an ETF's `t_end` (METRICS 1, Anchor). Group-level and cross-sectional outputs end at `C`; per-ETF outputs at the own last close.
- `<label>` is `project.universe_label`; `<bench>` is the selected benchmark's label; "the reference" is the selected benchmark (UI 7).
- **Weekly levels.** ETF: the last close of each W-FRI bucket on the `wdates` grid of its series file (DATA 9). Group: the last non-null `group_index` level of each W-FRI bucket, buckets up to the last Friday on or before `C`. Benchmark: the last level of `_bench_<id>` in the bucket. Weekly returns follow METRICS 1 (a missing week nulls two returns).
- **Browser-computed** means computed in `web/` from the published fields named, never published; windows and minimums come from `data.params` (PITFALLS 2). A number computed both in Python and in the browser needs the parity check named in its Done line.
- Reference-dependent fields are objects keyed by benchmark id with the DATA 9 `why` rule. New reason codes: `factors_missing` (UI text `No factor series is configured`) and `cutoff_prior_year` (UI text `The coverage cutoff is in the previous calendar year`).
- **Full-universe** panels show the UI 6 note under a group filter (`Unavailable for a filtered group selection. Restore all groups to view this full-universe analysis.`) and never recompute.
- Labels, legends and fitting follow UI 8: every group-coloured chart has the chip legend, scatters have the `All / Some / None` label mode, signed bars are coloured by sign with a group swatch beside the label, and every touched tab passes `check=1` with `overlaps 0, clipped 0`.
- Group marks stay hover-only (PRODUCT 3): a group ring, bar, card or row never opens the drawer or another tab. Group indices are opened deliberately through the Fund tab's subject select (13g.12).
- **Size.** Slice 13 adds no per-ETF weekly array to the series files. Every new per-ETF time series (relative lines, rolling volatility, beta, correlation, Sharpe, excess percentile) is browser-computed from `adj_close`, the slice-6 and slice-10 arrays, `_bench_<id>`, `_cash` and `group_index`, so a 500-ETF project with 5 benchmarks keeps the slice-10 size of its series files (about 15 MB of signal arrays plus the closes), far under 60 MB. The new top-level data (`correlation.corr_3y_residual`, about 0.6 MB for 520 ETFs, and `correlation.groups`) is stored at `CORR_DECIMALS`.

### Rule changes made in other docs

The docs below were edited in place; this table is the only place that records the change.

| Doc, section | Rule now | Changed by |
|---|---|---|
| PRODUCT 3, UI 4 `period` | Return period options 1W 1M 3M 6M YTD 1Y 3Y (`r_1w` added) | 13a |
| METRICS 3.1 | Horizons include 1W (7 calendar days), 5Y (60 months) and 10Y (120 months) | 13a |
| METRICS 4 | Reference-dependent list gains `rel_1w rel_1m rel_6m rel_ytd rel_3y_ann corr_3y med_z_13w ew_dir ew_days ew_rel_3m` | 13a, 13c |
| METRICS 6 | Member values entering group medians and shares are evaluated at `C`; group `r_ytd` null when `as_of` is in a later year than `C`; `med_z_13w` published | 13a |
| DATA 9 `benchmarks[].returns` | Evaluated at `C`, nine period ids | 13a |
| DATA 2b, 3, 4, 5, 15 | `[factors]` table; factor tickers are extra tickers; `project.load` allows `factors` | 13a |
| UI 8, UI 9 | End labels carry the last value and highlight their series on hover; crosshair lists at most `TIP_MAX_LINES` series | 13a |
| UI 11 | Loading and failed states for series files | 13a |
| METRICS 7.4, DATA 12 | `EW_BACK_LONG` 63 and `EW_BACK_SHORT` 21 rows before the last row | 13c |
| METRICS 7.4, DATA 9 `rotation` | `rank_trail` published for every return period | 13c |
| PRODUCT 4.9 | Colour by Group, Family or Computed cluster (was: always the group) | 13f |
| PRODUCT 3, UI 4, UI 9 | Chart history adds 10Y (120 months); growth log scale at 5Y, 10Y and MAX | 13g |
| UI 4 `fund` | May hold `g:<group>` or `b:<benchmark id>` (Fund tab subject; never opens the drawer) | 13g |
| UI 5, PRODUCT 2 | As-of line with benchmark, priced and stale counts and the cutoff hover | 13g |
| PRODUCT 3, 4.13, UI 13 item 13 | Every Market card built from `market` is full-universe | 13g |
| PRODUCT 4.10, METRICS 8 | Excess over Reference, Own group or Nothing; percentile minimum for custom windows | 13h |
| UI 13.3 (as built) | Leadership rings sit at the group index's own position, for every visible group (was: median member position, groups of 3 or more) | 13c |
| UI 13.2 (as built) | Wall Y scale `Free` (default), `Per group`, `Shared` (was: group or shared) | 13d |
| UI 13.6 (as built) | Lab histogram bins 0.25 wide; the cross-section may also draw one bar per ETF (second mode) | 13h |
| UI 13 intro | `docs/reference/PARITY.md` takes precedence over an older per-tab note | 13a |

---

## 13a Shared data, metrics and helpers

Depends on slice 10 (so every slice 1 to 12). Builds the data and browser helpers that several later sub-slices read.

### 13a.1 Factor series `[factors]`

- **Settings (DATA 2b).** Optional table `[factors]` with exactly the keys `equity` and `bond`, both required when the table is present (an empty table counts as absent). Each is a ticker (`universe.TICKER`) whose series is in `prices.csv`: a universe member, a benchmark component or an extra ticker; the cash ticker is refused. An unknown key, a wrong type or one key missing raises `ValueError` (build exits 2, `data.js` kept). `project.load` allows the top-level section `factors` and returns `factors` (`{}` when absent).
- **Prices.** The factor tickers join the extra tickers of `prices.read`, `sample` and `fetch` (DATA 3, 4, 5 "Slice 4 additions"); they are screened like benchmark-only tickers and are not listed in `quality.funds`.
- **Shipped projects.** The root `settings.toml` gains `[factors] equity = "SPY"`, `bond = "AGG"`; `examples/europe/settings.toml` gains the component tickers of its first two benchmarks (equity, then bond). Code never defaults a factor.
- **Factor returns.** `e_k`, `b_k` are the W-FRI weekly returns of the two tickers' `adj_close` (alignment carry of METRICS 1), on the weekly grid of the series being regressed.
- **Missing.** Without `[factors]`, or when a factor ticker has no rows, every factor output (13e.1 to 13e.4) is null with `factors_missing`; panels show `No factor series is configured`.
- **data.js.** `params.factors`: `{equity, bond}` (tickers) or `null`.
- **Done.** Removing `[factors]` builds with exit 0 and every factor field null with `factors_missing`; `[factors] equity = 1` exits 2 with `data.js` byte-identical.

### 13a.2 Weekly cash returns in `_cash.js`

- **Formula.** `cash_w` = the METRICS 4 weekly cash return `c_d` of each Friday `d`: `product over j in [d-7, d-1] of (1 + y_j / 100 / 360) - 1` with the bounded carry; null when any of the 7 days has no bounded rate. Fridays from the first Friday after the first cash observation to the last Friday on or before `as_of`. Full precision.
- **data.js.** Series file `_cash.js` gains `wdates` (ISO Fridays) and `cash_w` (array on `wdates`, decimal fraction or null). The browser reads it for every rolling Sharpe and never re-derives cash from `rate`.
- **Done.** Test: the `cash_w` of a Friday equals the weekly cash return `sharpe_3y` uses for that week; a hole of more than `CASH_CARRY_DAYS` (7) days nulls the Fridays whose 7-day window it touches (a single missing day is bridged by the carry and does not).

### 13a.3 New per-ETF metrics

All at the ETF's own last close; weekly slots per METRICS 1 and 5 (as built). Plain fields are appended to `metrics.FIELDS` in this order: `r_1w r_5y_ann r_10y_ann years vol_3y sortino_3y hit_1y`; `corr_3y` is appended to `metrics.REF_FIELDS`.

| id | Definition | Window | Min obs | Null (reason) |
|---|---|---|---|---|
| `r_1w` | `P / L_a - 1`, `L_a` the last close on or before `t_end - 7 days`, used only within `ANCHOR_TOL_DAYS` | 7 calendar days | anchor | `no_history_to_anchor` |
| `r_5y_ann` | `(P / back(60M))^(1/5) - 1` | 60 months | anchor | `no_history_to_anchor` |
| `r_10y_ann` | `(P / back(120M))^(1/10) - 1` (null on the shipped samples: `YEARS` and `FETCH_YEARS` are 6) | 120 months | anchor | `no_history_to_anchor` |
| `years` | `(t_end - first_date).days / 365.25` | full | 1 close | `no_data` |
| `vol_3y` | `sd(weekly r, ddof 1) * sqrt(52)` | last `REF_WEEKS` (156) slots | `REF_MIN_PAIRS` (104) finite returns | `insufficient_weekly_returns` |
| `sortino_3y` | `x = r - c` weekly; `mean(x) / sqrt(sum(min(x, 0)^2) / N) * sqrt(52)`, `N` the finite `x` | last 156 slots | 104 finite `x`; every slot with a finite `r` has a finite `c` | `cash_missing`, `cash_incomplete`, `insufficient_pairs`, `zero_variance` (no `x < 0`), in this order |
| `hit_1y` | share (fraction) of finite weekly returns above 0 | last `VOL_WEEKS` (52) slots | `VOL_MIN_WEEKS` (40) finite | `insufficient_weekly_returns` |
| `corr_3y` | Pearson of `r` and `m` on the masked pairs of `beta_3y` (reference-dependent) | last 156 slots | 104 pairs | `insufficient_pairs`, `zero_variance`, `reference_missing` |

- **data.js.** `funds[]` fields above (number or null; `corr_3y` an object keyed by benchmark id).
- **Done.** Test: a fund with 1.5 years of history (under 104 weekly returns; 2.5 years give 129 and a value) has `vol_3y` null with `insufficient_weekly_returns` and `r_5y_ann` null with `no_history_to_anchor`; `sortino_3y` is null with `cash_incomplete` when one cash week inside the window is missing; `r_1w` is null when the last close before `t_end - 7 days` is more than 7 days older.

### 13a.4 Relative returns for every period

| id | Definition | Null (reason) |
|---|---|---|
| `rel_1w`, `rel_1m`, `rel_6m`, `rel_ytd` | As `rel_3m` (METRICS 5 and its as-built note): `P / L_s - REF_{t_end} / REF_s`, `s` the ETF's anchor date of `r_1w`, `r_1m`, `r_6m`, `r_ytd`, the benchmark level read with the alignment carry | own anchor missing (`no_history_to_anchor`); benchmark level missing at `s` or `t_end` (`reference_missing`) |
| `rel_3y_ann` | `(P / L_s)^(1/3) - (REF_{t_end} / REF_s)^(1/3)`, `s` the anchor of `r_3y_ann` | as above |

- Period id to relative id: `r_1w rel_1w`, `r_1m rel_1m`, `r_3m rel_3m`, `r_6m rel_6m`, `r_ytd rel_ytd`, `r_1y rel_1y`, `r_3y_ann rel_3y_ann` (written `rel_<period>` below).
- **data.js.** `funds[]` `rel_1w rel_1m rel_6m rel_ytd rel_3y_ann`, objects keyed by benchmark id, appended to `metrics.REF_FIELDS` after `corr_3y`.
- **Done.** For the benchmark's own component ticker, `rel_<period>[that benchmark]` is 0 for every period where defined.

### 13a.5 Return period 1W and the nine period keys

- **Setting.** The shared return period gains `r_1w` (label `1W`), first in the list: `r_1w r_1m r_3m r_6m r_ytd r_1y r_3y_ann` (UI 4 `period`, PRODUCT 3). Every tab that keys by period reads the `r_1w` key; a tab whose period-keyed field has no `r_1w` value shows `—` with its reason.
- **Keys.** `groups[].group_return` and `benchmarks[].returns` hold nine ids: the seven period ids plus `r_5y_ann` and `r_10y_ann`.
- **Done.** With period 1W every tab redraws without an error; the Screener's first return column is `1W`.

### 13a.6 Group-index metrics

| id | Definition | Evaluated at |
|---|---|---|
| `index_metrics` | The per-ETF metric functions (every id of `metrics.FIELDS` and `metrics.REF_FIELDS`, including ids later sub-slices append) run on the group's `group_index` levels dated on or before `C`, nulls dropped, as if they were closes; partial-week limit `C`; cash and benchmarks as for an ETF. Keys in `FIELDS` then `REF_FIELDS` order. A group without levels has every value null with `no_data`. | the last level on or before `C` |
| `index_why` | `{field: reason}` for the null entries, same shape as `funds[].why` | as above |

- **Invariant.** `index_metrics[p]` equals `group_return[p]` for the nine period ids (one function, one anchor rule).
- **YTD rule.** When the calendar year of `as_of` is later than that of `C`, `group_return.r_ytd`, `index_metrics.r_ytd` and `index_metrics.rel_ytd` are null (`index_why` `cutoff_prior_year`).
- **data.js.** `groups[]` gains `index_metrics` (object) and `index_why` (object), appended after `med_z_13w` (13a.7).
- **Done.** Test (with 13a.7): `index_metrics.r_3m == group_return.r_3m`; the Signal lab's browser-computed group z at the last slot equals `index_metrics.z_13w[benchmark]` within 5e-5 (checked in the browser).

### 13a.7 Group aggregates at the cutoff

- **Members at `C`.** `med_r_1m med_r_3m med_r_1y med_vol_1y med_maxdd_1y med_z_13w breadth_200d beat_ref_3m dispersion_1y` take each member's value evaluated with `t_end` = its last close on or before `C`: `build` reruns the metric functions on the member's series cut at `C` when the member has a later close (members come through DATA 8). The published `funds[]` values stay at the own last close.
- **`med_z_13w`.** Object keyed by benchmark id: median of the members' finite `z_13w` at `C`; null when none is finite.
- **data.js.** `groups[]` gains `med_z_13w`, appended after `ew_tells`.
- **Done.** Test: a member with closes after `C` contributes its value at `C` to `med_r_3m` and `group_return` (equal to a build where those later closes are absent); with `as_of` in January and `C` in December `group_return.r_ytd` is null.

### 13a.8 Benchmark metrics and returns at the cutoff

- **`benchmarks[].returns`** is evaluated at `C` (the benchmark's last level on or before `C`), so it compares with `group_return` on the same dates; nine period ids (13a.5).
- **`benchmarks[].metrics`**: the per-ETF metric functions (`FIELDS` and `REF_FIELDS`) on the benchmark's level series at its own last level; **`benchmarks[].why`** the reasons. Against itself the formulas give `rel_*` 0, `beta_3y`, `corr_3y`, `up_cap`, `down_cap` 1 and `z_13w` null (`zero_variance`); nothing is special-cased.
- **data.js.** `benchmarks[]` entry keys: `id label components series_file returns metrics why`.
- **Done.** `benchmarks[i].metrics.beta_3y[its own id]` is 1 within 1e-9.

### 13a.9 Co-movement series

On every `pca_rolling` window (52 weekly returns ending at each weekly date up to `C`, the complete-case ETFs of METRICS 7.2: all 52 returns finite, sample SD above `PCA_STD_MIN`, member funds only).

| id | Definition | Min obs | Null when |
|---|---|---|---|
| `avg_corr` | mean of the upper-triangle Pearson correlations over all pairs of the window's complete ETFs | `PCA_MIN_FUNDS` (8) complete ETFs | fewer than 8 (the line breaks) |
| `within` | the same mean over pairs whose two ETFs share a `group` (after review moves) | 8 complete ETFs and 1 such pair | fewer, or no same-group pair |
| `across` | the same mean over pairs in different groups | 8 and 1 such pair | fewer, or no cross-group pair |
| `corr_ref` | per benchmark: mean over the window's complete ETFs of the Pearson of the ETF's 52 weekly returns with the benchmark's | 8 complete ETFs and all 52 benchmark returns finite | fewer; any benchmark return null in the window |

- **data.js.** `pca.pca_rolling` gains `avg_corr`, `within`, `across` (arrays on its `dates`) and `corr_ref` (`{benchmark id: array}`), rounded to `CORR_DECIMALS`.
- **Done.** `avg_corr` is null exactly where `pc1` is null.

### 13a.10 Metric registry `ARCADE.METRICS`

- One registry in `app.js`: id to `{label, short, fmt, signed, ref, set}`; `fmt` one of `pct pctu num z ratio years`; `signed` true for returns, `rel_*`, drawdowns, `z_13w`, `mom_12_1`, `vs_200d`, `from_52wh`, `from_52wl`, Sharpe, Sortino, alpha, `alpha_t`; `ref` true for reference-dependent ids (read with `ARCADE.ref`, labels end `vs <bench>`); `set` one of Returns, Relative, Trend, Risk, Risk-adjusted, Factor, Structure, History.
- Catalogue (every per-ETF numeric field): Returns `r_1w r_1m r_3m r_6m r_ytd r_1y r_3y_ann r_5y_ann r_10y_ann r_si`; Relative `rel_1w rel_1m rel_3m rel_6m rel_ytd rel_1y rel_3y_ann rel_13w z_13w pct_13w`; Trend `mom_12_1 vs_200d from_52wh from_52wl`; Risk `vol_1y vol_3y maxdd_1y maxdd dd_now beta_3y corr_3y up_cap down_cap hit_1y`; Risk-adjusted `sharpe_3y sortino_3y`; Factor `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y alpha_full alpha_t_full` (listed only when `params.factors` is set); Structure `pc1 pc2 rpc1 rpc2`; History `years`. Ids a sub-slice has not built yet are skipped (a missing field is null).
- Used by Heatmap colour, Chart wall sort, Risk axes and strip, Screener sets and headers, the drawer and tooltips, so a label is written once.
- **Done.** The drawer lists every catalogue id the data holds, labelled from the registry.

### 13a.11 Weekly helpers and rolling statistics

- `ARCADE.weekly(dates, values, limit)` returns `{wdates, levels}` (the last value of each W-FRI bucket up to `limit`, the conventions' weekly levels); `ARCADE.wret(levels)` the weekly returns. `ARCADE.roll` holds the rolling functions below; `params.roll` gives `{weeks, min}`.
- Statistics are computed on the full weekly series, then sliced to the chart history (METRICS 9). Evaluated at every weekly slot; an ETF ends at its last slot, a group at the last Friday on or before `C`, a benchmark at its last level.

| id | Definition | Window | Min obs | Null when |
|---|---|---|---|---|
| `roll_ret` | existing (METRICS 9): `W_k / W_{k-52} - 1` | 52 slots | both levels | either null |
| `roll_ex` | `roll_ret(series) - roll_ret(REF)` on the same W-FRI slot | 52 slots | both finite | either null |
| `roll_vol` | `sd(r, ddof 1) * sqrt(52)` | last `params.roll.weeks` (52) slots | `params.roll.min` (40) finite `r`, current `r` finite | fewer; current null |
| `roll_beta` | `cov(r, m) / var(m)` on slots where both are finite | 52 slots | 40 pairs, current pair finite | fewer; `var(m) = 0` |
| `roll_corr` | Pearson of `r`, `m` on the same pairs | 52 slots | 40 pairs, current pair finite | fewer; a zero variance |
| `roll_sharpe` | `mean(r - c) / sd(r) * sqrt(52)`, `c` from `_cash.js` `cash_w` | 52 slots | 40 finite `r`, current finite | no `[cash]`; any slot with a finite `r` and a null `c`; `sd <= 1e-12` |

- **data.js.** `params.roll`: `{weeks: VOL_WEEKS, min: VOL_MIN_WEEKS}`. Everything else browser-computed from the series files, `_bench_<id>`, `_cash` and `group_index`.
- **Done.** `roll_sharpe` is null at a week whose window holds one missing cash week, even with 51 complete weeks.

### 13a.12 Line charts: end labels, hover highlight, crosshair cap

- **End labels** read `<label> <last finite value>` in the chart's tooltip format and sit at the series' last finite point (a series that ends early is labelled where it ends). The full text stays in `<title>`.
- **Highlight.** Hovering or focusing a line or its end label fades the other series to 12 % (the legend preview, UI 8); leaving restores them. Applies to every `ARCADE.lineChart`, including the Overview group indices.
- **Crosshair cap.** The crosshair tooltip lists at most `TIP_MAX_LINES` series sorted by value, then `+N more`; series switched off in the legend are not listed.
- **Constants.** `TIP_MAX_LINES` = 16 (`web/app.js`, display only).
- **Done.** The Overview group-index tooltip with 59 groups shows 16 lines and `+43 more` inside the viewport; Fund growth end labels read like `<bench label> 132`; `check=1` reports `overlaps 0, clipped 0`.

### 13a.13 Series-file loading and failure

- While `ARCADE.series` loads a file the panel shows `Loading <ticker>…` at its final height (no layout jump). A file that fails to load shows `Failed to load series/<file>. Rebuild, then reload.`; other panels still draw.
- **Done.** Renaming one series file makes that ETF's Fund panels show the failure text while the header and tiles render.

---

## 13e Correlation and Structure & PCA

Depends on 13a. Built second: its residual structure, factor fits and computed clusters are read by 13b, 13c, 13d, 13f and 13g.

### 13e.1 Equity-factor residuals

| id | Definition | Window | Min obs | Null when |
|---|---|---|---|---|
| `resid` | For each member ETF: OLS with intercept of weekly `r` on the equity factor's weekly return `e` over the weekly slots up to the last Friday on or before `C` where both are finite; `resid_k = r_k - a - beta * e_k` on those slots, null elsewhere | full history to `C` | `RESID_MIN_WEEKS` (104) slots | fewer (the ETF has no residuals); no `[factors]` (`factors_missing`) |

`resid` is an intermediate (not published). It uses raw returns, so it needs no cash.

### 13e.2 Factor fits

Rows are the weekly slots of the series (ETF: its own grid to its last slot; group: group weekly levels to `C`) where `r`, `e`, `b` and the weekly cash return `c` are all finite. A slot where `r`, `e` and `b` are finite but `c` is null makes the fit null (`cash_incomplete`).

| Item | Rule |
|---|---|
| Model | OLS of `y = r - c` on `[1, e - c, b - c]` |
| Windows | `3y`: the last `REF_WEEKS` (156) slots; `full`: every slot from the first slot with a finite weekly cash return on (an ETF older than the cash series is fitted on the shared part; a null `c` after that start is `cash_incomplete`), publishing its row count and start; `12m` (groups only): slots whose previous Friday is on or after `t_end - 12 months`; `ytd` (groups only): slots whose previous Friday is on or after 31 Dec of the prior year of `t_end` |
| Minimum rows | `3y`, `full`: `REF_MIN_PAIRS` (104); `12m`, `ytd`: `FACTOR_SHORT_MIN` (26) |
| Standard errors | Newey-West: `V = (X'X)^-1 S (X'X)^-1`, `S = sum u_t^2 x_t x_t' + sum_{L=1..HAC_LAGS} (1 - L / (HAC_LAGS + 1)) sum_t u_t u_{t-L} (x_t x_{t-L}' + x_{t-L} x_t')`, lags counted on consecutive rows; no small-sample correction |
| Outputs | `alpha = a * 52`; `alpha_se = se(a) * 52`; `alpha_t = a / se(a)`; `beta_equity`, `beta_bond`; `res_vol = sd(u, ddof 3) * sqrt(52)`; `ar = alpha / res_vol` (appraisal ratio); `alpha_lo`, `alpha_hi` = `alpha -+ T_CRIT * alpha_se` (browser) |
| Null reasons, in order | `factors_missing`, `cash_missing`, `cash_incomplete`, `insufficient_pairs` (fewer rows), `zero_variance` (design rank below 3) |
| Evaluated at | ETF: own last slot; group: the last Friday on or before `C` |

- **data.js.**
  - `funds[]` plain fields appended to `metrics.FIELDS`: `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y` (window `3y`) and `alpha_full alpha_t_full` (window `full`). Decimal fractions for alpha and `res_vol`, plain numbers otherwise. Being in `FIELDS`, they also appear in `groups[].index_metrics` and `benchmarks[].metrics` (13a.6, 13a.8).
  - `groups[]` gains `factor_fit`: `{"3y": F, "12m": F, "ytd": F, "full": F}` with `F = {alpha, alpha_se, alpha_t, beta_equity, beta_bond, res_vol, ar, n, start, end, status}`; `n` rows used (int), `start` the first row's Friday minus 7 days and `end` the last row's Friday (ISO, null without rows), `status` `ok` or one of the reasons above. `factor_fit["3y"].alpha` equals `index_metrics.alpha_3y`.
  - `params.t_crit` (from `T_CRIT`).
- **Done.** Test: a synthetic ETF built as `cash + 0.5 * (equity - cash) + 0.5 * (bond - cash) + 2 % a year` (the plain `0.5 equity + 0.5 bond + cash + 2 %` has alpha 2 % plus the cash rate, because the model regresses `r - c` on `e - c` and `b - c`) has both betas within 0.01 of 0.5 and `alpha_3y` within 0.5 % of 2 % (also with no noise at all: an exactly zero residual is `zero_variance`, a tiny one is not); a 12M group fit with one missing cash week has status `cash_incomplete` and null alpha; without `[factors]` every fit is `factors_missing`.

### 13e.3 Residual PCA, absorption ratio, PC1 ex-market

| id | Definition | Window | Min obs | Null / status |
|---|---|---|---|---|
| `pca_resid` | As `pca_fixed` (METRICS 7.2) on `resid` over the last `PCA_WEEKS` (104) weekly rows to `C`, complete-case (all 104 residuals finite, SD above `PCA_STD_MIN`); shares of components 1 to `PCA_COMPONENTS`; PC1 sign by the `pca_orient` rule, PC2 by the largest-magnitude loading; loadings `v * sqrt(max(lambda, 0))` | 104 weeks | 8 complete ETFs | statuses of `pca_status` plus `factors_missing` |
| `ar5` | `sum(lambda_1..PCA_AR_COMPONENTS) / sum(lambda)` of each `pca_rolling` window | 52 weeks | 8 complete ETFs | as `pca_rolling` |
| `pc1_resid` | `lambda_1 / sum(lambda)` of the correlation of `resid` over the window's complete raw ETFs that also have all 52 residuals finite with SD above `PCA_STD_MIN` | 52 weeks | 8 such ETFs | fewer; no `[factors]` |

- **data.js.** `pca` gains `pca_resid_status`, `pca_resid_n` (`{used, excluded}`), `pca_resid_fixed` (null unless ok, else `{end, weeks, shares}`); `pca.pca_rolling` gains `ar5` and `pc1_resid` (arrays on its `dates`, rounded to `CORR_DECIMALS`); `funds[]` gains `rpc1`, `rpc2` (residual loadings; null outside the sample; no `why` entry, not in `FIELDS`, like `pc1 pc2`).
- **Done.** Test: fewer than 8 complete residual series give `pca_resid_status` `insufficient_data` and null `rpc1`.

### 13e.4 Residual map (MDS) and computed clusters

| id | Definition | Min obs | Null / status |
|---|---|---|---|
| `rho` | Pearson of `resid` for each pair over the full history to `C`, pairwise-complete | `RESID_MIN_WEEKS` (104) overlapping slots | blank pair |
| Sample | ETFs with residuals; while any pair is blank, drop the ETF with the most blank pairs (ties: the later one in universe order) | `PCA_MIN_FUNDS` placed | fewer: `mds.status` `insufficient_data` |
| `mds_x`, `mds_y` | Classical MDS in numpy: `D = sqrt(max(2 (1 - rho), 0))`, `B = -1/2 J D^2 J` with `J = I - 11'/n`; coordinates = the top two eigenvectors times `sqrt(max(lambda, 0))`; each axis signed so its largest-magnitude coordinate is positive | as above | ETF dropped or without residuals: null |
| `cluster` | Ward linkage on `D` over the placed ETFs (numpy Lance-Williams update on squared distances, no scipy; average linkage stays for matrix order only, because it leaves one giant cluster and many singletons), stopped when `K = min(number of groups with a placed member, n)` clusters remain. Naming: repeatedly take the (cluster, group) pair with the largest member count among unnamed clusters and unused groups (ties by group name, then by the cluster's first member in universe order); the cluster takes that group's name. Clusters left unnamed are `Cluster 1`, `Cluster 2`, ... by their first member in universe order | as above | ETF not placed: null |

- **data.js.** `funds[]` gains `mds_x`, `mds_y` (numbers or null) and `cluster` (string or null), no `why` entries; `pca` gains `mds` (`{status, n, dropped}`: status `ok`, `insufficient_data` or `factors_missing`, ETFs placed, ETFs dropped) and `clusters` (cluster names: the named ones in group order, then `Cluster k`).
- **Done.** Test: on a synthetic set with two blocks of four ETFs driven by two factors the clusters split the blocks and take the names of the two groups; ties are named by group name.

### 13e.5 Correlation: 3Y residual window

- **Reader sees.** The matrix Window gains a third option **3Y residual**: correlations of weekly residuals after removing each ETF's equity-factor exposure. Without `[factors]` the option is disabled with the hover `No factor series is configured`. Order Clustered uses the window's own order.
- **Formula.** `corr_3y_residual`: Pearson of `resid` over the last `CORR_WEEKLY_ROWS` (156) weekly rows to `C`, pairwise-complete, minimum `CORR_WEEKLY_MIN` (104) pairs; average-linkage order as METRICS 7.1.
- **data.js.** `correlation.corr_3y_residual`: the window shape of DATA 9 (`{end, rows, min_pairs, n, status, order, upper}`), status also `factors_missing`.
- **Done.** The residual window renders with its own order; it is disabled without `[factors]`.

### 13e.6 Correlation: ETF select

- **Reader sees.** An **ETF** select above Closest and furthest lists every ETF of the current window by display name; it shares `tabState("correlation").sel` with matrix and pair clicks. A window without the stored ETF shows the first ETF without overwriting the stored choice.
- **Done.** Choosing an ETF in the select redraws the bars; switching window keeps the stored choice.

### 13e.7 Correlation: pairs mode and group columns

- **Reader sees.** Segmented **Across groups** (default) / **All pairs** above the pair lists; Across groups keeps only pairs whose two ETFs are in different groups (`group` after review moves). Columns: Set (Highest, Lowest), Pair (short names, full text in hover), Group A, Group B (`narrow: false`), rho. Kept in `tabState("correlation").pairs`.
- **Done.** Across groups lists no pair whose ETFs share a group.

### 13e.8 Group correlation

- **Reader sees.** Panel *Group correlation*: a square heat table, **one cell = the correlation of two group indices**, plus a last row and column for the selected benchmark labelled with its label. Window segmented **3Y weekly** (default) / **1Y daily**. Column heads use initials when a name is longer than 14 characters, full name in hover. Colour -1..1 (UI 9 diverging). Blank pairs in `--raised`. Under a group filter rows and columns of hidden groups are removed (exact; footnote `Recomputed for N visible groups`).
- **Formula.** `corr_3y_weekly`: Pearson of weekly returns of the group weekly levels and of each benchmark's weekly levels, last 156 weekly rows to `C`, minimum 104 pairs. `corr_1y_daily`: Pearson of daily returns of `group_index` (one step per market-axis date; null where a level is missing) and of each benchmark level on the same axis dates (alignment carry), last `CORR_DAILY_ROWS` (252) rows to `C`, minimum `CORR_DAILY_MIN` (200) pairs. Zero variance blank.
- **data.js.** `correlation.groups`: `{corr_3y_weekly, corr_1y_daily}`, each `{end, rows, min_pairs, ids, upper}`; `ids` lists group names in `groups[]` order, then `bench:<id>` per benchmark; `upper` in the DATA 9 layout, rounded to `CORR_DECIMALS`, nulls kept.
- **Done.** Hiding a group removes its row and column and keeps the benchmark row.

### 13e.9 Correlation regime

- **Reader sees.** Panel *Correlation regime* (full-universe): **one line = one average correlation over time** from 52-week windows: all pairs (bold), within groups, across groups, and ETF-to-benchmark for the selected benchmark. Reference line at 0, span the chart history, end labels. A window with fewer than 8 complete ETFs is a gap.
- **Data.** Browser reads `pca.pca_rolling.avg_corr within across corr_ref` (13a.9).
- **Done.** The lines break where `avg_corr` is null; a group filter shows the full-universe note.

### 13e.10 Rolling correlation to the reference

- **Reader sees.** Panel *Rolling correlation to the reference*: **one line = one group index's 52-week correlation with the selected benchmark**; reference line at 0, end labels, the shared group legend (UI 8), line hover highlight; span the chart history.
- **Formula.** `roll_corr` (13a.11) on group weekly returns against benchmark weekly returns.
- **Data.** Browser-computed from `group_index` and `_bench_<id>`.
- **Done.** A group with fewer than 40 weekly pairs in a window has a gap there, never 0.

### 13e.11 Correlation layout

Matrix full width; then two rows of two: (Group correlation, Correlation regime) and (Rolling correlation to the reference, Closest and furthest); Pairs full width. Stacked below 900 px.

### 13e.12 Rolling components: absorption and PC1 ex-market

- **Reader sees.** *Rolling components* gains **Top-5 absorption** (`ar5`) and **PC1 ex-market** (`pc1_resid`) beside PC1 to PC3; kicker `One line = a variance share over a 52-week window`. Without `[factors]` the ex-market line is absent and the footnote says `No factor series is configured`.
- **Done.** Both new lines appear with `[factors]`; under a group filter the panel shows the full-universe note.

### 13e.13 Scree: raw and market-residual

- **Reader sees.** Two bars per component, **Raw** and **Market-residual** (from `pca_resid_fixed.shares`). Footnote `Raw: N in, M excluded · Residual: N in, M excluded` (`pca_n`, `pca_resid_n`). Each side shows its own status text when not ok.
- **Done.** Without `[factors]` the residual side reads `No factor series is configured` and the raw bars still draw.

### 13e.14 Colour by, group outlines, shared legend

- **Reader sees.** A controls row above the scatters: **Colour by** Group (default) / Computed cluster (cluster colours `--g1` to `--g12` by `pca.clusters` order); **Labels** `All | Some | None` (UI 8) for the map and the loadings; **Group outlines** checkbox (default on): hulls for each colour category with at least 3 dots, drawn only when 16 or fewer categories are shown, with a hover that says why they are suppressed otherwise; one chip legend for the active colour mode driving both scatters (hover previews, click toggles, Reset). Kept in `tabState("pca")` (`color`, `labels`, `hulls`, `off`).
- **Done.** Computed cluster recolours both scatters and the legend; unticking outlines removes hulls.

### 13e.15 Residual map

- **Reader sees.** Panel *Residual map* beside Loadings: **one dot = one ETF**, placed so that distance approximates how differently two ETFs move after removing equity-factor exposure; close dots trade alike beyond the market. Axes unitless (no tick labels). Colour by the active mode, hulls per 13e.14, label budget and label mode (UI 8). Hover: name (ticker), group, cluster, 1Y return, `beta_3y vs <bench>`. Click opens the drawer. Footnote: ETFs placed and dropped (`pca.mds`). Full-universe.
- **Data.** `funds[].mds_x`, `mds_y`, `cluster`.
- **Done.** The map shows `n` dots equal to `pca.mds.n`; under a group filter it shows the full-universe note.

### 13e.16 Loadings: Raw / Market-residual

- **Reader sees.** Segmented **Raw** (default) / **Market-residual** on the Loadings panel; residual mode plots `rpc1` against `rpc2`. Kept in `tabState("pca").load`.
- **Done.** Switching to Market-residual redraws with `rpc1`/`rpc2`; ETFs outside the residual sample are counted in the footnote.

### 13e.17 Alpha forest

- **Reader sees.** Panel *Alpha forest*: **one row = one group's annualised alpha beyond the equity and bond factors**, a horizontal 95 % interval (`alpha_lo` to `alpha_hi`) in the group colour, a marker filled when `|alpha_t| >= params.t_crit` and hollow otherwise, value text `+a.a % t x.x`. Window segmented **3Y** (default) / **12M** / **YTD** / **Full** (`tabState("pca").fw`), marker shape by window (circle, diamond, triangle, circle). Sorted by alpha descending; x includes 0 with a reference line. Footnote `k of n groups have an eligible <window> fit` and the minimum rows (`FACTOR_SHORT_MIN` for 12M and YTD, `REF_MIN_PAIRS` otherwise). The shared legend of the two alpha panels highlights groups. Group rows are hover-only: window, alpha, t, interval, matching weeks (`n`), fit dates (`start`, `end`). Visible groups only (not full-universe).
- **Data.** `groups[].factor_fit`, `params.t_crit`.
- **Done.** A marker is filled only when `|t| >= 1.96`; 12M and YTD rows show their row count.

### 13e.18 Alpha against residual risk

- **Reader sees.** Panel beside the forest, sharing its window: **one filled dot = a group's alpha in the chosen window; one hollow dot = its full-history fit**, joined by a thin line, the hollow one labelled `since YYYY`. x = residual volatility a year (from 0), y = alpha a year (symmetric around 0); dashed rays through the origin at appraisal ratios -0.5, -0.25, +0.25, +0.5, +1 labelled `AR +x`. Hover: alpha, t, residual volatility, AR, `beta_bond`, rows. Below it a table: Group, ETFs (`n`), beta bonds, alpha, t, residual vol, AR, alpha full (start year), and for 12M and YTD `Matching weeks · status`; default sort alpha descending.
- **Done.** Each group with both fits shows a filled and a hollow dot joined by a line; the table sorts by every column.

### 13e.19 Group vs computed cluster

- **Reader sees.** Full-width panel: a crosstab, **one cell = the number of ETFs of a group (row) placed in a computed cluster (column)**; sequential colour, a cell whose cluster is named after its row's group in bold, a Total column, column heads abbreviated with a key `abbr = name` in the footnote. Under a group filter it lists the visible groups and notes `Clusters come from the full-universe fit`.
- **Data.** Browser-computed from `funds[].group` and `cluster`, columns in `pca.clusters` order.
- **Done.** Row totals equal the placed ETFs of each group.

### 13e.20 Structure layout and filter

Rows: (Rolling components, Scree), controls row, (Residual map, Loadings), (Alpha forest, Alpha against residual risk), Group vs computed cluster. The map, both loadings modes, both scree sides and the rolling lines are full-universe; the alpha panels simply hide hidden groups.

---

## 13b Overview and Heatmap

Depends on 13a and 13e.

### 13b.1 KPI strip additions

- **Reader sees.** Four tiles join the six (ten in all, wrapping): **Reference, <period>** (`benchmarks[].returns[period]` at `C`, sign-coloured; sub-line `1M <value>`; footnote names `C`); **<label> ahead of the reference, <period>** (share of visible ETFs with `rel_<period>[bench] > 0` among those with a finite value; sub-line `<a> of <n> <label>`); **<label> above 200-day** (last non-null `market.above_200d`; sub-line `above 50-day <value>`); **Average correlation, 52 weeks** (last non-null `pca_rolling.avg_corr`, two decimals; sub-line `PC1 explains <pc1>` from the same row).
- **Filter.** The breadth and correlation tiles are full-universe: under a group filter they read `Unavailable` with the UI 6 note as hover. The ahead share is recomputed for the visible ETFs. Empty scope blanks every tile to `—`.
- **Done.** Hiding one group turns the breadth and correlation tiles into `Unavailable` and changes the ahead count.

### 13b.2 ETF hover card (Overview and Heatmap treemaps)

After the name, ticker and group: the measure in bold, then 1M, YTD, 1Y, Vol 1Y and `z 13W vs <bench>` (skipping a line that repeats the measure). **Done.** Hovering a tile shows six value lines.

### 13b.3 Leaderboard hover

Hover a bar: group (header), its period return (bold), `Reference <returns[period]>`, `Median ETF <med_r_<period>>` (only for 1M, 3M and 1Y, the published medians), `<n> <label>`. **Done.** With period 3M every bar's hover shows four lines.

### 13b.4 Signal snapshot

- **Reader sees.** Under the leaderboard bars: **one bar = a group's 13-week excess return over the reference as a z-score against its own trailing three years**, sorted high to low, coloured by sign with the group swatch beside the label (UI 8), values signed with two decimals. More than `SNAPSHOT_BARS` groups: the top and bottom halves only. Hover: group, z, 13-week excess. Groups without a z are counted in the footnote.
- **Data.** `groups[].index_metrics.z_13w[bench]` and `rel_13w[bench]` (13a.6).
- **Constants.** `SNAPSHOT_BARS` = 24 (`web/tabs/overview.js`, display only).
- **Done.** With 59 groups the snapshot shows 12 and 12 bars; switching the benchmark redraws it.

### 13b.5 How much moves together

- **Reader sees.** Panel beside Group indices: two lines over the chart history, **the first component's share of 52-week return variance (percent)** and **the average pairwise correlation x 100**, end labels. Footnote `52-week windows of weekly returns, at least 8 complete <label>; ends <C>`. Full-universe.
- **Data.** `pca.pca_rolling.pc1`, `avg_corr`.
- **Done.** Under a group filter the panel shows the full-universe note.

### 13b.6 Top 12 / Bottom 12: z column

Add `z 13W` (`z_13w[bench]`, `narrow: false`, sign-coloured); when the period is YTD or 1Y the duplicate column is not repeated. **Done.** Period YTD shows one YTD column.

### 13b.7 Overview group indices legend

The group-index chart carries the shared chip legend (UI 8), its selection in `tabState("overview").off`. **Done.** A chip click fades its line and end label to 12 %.

### 13b.8 Heatmap: Colour by additions

Colour by adds 1W, 5Y (annualised), 10Y (annualised), relative return 1W, 1M, 6M, YTD and 3Y, relative return 13W, z 13W, distance from 52-week high, gain from 52-week low, Sharpe 3Y and, when `params.factors` is set, alpha 3Y. Labels from `ARCADE.METRICS`. The default stays the shared period; picking a period id writes no shared setting (PRODUCT 3). **Done.** Every option recolours the treemap only.

### 13b.9 Heatmap: Colour scale

Segmented **Tight | Auto | Wide** (default Auto, `tabState("heatmap").scale`): cap = 95th percentile of `abs(measure)` times `HM_SCALE[choice]`. **Constants.** `HM_SCALE` = `{tight: 0.6, auto: 1, wide: 1.8}` (`web/tabs/heatmap.js`). **Done.** Tight visibly saturates more tiles than Wide.

### 13b.10 Heatmap: Size by additions

| Option | Size value (browser) | Left out (counted in footnote) |
|---|---|---|
| Years of history | `max(HM_YEARS_FLOOR, years)` | `years` null |
| Inverse 1Y volatility | `1 / max(vol_1y, HM_INV_VOL_FLOOR)` | `vol_1y` null |
| Size of the move | `max(abs(measure), HM_MOVE_FLOOR)` | measure null |

**Constants.** `HM_YEARS_FLOOR` = 0.5, `HM_INV_VOL_FLOOR` = 0.02, `HM_MOVE_FLOOR` = 0.003 (`web/tabs/heatmap.js`). **Done.** Inverse volatility makes the calmest ETF of a block the largest tile.

### 13b.11 Heatmap: Group by cluster

Group by adds **Computed cluster** (`funds[].cluster`, blocks in `pca.clusters` order, unplaced ETFs in a last block `Not placed`). **Done.** Cluster grouping shows one block per cluster name.

### 13b.12 Heatmap: tile value line

A labelled tile tall enough for two lines (height at least 2.4 times the label font size, inside the UI 13.1 size rule) shows the formatted measure under the short name. **Done.** `check=1` reports no clipped tile text.

### 13b.13 Heatmap: block header and title

Block headers read `<group> · <n> · med <median of the block's finite measure>`; the treemap title reads `ETF heatmap · <measure label>` (`<label>` replaces ETF where it names the plural). **Done.** Changing Colour by changes the title and each block median.

### 13b.14 Groups across periods: 1W, 5Y and 10Y

Columns 1W, 1M, 3M, 6M, YTD, 1Y, 3Y, 5Y, 10Y from `group_return` (5Y and 10Y `narrow: false`), then the share above 200-day. **Done.** The 1W column sorts.

### 13b.15 Performance grid

- **Reader sees.** A full-width table below: **one row = one ETF**, columns ETF, group, 1W, 1M, 3M, 6M, YTD, 1Y, 3Y, 5Y, vs reference 3M, vs reference 1Y, z 13W, vs 200-day, vs 52-week high, current drawdown. Each column has its own diverging scale (UI 9, 95th percentile of the column's shown absolute values). Controls: **Keep groups together** (checkbox, default on: rows sorted inside group blocks in group order, each block opened by a swatch-and-name separator row) and a **Filter** search box (`ARCADE.search` over ticker, name, short name, group, family and the shown values, debounced; count line `<shown> of <total> <label>`). Default sort the shared period descending until a header is clicked (`tabState("heatmap").gridSort`). Scrolls inside the panel (at most `GRID_MAX_HEIGHT` px) with sticky header and first column. Row click opens the drawer.
- **Constants.** `GRID_MAX_HEIGHT` = 720 (`web/tabs/heatmap.js`).
- **Done.** The grid shows every visible ETF in group blocks, filters as you type, and its sort follows the period until a header is clicked.

---

## 13c Leadership and Rotation

Depends on 13a and 13e.

### 13c.1 Leaders and laggards: controls and legend

- **Reader sees.** Controls: **Labels** `All | Some | None` (UI 8; default Some); **Group indices** checkbox (default on): one hollow ring per visible group at the group index's own position (`1 + index_metrics.from_52wl`, `index_metrics.from_52wh`), titled `<group> (group index)`, labelled with the group name when `LEAD_RING_LABEL_MAX` or fewer groups are visible; hover only. **Colour by** Group (default) / Computed cluster. A chip legend for the active colour mode (click toggles, hover previews, Reset). Kept in `tabState("leaders")` (`labels rings color off`).
- **Rule change.** Rings sit at the group index's own position (was: the median member position).
- **Constants.** `LEAD_RING_LABEL_MAX` = 16 (`web/tabs/leaders.js`).
- **Done.** Labels, rings, colour mode and legend change the scatter without a reload and survive a history change.

### 13c.2 Label priority

Labels go first to the selected ETF, then to rings, then by `prio = 100 * ln(x) - 0.6 * y_pct + (30 when y > LEAD_PRIO_NEAR_HIGH)` (x the gain factor, `y_pct` the percent below the high), skipping a label that would collide. **Constants.** `LEAD_PRIO_NEAR_HIGH` = -0.03. **Done.** With Some, the leaders corner is labelled first.

### 13c.3 Scatter hover

Hover lines: group, `Above 52W low` (bold), `From 52W high` (bold), 3M, 1Y. **Done.** Every dot's hover shows five lines.

### 13c.4 Early-warning metrics

| id | Definition | Min obs | Null when | Evaluated at |
|---|---|---|---|---|
| `ew_tells` offsets | Tests 4 and 5 compare the last ratio with the ratio `EW_BACK_LONG` (63) and `EW_BACK_SHORT` (21) rows before the last row | as `ew_tells` | as `ew_tells` | the rows of `ew_tells` |
| `ew_dir` | `sign(MA200(ratio)_last - MA200(ratio)_{EW_LAG rows before the last})`: -1, 0 or 1 | 200 + 20 rows | either mean null | same rows |
| `ew_days` | Signed streak: the count of consecutive most recent rows with `ratio < MA200(ratio)` (positive), else with `ratio >= MA200(ratio)` (negative); a row without ratio or mean ends the streak | last row has ratio and mean | last ratio or mean null | same rows |
| `ew_rel_3m` | `ratio_last / ratio_{EW_BACK_LONG rows before the last} - 1` | both values | either null | same rows |

- **data.js.** `groups[]` gains `ew_dir`, `ew_days` (integer) and `ew_rel_3m` (decimal fraction), objects keyed by benchmark id, appended after `factor_fit`.
- **Done.** Test: `ew_days` counts a below-average streak as positive and an at-or-above streak as negative, and a missing ratio row ends the streak.

### 13c.5 Early-warning cards

- **Reader sees.** A **Sort** control `Most warnings | Weakest 3M | Name` (default Most warnings, ties by weakest 3M; groups without a value last; `tabState("leaders").ewSort`). Each card adds an arrow (`▲` rising, `▼` falling, `→` flat, `—` unavailable; hover `200-day average of the ratio is rising|falling|flat`) from `ew_dir`; a streak line (`below its 200-day for 12 days`, `at or above its 200-day for 30 days`, `streak unavailable`) from `ew_days`; six dots with the warnings filled in `--negative` (hover `k of 6 warnings`); and `+2.1 % vs <bench>, 3M` coloured by sign from `ew_rel_3m`. Groups without a usable ratio get no card and are listed once under the grid as `Groups without usable history (n)` (focusable, hover `no usable index history · <n> <label>`).
- **Done.** Cards sort three ways and show arrow, streak, dots and 3M relative.

### 13c.6 Participation

- **Reader sees.** *How many take part*: two linked charts over the chart history (one x domain and left margin). Top: **the equal-weight index of all member ETFs divided by the reference**, rebased to 100 at the first date inside the history where both exist, its 200-row average in grey and a dashed line at 100; kicker `Falling means the average <label> lags the reference`. Bottom: **share of ETFs** (`market.above_200d`) and **share of groups** (`market.groups_above_200d`) above their own 200-day average, bands 80-100 % (stretched, negative tint) and 0-20 % (washed out, positive tint), a two-chip legend with the latest values (`<label> 63 %`, `Groups 55 %`; `aria-label` `<label> above their 200-day average today: 63 %`). Both full-universe.
- **Formula.**

| id | Definition | Min obs | Null when | Evaluated at |
|---|---|---|---|---|
| `ew_index` | The METRICS 6 (as built) `group_index` chain run over all member funds on the market axis: level 100 at the first axis date with a member close, factor = mean of `L_t / L_d` over members with a close on `t` and on the previous level date `d`; no qualifying member gives a null level and `d` stays | 1 member | no member qualifies | each axis date up to `C` |
| `groups_above_200d` | Share of groups whose `group_index` level on the date is above the mean of its last 200 non-null levels (a full 200 and a current level required) | 1 eligible group | none eligible | each axis date up to `C` |
| EW ratio (browser) | `ew_index_t / B_t` on dates where both exist (no filling), rebased at the first such date inside the history; `B` from `_bench_<id>` | both legs | either leg null | as inputs |

- **data.js.** `market` gains `ew_index` and `groups_above_200d` (arrays on `market.dates`; a level, and a fraction or null).
- **Done.** Test: on synthetic data where every member equals the same series, `ew_index` is that series rebased to 100; `groups_above_200d` is null before 200 group levels exist. A group filter turns both charts into the full-universe note.

### 13c.7 Group rotation: stale heads and head click

A head whose last non-null point is before the chosen end week is drawn hollow and its hover adds `Latest week <date>`. Clicking a group head (or Enter on it) selects that group in *ETF rotation inside a group* (an in-tab selection, not a navigation). **Done.** Clicking a head switches the inside-group panel to that group.

### 13c.8 ETF rotation inside a group

- **Reader sees.** A second quadrant chart beside Group rotation for the member ETFs of one group: **one head dot = one ETF at its latest published week**, with a tail of earlier weeks. Controls: **Group** select (visible groups, default the first; set by a head click) and **Tail** 1 to 20 weeks (default 5), kept in `tabState("rotation")` (`fgroup ftail`). It always ends at each ETF's own latest week (the end-week slider does not move it). Colours cycle `--g1` to `--g12` by member order; short-name labels within the budget; click opens the drawer. Fewer than 2 members with a ratio: `Insufficient data: needs 2, has M`.
- **Data.** Existing `rotation.benchmarks[id].funds`.
- **Done.** Changing Group or Tail redraws only this panel.

### 13c.9 Leadership ranks follow the return period

- **Reader sees.** *Leadership ranks* and *Leadership table* follow the shared return period (1W, 1M, 3M, 6M, YTD, 1Y, 3Y annualised; default 3M); the bump chart shows the month ends inside the chart history; the table's return column and note name the period.
- **Formula.** `rank_trail[period]`: at each month end `m` (the last `group_index` date on or before `min(month end, C)`), the trailing return over the period with the anchors of METRICS 3.1 (`r_3y_ann` annualised, `r_1w` 7 days); rank 1 = highest among visible groups (the browser re-ranks), ties by group order; null when there is no anchor.
- **data.js.** `rotation.rank_trail.by_period`: `{period id: {ret: {group: [..]}, rank: {group: [..]}}}` on `rank_trail.months` for the seven period ids (the 3M entry equals the existing `ret`, `rank`).
- **Done.** Switching the period to 1Y re-ranks the bump chart and the table without resetting tail or end week.

### 13c.10 Monthly returns by group

- **Reader sees.** A full-width panel last on the tab: a heat grid, **one row = one group, one column = one calendar month, one cell = that month's group return**; cell text the return in percent (one decimal, only where the cell is at least 28 px wide); colour = the rank among visible groups that month (diverging, best `--positive`, worst `--negative`); a group-colour band at each row start; month labels every third column; span the chart history. Hover: group, month, return, `rank r of N`. A missing month is blank and not ranked.
- **Data.** Browser-computed from `calendar.groups[name].monthly_return` (`start`).
- **Done.** A missing month is blank; hiding a group re-ranks the others.

---

## 13d Chart wall and Groups

Depends on 13a and 13e.

### 13d.1 Chart kinds and reference lines

| Kind | Series on a card (browser) | Reference lines |
|---|---|---|
| Growth of 100 | daily `adj_close` rebased at the first close in the history (existing) | 100 |
| Relative to the reference | `rel_ref_t = L_t / REF_t`, rebased to 100 at the first date in the history where both exist; breaks where a leg is null | 100 |
| Relative to own group | `rel_grp_t = L_t / G_t` with the ETF's `group_index`, rebased the same way | 100 |
| Drawdown | existing `drawdown_t` | 0 |
| Rolling 52-week return | existing `roll_ret` | 0 |
| Rolling 52-week excess | `roll_ex` vs the selected benchmark | 0 |
| Rolling 52-week volatility | `roll_vol` | none |
| Rolling 52-week beta | `roll_beta` vs the selected benchmark | 1, and 0 without widening the scale |
| Rolling 52-week correlation | `roll_corr` vs the selected benchmark | 0 |
| Rolling 52-week Sharpe | `roll_sharpe` (`cash_w`) | 0 |
| 13-week excess z-score | the published weekly `z_13w[bench]` array | 0, and dashed +-2 without widening the scale |
| 13-week excess percentile | `pct13_t`: share of finite `rel_13w[bench]` in the trailing `SIGNAL_WIN` (156) slots at or below the current value, current included; needs `SIGNAL_PCT_MIN` (52) finite and a finite current value | 50 |

Rolling ids are those of 13a.11. A kind that cannot be computed shows `Not available` with the reason (`cash_missing` for Sharpe without `[cash]`). **Done.** All twelve kinds draw on every card with their reference lines.

### 13d.2 Group by and Order groups by

**Group by** `Group | Family | Computed cluster | None` (None: one section `All <label>`). **Order groups by** `Average of sort metric | Name | Count | Group order` (default Average, nulls last, following the direction). Kept in `tabState("wall")` (`by order`). **Done.** Order by Average puts the section with the highest average sort metric first when sorting high to low.

### 13d.3 Y scale and aligned dates

**Y scale** `Free | Per group | Shared` (default Free: each card its own range; Per group: min and max over the section's cards including the compare line; Shared: the existing 1st to 99th percentile over every card). Every card spans the same dates, history start to `as_of`, so cards align. **Done.** Free gives each card its own axis range; a stale ETF's line ends early on the shared x range.

### 13d.4 Compare with

**Compare with** `Reference | Own group index | Nothing` (default Reference): a muted 1 px line of the same kind behind each card's line, for the selected benchmark or the ETF's group index. Left out where it would be the card's own denominator (Relative to the reference with Reference; Relative to own group always); a group card compares with the reference. **Done.** Switching to Own group index swaps every grey line.

### 13d.5 Search

A search box (ticker, name, short name, group, family, cluster; debounced 200 ms; hash `q`) filters the cards; empty sections are hidden. **Done.** Typing a ticker leaves one card.

### 13d.6 Group index first

Checkbox **Group index first** (default on; group sections only): each section starts with a dashed-border card titled `Group index`, sub-label `<n> <label>, equal weight`, drawing the chosen kind for the group index (weekly levels to `C`; the z and percentile kinds computed with `ARCADE.lab`); no rank; hover only. **Done.** Every group section starts with the dashed card when ticked.

### 13d.7 Summary line, section header and collapse

- Summary line under the controls: `<N> <label> in <S> sections · <chart> · <history> · sorted by <metric> · grey line = <bench label | group index | none>`.
- The sticky section header adds `average <metric short> <value>` (mean of the section's finite sort-metric values; omitted when none) and a `Hide | Show` toggle (kept for the visit in `tabState("wall").collapsed`, not persisted).
- **Done.** Hiding a section keeps it hidden across a history or benchmark change.

### 13d.8 Card header

`#rank`, short name (full name in hover), the sort metric's value (sign-coloured for signed metrics, metric label in hover; sorted by name: the window value of the chart kind); outside group sectioning the ETF's group under the name. **Done.** The card value always equals the sort metric shown in the summary line.

### 13d.9 Groups: log scale

Checkbox **Log scale** in the legend row (default on; `tabState("groups").log`, not persisted): Growth of 100 uses a log y at 2Y and longer histories, the per-group panels at 5Y, 10Y and MAX; off is always linear. **Done.** The checkbox switches Growth of 100 between log and linear and survives a history change.

### 13d.10 Groups: rolling statistic

- **Reader sees.** Full-width panel after Drawdown: **one line = one group index's statistic over a trailing 52-week window**, plus the selected benchmark dashed (omitted for the excess kind). Select (`tabState("groups").roll`): Return, Excess vs reference (default), Volatility, Beta to reference, Correlation to reference, Sharpe, 13-week excess z-score. Reference lines as 13d.1. Kicker `Weekly, 52-week window, minimum 40` or `13-week excess, z in a trailing 156-week window`. Driven by the tab's legend.
- **Formula.** 13a.11 rolling ids and the METRICS 8 canonical signal on group weekly levels (`ARCADE.lab`).
- **Done.** All seven kinds draw; nulls break the line (never 0); the beta kind has a reference line at 1.

### 13d.11 Groups: excess z-score over time

- **Reader sees.** Full-width heat grid: **one row = one group, one column = one week, one cell = the z-score of the group's h-week excess return over the reference**. Horizon segmented 4W / 13W (default) / 26W / 52W (`tabState("groups").zh`). Span: the chart history, starting at the first week where any visible group has a z. Diverging colour `--negative` through `--panel` to `--positive`, capped at `Z_HEAT_CAP`; group-colour band at the row start; year labels on the x axis (every other year above 400 columns). Hover: group, week, z, the h-week excess. Null cells `--raised`. Canvas above about 2,000 cells (UI 9).
- **Formula.** `g_rel_h = prod(1 + r_g) - prod(1 + m)` over the last `h` weekly returns, all `2h` finite; `g_z_h` = z of `g_rel_h` in the trailing `SIGNAL_WIN` values (current included), minimum `SIGNAL_Z_MIN`, null when `sd <= SIGNAL_SD_MIN`; group weekly levels to `C`.
- **Constants.** `Z_HEAT_CAP` = 2.5 (`web/tabs/groups.js`).
- **Done.** The grid changes with the horizon; a group with fewer than 104 values has blank cells.

### 13d.12 Groups: statistics sets

The set selector becomes **Returns | Risk | Alpha | Breadth** (default Returns); switching resets the sort to the set's first column, descending.

| Set | Columns |
|---|---|
| Returns | `group_return` 1W, 1M, 3M, 6M, YTD, 1Y, 3Y, 5Y, 10Y; `index_metrics.rel_1y[bench]` |
| Risk | `index_metrics` `vol_1y vol_3y sharpe_3y sortino_3y maxdd dd_now beta_3y corr_3y up_cap down_cap` (group index at `C`), then member values `med_vol_1y med_maxdd_1y dispersion_1y` |
| Alpha | `factor_fit["3y"]` `alpha alpha_t beta_equity beta_bond res_vol ar`, then `factor_fit.full` `alpha alpha_t`; without `[factors]` the columns show `—` with `No factor series is configured` |
| Breadth | `n`, `med_r_1m med_r_3m med_r_1y med_z_13w[bench]`, `breadth_200d`, `beat_ref_3m[bench]`, `dispersion_1y` |

**Done.** Each set shows its columns with null reasons on hover; the Alpha set reads `—` with the reason without `[factors]`.

---

## 13f Risk & return and Screener

Depends on 13a and 13e.

### 13f.1 Axis catalogue and dynamic title

Both axis selects list the `ARCADE.METRICS` catalogue (13a.10) grouped by `set`; defaults stay `vol_1y` / `r_3y_ann`; the panel title reads `<Y label> vs <X label>`; reference-dependent axis titles end `vs <bench>`. Hash `x`, `y` unchanged. Picking a return id never changes the shared period (PRODUCT 3). **Done.** Choosing `sortino_3y` on Y titles the scatter `Sortino 3Y vs Volatility 1Y` and a history change keeps it.

### 13f.2 Colour by

Select **Colour by**: Group (default), Family, Computed cluster (family and cluster colours `--g1` to `--g12` by family order and `pca.clusters` order). The legend follows the mode. `tabState("risk").color`. **Done.** Family mode shows one chip per family and recolours every dot.

### 13f.3 Dot size

Select **Dot size**: Equal (default, 4.5 px), Years of history (`2.5 + 1.1 * sqrt(years)`), 1Y volatility (`2.5 + 0.9 * sqrt(100 * vol_1y)`), |alpha 3Y| (`2.5 + 0.9 * sqrt(100 * abs(alpha_3y))`, listed only when `params.factors` is set); a null size value gives 3 px. `tabState("risk").size`. **Done.** Years makes the longest-history ETF the largest dot.

### 13f.4 Label mode and legend

**Labels** `All | Some | None` (UI 8; All draws a colliding label at its first candidate position, None draws none) and the chip legend above the main scatter (dots fade to 12 %, labels to 15 %); both act on the main scatter only. `tabState("risk")` (`labels off`). **Done.** None removes every point label; a chip hover previews one category and changes no count.

### 13f.5 Group-index rings

Checkbox **Show group indices** (default on): one hollow ring (8 px, group colour) per visible group at `index_metrics[x]`, `index_metrics[y]`, labelled with the group name first within the budget; hover `<group> (group index)` with both values; a ring with a null axis is omitted and counted. `tabState("risk").rings`. **Done.** With every group visible there is one ring per group whose position equals its `index_metrics`.

### 13f.6 Reference crosshair and strip reference line

Dashed `--chart-ref` lines at the selected benchmark's X value (vertical) and Y value (horizontal) from `benchmarks[].metrics`, drawn only inside the domain, plus a diamond marker labelled with the benchmark label; in the strip a dashed vertical line at the benchmark's value of the strip metric. **Done.** Switching the benchmark moves both lines and the diamond; a benchmark without a value on an axis draws no line on it.

### 13f.7 Hovers

Main scatter: group (or `Group index`), X (bold), Y (bold), 1Y return, Sharpe 3Y. Capture scatter: group, up capture, down capture, `Beta 3Y vs <bench>`. A null shows `—` with its reason. **Done.** Every dot shows these lines.

### 13f.8 Capture parity lines

Besides the `y = x` diagonal, dashed `--chart-ref` lines at 1.0 on both axes. **Done.** Three dashed lines meet at (1, 1).

### 13f.9 Strip metric

A select in the strip panel header over the catalogue; default the shared period id until the reader picks one; independent of the axes. Hash `m`, `tabState("risk").m`. **Done.** Changing the strip metric does not move the scatter; a history change keeps it.

### 13f.10 Drawdown watch heat

Diverging heat cells (UI 9, per column) on `from_52wh`, `from_52wl` and `r_1m`; bars stay on `dd_now` and `maxdd`; the table scrolls inside the panel (about 480 px). **Done.** An ETF 20 % under its high shows a red cell with readable text.

### 13f.11 Screener: column sets

Segmented **Columns** (`tabState("screener").set`): Overview (default), Returns, Risk, Signals, Factor (only when `params.factors` is set), All. Fixed columns before the set: ETF (sticky), Group, Family, history sparkline (13f.14); after it: First date, Last date, Status. The shared period id is always the first metric column.

| Set | Columns |
|---|---|
| Overview | `r_1m r_3m r_ytd r_1y r_3y_ann vol_1y sharpe_3y maxdd_1y z_13w` |
| Returns | `r_1w r_1m r_3m r_6m r_ytd r_1y r_3y_ann r_5y_ann r_10y_ann r_si rel_3m rel_1y` |
| Risk | `vol_1y vol_3y sharpe_3y sortino_3y maxdd_1y maxdd dd_now beta_3y corr_3y up_cap down_cap hit_1y` |
| Signals | `rel_13w z_13w pct_13w mom_12_1 vs_200d from_52wh from_52wl rel_3m rel_1y` |
| Factor | `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y alpha_full alpha_t_full pc1 pc2 rpc1`, then a text column `Cluster` |
| All | the union in this order |

Reference-dependent columns read `ARCADE.ref`. Changing the set resets the sort to the default (period, descending); changing period or history does not. **Done.** Risk shows twelve metric columns plus the period column; a header click survives a history change and is reset by switching to Returns.

### 13f.12 Screener: colour cells

Checkbox **Colour cells** (default on, `tabState("screener").heat`): every column whose registry entry is `signed` gets diverging heat centred on 0 (UI 9, per column over the shown rows, text by contrast); unsigned columns keep plain text. Off restores the `pos`/`neg` classes. **Done.** The 1Y column is a heat column and Vol 1Y stays plain.

### 13f.13 Screener: include group indices

Checkbox **Include group indices** (default off): one row per visible group (key `g:<group>`), ETF cell `<group>` with a muted `group index` tag, values from `index_metrics` with `index_why` reasons, family and status cells blank, included in sort and search, hover-only (not activatable). Count line adds `+ <g> group indices`. **Done.** Ticking adds exactly the visible groups' rows; sorting by 1Y interleaves them.

### 13f.14 Screener: history sparkline

A column headed with the chart history label (`3Y`), not sortable, `narrow: false`: a sparkline (about 90 x 22 px) of the ETF's `adj_close` inside the history window, rebased to 100, in `--accent-2` with a dot on the last point and gaps broken per `LINE_GAP_DAYS`. Series files load lazily per row (`ARCADE.lazy`); no file gives `—` with the reason. Group rows draw from `group_index`. **Done.** Scrolling 520 rows loads only rows near the viewport; changing history redraws the sparklines and keeps the sort.

### 13f.15 Screener: cluster column

In the Factor set, a text column `Cluster` (`funds[].cluster`, `—` when not placed), sortable and searchable. **Done.** Sorting by Cluster groups the rows by cluster name.

### 13f.16 Screener: header hover

Every metric header has `title` = the registry's full label (plus `vs <bench>` for reference-dependent ids); the text is the short label. **Done.** Hovering `Sortino` shows `Sortino ratio 3Y (weekly, vs <cash label>)`.

---

## 13g Market, Fund and global shell

Depends on 13a, 13e (cluster) and 13c (`market.ew_index`).

### 13g.1 As-of line

- **Reader sees.** `<subtitle · >As of <as_of> · <source> · vs <bench> · <priced> of <n> <label> priced, <stale> stale · <g> groups`, then `· groups to <C>` when `C` is before `as_of`. Hovering `groups to <C>` reads: `Group, market, correlation, PCA and rotation outputs end at this cutoff, <CUTOFF_LAG_BDAYS> business days before the latest close, so late-reporting <label> are included; each fund keeps its own last close. <on_cutoff> of <n> have a close on <C>.` The benchmark part is absent without benchmarks.
- **Formula.** `priced` = funds whose status is not `no_data` or `review_broken`; `stale` = funds with status `stale`; counts, never null.
- **Data.** Browser-computed from `funds[].status`, `quality.coverage`, `quality.params.cutoff_lag_bdays`.
- **Done.** With the sample's stale and no-data tickers the line reads `<n-1> of <n> <label> priced, 1 stale`.

### 13g.2 Market brief

- **Reader sees.** One muted sentence under the settings strip on every tab (`aria-live="polite"`, em dash before data): `<bench> returned <x> over <period>. <k> of <g> groups beat it, led by <A> (<a>) and <B> (<b>); <Z> returned <z>. Average <label> correlation is <c> and PC1 explains <p> of 52-week variance. <w> groups show <EW_WARN> or more of 6 early-warning tells against <bench>.`
- **Formula.** `x` = `benchmarks[].returns[period]` (at `C`); `g` = visible groups with a finite `group_return[period]`, `k` those with `group_return[period] > x`; A, B the two highest, Z the lowest (with fewer than 3 groups the Z clause, with 1 the B part, is dropped); `c`, `p` from the last `pca.pca_rolling` row with a non-null `pc1`; `w` = visible groups with `ew_tells[bench] >= params.ew_warn`. A clause whose input is null or missing is left out. Under a group filter the group clauses use the visible groups and the correlation clause is dropped.
- **Done.** Switching the benchmark rewrites the benchmark name, `k` and `w`; hiding a group changes `g` and removes the correlation clause.

### 13g.3 Ten-year chart history

Chart history options YTD, 1Y, 2Y, 3Y, 5Y, 10Y, MAX (default 3Y); 10Y starts at `as_of` minus 120 calendar months; growth charts use a log scale at 5Y, 10Y and MAX. State `history` accepts `10Y`. **Done.** 10Y on a 6-year sample equals MAX in extent and keeps sort, search and fund.

### 13g.4 Collapsed settings summary

Below 900 px the collapsed settings disclosure reads `Settings · <period> · <history>`; Escape, a click outside or focus leaving closes it and returns focus to the summary; opening it closes the group picker and the reverse. **Done.** At 390 px the summary shows `Settings · 3M · 3Y` by default and Escape returns focus to it.

### 13g.5 Tab change scroll

A tab change (click, arrow key, Back) scrolls the window to the top and hides the tooltip; a redraw inside a tab never scrolls. **Done.** Switching from a scrolled Data tab to Market starts at the top; sorting a long table keeps the scroll position.

### 13g.6 Market: large-move note

Above the cards: `<n> <label> have close-to-close moves above <MOVE_WARN> in this range; market statistics include them. See the Data tab.`, where `n` = member funds with a `quality.funds[].moves` date on or after the history start and on or before `C`. Absent when `n` is 0 and under a group filter. **Done.** The sample's large-move ticker produces the note at MAX and not at a history that excludes its move.

### 13g.7 Market: dispersion scale

Segmented `Auto | Linear | Log` on the dispersion card (default Auto; `tabState("market").scale`, kept across history and benchmark changes). Log is a symmetric log scale (linear within +-1 %, logarithmic beyond); Auto uses it when the largest shown value exceeds `DISP_LOG_RATIO` times `max(1 %, median of the shown values)`. The footnote says which scale is in use (`Logarithmic above 1 %, linear near zero; hover for exact values`). **Constants.** `DISP_LOG_RATIO` = 50 (`web/tabs/market.js`). **Done.** One extreme day switches Auto to Log; Linear keeps the spike linear.

### 13g.8 Market: equal weight vs the reference

New card `Equal weight vs <bench>`: one area line around a dashed 100 line, the EW ratio of 13c.6 (`market.ew_index` over the selected benchmark, rebased to 100 at the first date inside the history); kicker `Falling means the reference is ahead`. Full-universe. **Done.** Switching the benchmark redraws it.

### 13g.9 Market: share up over 21 days

- **Reader sees.** New card `Share up over 21 days`: one line, y fixed 0 to 100 %, dashed 50 % line. Full-universe.
- **Formula.** `pos_21d` = share of member funds whose 21-row return (close on `d` over the close `MARKET_DISP_ROWS` own observation rows earlier, minus 1; the input of `dispersion_21d`) is above 0, among members with a finite 21-row return on `d`; minimum 1 fund; null when none; market axis to `C`.
- **data.js.** `market.pos_21d` (array on `market.dates`, fraction or null).
- **Done.** Test: 3 of 4 members up over 21 rows gives 0.75 on that date; a date with no finite 21-row return is null.

### 13g.10 Market: member squares hover

Hovering a square shows `<name> (<ticker>)`, `Rank i of N`, then 1M, 3M, YTD, 1Y; the squares container carries the same text for keyboard users in a visually hidden list. Not clickable. **Done.** The first square shows `Rank 1 of N`.

### 13g.11 Market cards under a group filter

Every card drawn from `market` (breadth, new highs and lows, implied correlation, dispersion, equal weight vs the reference, share up) shows the full-universe note under a group filter; nothing is recomputed. Reference drawdown and cash stay; member squares show the visible ETFs. **Done.** Hiding one group leaves exactly the drawdown, cash and squares cards drawn.

### 13g.12 Group index and benchmark as Fund subjects

- **Reader sees.** The Fund select gains two optgroups before the ETF groups: `Group indices (equal weight)` (visible groups) and `Benchmarks`. Hash `fund=g:<group>` or `fund=b:<benchmark id>` (a ticker never contains `:`); only a ticker opens the drawer. A group subject shows its name, a swatch pill `<n> <label>, equal weight` and a family pill; a benchmark subject its label and the pill `Benchmark` (plus `Selected benchmark` when it is the selected one). Tiles read `groups[].index_metrics` / `index_why` or `benchmarks[].metrics` / `why`. Charts use `group_index` (ends at `C`) or `_bench_<id>`; the monthly grid uses `calendar.groups` or `calendar.benchmarks`; the peers table lists the group's members (hidden for a benchmark); flags, Trades like and Most correlated are hidden.
- **Done.** `#tab=fund&fund=g:<first group>` shows the group index chart and its members; Back returns to the previous ETF.

### 13g.13 Flag chips

After the status pill, the ETF's quality flags as chips (`stale`, `gap`, `spike`, `break`), hover listing the dates behind them (`quality.funds[]` gaps, spikes, `break_date`) and `See the Data tab`. **Done.** The sample's gap ticker shows a `gap` chip whose hover lists the gap dates.

### 13g.14 Trades like

When `funds[].cluster` is set, the header shows the pill `Trades like: <cluster>`, in warning colour when it differs from the ETF's group; hover `Computed cluster of the residual map (Structure & PCA)`. No pill when null. **Done.** An ETF whose cluster equals its group shows a neutral pill.

### 13g.15 KPI tiles

Tiles gain 5Y (annualised), alpha 3Y and its t-statistic (`alpha_t_3y`), `z 13W vs <bench>`, below 52-week high (`from_52wh`) and above 52-week low (`from_52wl`). **Done.** Without `[cash]` alpha shows `—` with `cash_missing`.

### 13g.16 Growth chart: group line

The growth chart adds the ETF's group index (thin, muted, end label the group's short name), rebased at the same first date; it may end before the ETF (it ends at `C`). Absent for group and benchmark subjects. **Done.** The group line starts at 100 on the ETF's first date inside the window.

### 13g.17 Drawdown chart: benchmark line

The drawdown chart adds the selected benchmark's drawdown as a dashed line (peak over the benchmark's full history, METRICS 9 `drawdown_t` rule). **Done.** Switching the benchmark changes only the dashed line.

### 13g.18 Rolling 52-week return and excess

New panel: two lines, the subject's `roll_ret` and `roll_ex` vs the selected benchmark (13a.11), 0 line, end labels, over the chart history. **Done.** With a benchmark as subject the excess line is 0 where defined.

### 13g.19 Rolling beta and correlation

New panel `Rolling beta and correlation vs <bench>`: `roll_beta` and `roll_corr` (13a.11), a solid 0 line and a dotted 1 line; footnote `52-week window on weekly returns; minimum 40 matching weeks. History changes only the displayed dates.` **Done.** A benchmark as its own subject shows beta 1 and correlation 1 once 40 weeks exist.

### 13g.20 Monthly returns on the Fund tab

Full-width panel: the Calendar month grid for the subject (a shared helper `ARCADE.monthGrid`, reused by 13h.8), years from the chart-history start year to the last, one decimal, diverging colours, last column `Year` compounding the months shown (`*` and `n of 12` for part years). Data: `calendar.funds`, `calendar.groups`, `calendar.benchmarks`. **Done.** History 1Y leaves two year rows; the Calendar tab's `series` is unchanged.

### 13g.21 Peers columns

Columns: ETF (current underlined), the shared period, 1M, YTD, 1Y (diverging heat), Vol 1Y and Sharpe 3Y (`narrow: false`), `z 13W vs <bench>` (diverging). Default sort the shared period descending until a header is clicked (`tabState("fund").peerSort`). **Done.** A click on `Vol 1Y` survives a history change and a benchmark switch.

### 13g.22 Most correlated

Panel `Most correlated <label>`: bars of the `FUND_PEERS` highest correlations in the ETF's row of `correlation.corr_3y_weekly` among visible member ETFs (blank pairs skipped, ties by ticker), coloured by sign with the peer's group swatch beside the label (UI 8), value labels; click opens that ETF on the Fund tab. Hover: `<name> (<ticker>)`, correlation, `156 weekly returns to <end>, at least 104 overlapping`. Fixed window, independent of chart history. An ETF outside the matrix shows `Insufficient data: needs 104 weekly returns in the 3-year window`. Hidden for group and benchmark subjects. **Constants.** `FUND_PEERS` = 14 (`web/tabs/fund.js`). **Done.** The panel lists 14 bars high to low and never the ETF itself.

---

## 13h Signal lab, Calendar, Data and review mode

Depends on 13g (month-grid helper) and therefore on every earlier sub-slice. Built last because the review marker works on every tab.

### 13h.1 Signal lab: benchmarks as series

The Series select gains a `Benchmarks` optgroup (`series=b:<id>`). A benchmark against itself has no comparison: Excess over falls back to Nothing and the panels say `<h>-week return`. **Done.** `#tab=lab&series=b:<first id>` draws six panels of the benchmark's own return signal.

### 13h.2 Signal lab: Excess over

- **Reader sees.** Segmented **Excess over**: Reference (default), Own group, Nothing (`tabState("lab").cmp`; history and benchmark changes keep it). Own group = the ETF's group weekly level (for a group or benchmark series it falls back to the reference); Nothing = the plain `h`-week compounded return. The footnote notes that the group index includes the ETF itself. Only Reference at 13 weeks and 156 rolling reads the published arrays; everything else is computed with `ARCADE.lab`.
- **Formula.** `rel_h_cmp = prod(1 + r_i) - prod(1 + c_i)` over `h` weeks (`c` the comparison weekly return), or `prod(1 + r_i) - 1` for Nothing; needs all `2h` (or `h`) returns.
- **Done.** Own group on an ETF titles panel 2 `13-week excess return over <group>` and gives different z values from Reference.

### 13h.3 Signal lab: pipeline strip

Under the controls a numbered text line: `1 Index → 2 <h>W return − <comparison> <h>W` (or `<h>W return`) `→ 3 mean and SD over <n>-week rolling window` (or `expanding window`) `→ 4 z-score → 5 percentile → 6 histogram`; wraps on narrow screens. **Done.** Horizon 26W and Expanding rewrite steps 2 and 3.

### 13h.4 Signal lab: stats tiles

Tiles: Series (full name, ticker in hover), As of (the series' last weekly level date), `<h>W excess`, Window mean, Window SD, z, Percentile in window, z rank in shown history; browser-computed at the final slot from the arrays of the panels. **Done.** Window mean and SD equal panel 3 at the last date.

### 13h.5 Signal lab: histogram

Bins `LAB_HIST_BIN` wide; x domain at least -3 to +3, widened to the next half unit beyond the data; y = share of shown weeks in percent; the bin holding today's z in the series colour, others `--muted`; a vertical marker and `today z <z> · <p>th percentile` (`p` = share of shown z at or below today); bin hover `z <a> to <b>` / `Share of weeks <x> %`. **Rule change.** Bin width 0.25 (UI 13.6 said 0.5). **Constants.** `LAB_HIST_BIN` = 0.25 (`web/tabs/lab.js`). **Done.** The shares of all bins add up to 100 %.

### 13h.6 Signal lab: percentile minimum

METRICS 8: the percentile minimum is `SIGNAL_PCT_MIN` (52) only for the canonical setting; for a custom horizon, window or comparison it equals that setting's z minimum (`max(SIGNAL_CUSTOM_FLOOR, round(SIGNAL_CUSTOM_SHARE * win))`), and `SIGNAL_Z_MIN` (104) for an expanding window. **Done.** A 52-week custom window shows no percentile before 35 finite values.

### 13h.7 Signal lab: cross-section

- **Reader sees.** A **Show** segment `Groups | <label> and groups` (default Groups, the slice-10 chart; `tabState("lab").xs`). The second mode draws one bar per visible ETF's latest z plus one bar per visible group index (outlined `--text`), sorted high to low, coloured by sign (UI 8); the selected series outlined 2 px and labelled with its short name above (positive) or below (negative) its bar; hover full name, z, group; click loads it into the Series control. With canonical settings ETF bars read the published `z_13w` and group bars `index_metrics.z_13w`; with any other setting a button `Recompute for <n> <label>` loads every visible series file (progress `k of n`) and computes with `ARCADE.lab`; until then the footnote reads `Showing the canonical 13-week signal; your settings differ`.
- **Done.** With canonical settings the bars match `z_13w`; group bars are outlined; clicking a group bar sets `series=g:<group>`.

### 13h.8 Calendar: history scope

The monthly grid, the seasonality bars and the calendar-year table show only years from the calendar year of the chart-history start (MAX = all); the compounded column is headed `Shown months`. Seasonality is recomputed in the browser from `calendar.*.monthly_return` over the shown full years (METRICS 7.5 "across shown years"); the published `season_*` arrays remain the MAX values. **Done.** History 3Y on a 6-year series shows 4 calendar years and seasonality counts at most 3 full years.

### 13h.9 Calendar: seasonality hover

Hover lines Average, Median, Positive (share of years up), Years (count of full years); the label stays `<Mon> · <n>% up`. **Done.** Years equals the number of full years shown.

### 13h.10 Calendar: year-table headers

Group columns headed by initials (words split on spaces and `&`), full name in hover; the footnote lists `initials = name` for every visible group and explains `*` (`Fewer than 12 monthly returns in the year; the figure compounds only the months available`). The reference column is headed `Reference`, hover its label. **Done.** Every header has a key entry.

### 13h.11 Calendar: seasonal paths

- **Reader sees.** Panel `Seasonal paths · <series> · <Mon>–<Mon>`, kicker `One card = one year: the window rebased to 100 at its first close; the dot marks the low.` Controls (`tabState("calendar")`: `pSeries pStart pLen pYears`): Series (every benchmark, then every visible group index; default the selected reference), Start month (`From Jan` to `From Dec`, default Sep), Length (1, 2, 3, 6, 12 months; default 2), Years (All, Midterm, Election, Pre-election, Post-election, Last 12; default All; four-year cycle phase `year mod 4`: 0 election, 1 post-election, 2 midterm, 3 pre-election). Years limited to the chart history. One compact line card per year, newest first, two per row on wide screens: daily levels dated in `[first of start month, first of start month + length)`, rebased to 100 at the first close, x fixed to the window, a line at 100, a dot at the minimum; title `YYYY` (an unfinished window `YYYY (so far)`, highlighted with `--text`) and `low <Mon D> · <window return>`. A year with fewer than `SEASON_PATH_MIN_CLOSES` closes in its window, or no first close, is skipped. Stats line over completed windows only: Years, Average window return, Median, Positive share, Median low vs start, Low in first half (share of years whose low is before `start + length x 15 days`). Empty: `No years with data for this window`.
- **Data.** Browser-computed from `_bench_<id>` and `group_index`; the minimum from `params.constants.SEASON_PATH_MIN_CLOSES`.
- **Constants.** `SEASON_PATH_MIN_CLOSES` = 5 (`constants.py`).
- **Done.** Sep, 2 months, All on the reference shows one card per year in the history with the low dot on each card's minimum, and the stats count only completed windows.

### 13h.12 Calendar: fund-by-fund monthly heat

Panel `Monthly returns, fund by fund`, kicker `One cell = one month's return; rows are one group's <label> plus its index and the reference.` Group select (visible groups, default the first; `tabState("calendar").heatGroup`). Rows: the selected benchmark (`Reference`, band `--chart-ref`), the group index (`Group index`, band in the group colour), then the member ETFs by short name; columns the calendar months inside the chart history up to the last published month; cell text one decimal; diverging colour (95th percentile of shown absolute values); blank for null; labels every third month `Mon YY`; canvas above about 2,000 cells. Hover `<full name> · <Mon YYYY>` / Return; activating an ETF row opens the drawer. Data: `calendar.benchmarks`, `calendar.groups`, `calendar.funds`. **Done.** History 2Y shows 24 or 25 columns; a missing month is blank, never zero.

### 13h.13 Data: timeline detail

Rows grouped by group in group order, each block under its group name, rows inside a block by first date. The time axis spans the chart history (MAX = full); a bar starts at the later of its first date and the window start; ETFs whose last date is before the window are left out and counted in the footnote. Hover `<name> (<ticker>)`, From, To, Years (one decimal), Status. **Done.** At 1Y an ETF that ended two years ago is counted in the footnote, not drawn.

### 13h.14 Data: Method content

- **Reader sees.** Method, in this order, every number live: Returns and missing data (adjusted closes, anchors within `ANCHOR_TOL_DAYS`, W-FRI buckets, missing stays missing, `CARRY_DAYS`); Benchmarks (one line each: `<label> = <weight> <ticker> + ...`, weekly rebalanced, the chain ends when a component is missing); Cash (`cash.label`, act/360 weekly accrual, never zero-filled; or `No cash series`); Factors (`params.factors` or `No factor series is configured`); Group indices (equal-weight chain of members, the member filter, end at `C`); Shared settings (group scope across tabs, what is recomputed and what is unavailable, settings stored in this browser and never in links); Metrics (one bullet per window: volatility, Sharpe and beta, signal, correlation, PCA, rotation, market, factor fits, residual map, cutoff lag); Adding <label> (edit `universe.csv`, run `fetch` then `build`; the previous page stays if the build fails); Publication record (generated, source, synthetic).
- **data.js.** `params.constants`: `{name: value}` of every constant in `constants.py` after the project's `[constants]` overrides.
- **Done.** Test: `[constants] STALE_DAYS = 3` gives `params.constants.STALE_DAYS == 3`, and the Method text shows 3.

### 13h.15 Review strip

- **Reader sees.** In review mode a review strip under the settings strip holds: `Mark on click: Off | Broken price | Exclude | Move` (default Off, memory only, reset to Off when review turns off); a `Note` input (at most 500 characters) whose text goes into the next mark's `note` and is then cleared; `Suspects only (N)`; `Discard pending (N)` with the confirm `Discard N pending marks? Built marks stay.`; the export button. With a marker other than Off, activating any ETF mark on any tab (row, tile, card, dot, bar) toggles that pending mark instead of opening the drawer (Broken price = whole series; Move opens the unscoped group select), and the live line reports it. `Suspects only` restricts every ETF list, table, tile set and scatter to ETFs with a quality flag, a non-ok review status or any mark (applied in `ARCADE.scope`); published aggregates are unchanged and footnotes say `Showing N suspect <label>`. Pending and built marks show as per-mode pills on Screener rows and wall cards and as an outline (no text) on treemap tiles.
- **Data.** Browser-computed from `quality.funds[].flags`, `funds[].status`, `review.marks` and pending marks; every tab activates an ETF through `ARCADE.openFund`, where the marker intercepts.
- **Done.** Marker Exclude plus a Screener row click creates one pending exclude mark without opening the drawer; Suspects only on the sample leaves exactly the flagged and marked ETFs.

---

## Coverage table

One row per feature item. IDs: `A-` Overview, Heatmap, Leadership, Chart wall; `B-` Groups, Rotation, Correlation, Structure & PCA; `C-` Risk & return, Signal lab, Screener, Calendar; `D-` header, settings, router, shared helpers, benchmark switch, Market, Fund, Data, review mode, separate review page, keyboard and drawer; `-PS-` project-specific items. "excluded: decision" items are deliberate choices of this plan.

| ID | Item | Status |
|---|---|---|
| A-Overview-1 | Market brief sentence | slice 13g (13g.2) |
| A-Overview-2 | KPI tile: reference return | slice 13b (13b.1) |
| A-Overview-3 | KPI tile: share ahead of the reference | slice 13b (13b.1) |
| A-Overview-4 | KPI tile: share above 200-day | slice 13b (13b.1) |
| A-Overview-5 | KPI tile: average correlation, 52 weeks | slice 13b (13b.1) |
| A-Overview-6 | KPI tile: leading group | slice 5 (already) |
| A-Overview-7 | KPI tile: lagging group | slice 5 (already) |
| A-Overview-8 | Strategy heat treemap | slice 5 (already) |
| A-Overview-9 | ETF hover card context lines | slice 13b (13b.2) |
| A-Overview-10 | 1W return period | slice 13a (13a.5) |
| A-Overview-11 | Group leaderboard | slice 5 (already) |
| A-Overview-12 | Leaderboard hover | slice 13b (13b.3) |
| A-Overview-13 | Leaderboard click opens the group | excluded: decision (group marks hover only, PRODUCT 3) |
| A-Overview-14 | Signal snapshot | slice 13b (13b.4) |
| A-Overview-15 | Group indices chart | slice 5 (already) |
| A-Overview-16 | Line and end-label hover fades the others | slice 13a (13a.12) |
| A-Overview-17 | How much moves together | slice 13b (13b.5) |
| A-Overview-18 | Top 12 / Bottom 12 | slice 5 (already) |
| A-Overview-19 | Top/Bottom z 13W column | slice 13b (13b.6) |
| A-Overview-20 | KPI behaviour under a group filter | slice 13b (13b.1) |
| A-Overview-21 | Group summaries at the cutoff, YTD rule | slice 13a (13a.7) |
| A-Heatmap-1 | Colour by additions | slice 13b (13b.8) |
| A-Heatmap-2 | Colour scale Tight / Auto / Wide | slice 13b (13b.9) |
| A-Heatmap-3 | Size by additions | slice 13b (13b.10) |
| A-Heatmap-4 | Group by computed cluster | slice 13b (13b.11) |
| A-Heatmap-5 | Tile value line | slice 13b (13b.12) |
| A-Heatmap-6 | Block header median, measure in the title | slice 13b (13b.13) |
| A-Heatmap-7 | Treemap hover card | slice 13b (13b.2) |
| A-Heatmap-8 | Groups across periods: 1W, 5Y, 10Y | slice 13b (13b.14) |
| A-Heatmap-9 | Performance grid | slice 13b (13b.15) |
| A-Heatmap-10 | Grid: keep groups together | slice 13b (13b.15) |
| A-Heatmap-11 | Grid: filter | slice 13b (13b.15) |
| A-Leadership-1 | Scatter axes | slice 7 (already) |
| A-Leadership-2 | Leader and laggard zones | slice 7 (already) |
| A-Leadership-3 | Dot colour (group or computed cluster) | slice 13c (13c.1) |
| A-Leadership-4 | Label mode All / Some / None | slice 13c (13c.1) |
| A-Leadership-5 | Label priority | slice 13c (13c.2) |
| A-Leadership-6 | Group-index rings toggle and position | slice 13c (13c.1) |
| A-Leadership-7 | Legend chips | slice 13c (13c.1) |
| A-Leadership-8 | Scatter hover | slice 13c (13c.3) |
| A-Leadership-9 | Early-warning cards (core) | slice 7 (already) |
| A-Leadership-10 | Early-warning sort | slice 13c (13c.5) |
| A-Leadership-11 | Direction arrow | slice 13c (13c.4, 13c.5) |
| A-Leadership-12 | Streak line | slice 13c (13c.4, 13c.5) |
| A-Leadership-13 | Tell dots | slice 13c (13c.5) |
| A-Leadership-14 | 3M relative line | slice 13c (13c.4, 13c.5) |
| A-Leadership-15 | Tell offsets 63 and 21 rows | slice 13c (13c.4) |
| A-Leadership-16 | Groups without usable history note | slice 13c (13c.5) |
| A-Leadership-17 | Participation: ETFs above 200-day | slice 7 (already) |
| A-Leadership-18 | Participation: groups above 200-day | slice 13c (13c.6) |
| A-Leadership-19 | Stretched and washed-out bands | slice 7 (already) |
| A-Leadership-20 | Equal weight vs reference chart | slice 13c (13c.6) |
| A-Leadership-21 | Linked participation charts | slice 7 (already) |
| A-Leadership-22 | Participation legend with latest values | slice 13c (13c.6) |
| A-Leadership-23 | Participation recomputed under a group filter | excluded: decision (full-universe note, PRODUCT 4.3) |
| A-Wall-1 | Twelve chart kinds | slice 13d (13d.1) |
| A-Wall-2 | Range follows the chart history | slice 6 (already) |
| A-Wall-3 | Group by Group / Family / Cluster / None | slice 13d (13d.2) |
| A-Wall-4 | Order groups by | slice 13d (13d.2) |
| A-Wall-5 | Sort ETFs by | slice 6 (already) |
| A-Wall-6 | Direction | slice 6 (already) |
| A-Wall-7 | Card height | slice 6 (already) |
| A-Wall-8 | Y scale Free / Per group / Shared | slice 13d (13d.3) |
| A-Wall-9 | Compare with | slice 13d (13d.4) |
| A-Wall-10 | Search | slice 13d (13d.5) |
| A-Wall-11 | Group index first | slice 13d (13d.6) |
| A-Wall-12 | Summary line | slice 13d (13d.7) |
| A-Wall-13 | Section header average | slice 13d (13d.7) |
| A-Wall-14 | Section collapse | slice 13d (13d.7) |
| A-Wall-15 | Card header value and sub-label | slice 13d (13d.8) |
| A-Wall-16 | Reference lines per kind | slice 13d (13d.1) |
| A-Wall-17 | Log scale at long ranges | slice 6 (already) |
| A-Wall-18 | Gradient area fills | excluded: decision (no gradients on marks, UI 9) |
| A-Wall-19 | Aligned x domain | slice 13d (13d.3) |
| A-Wall-20 | Lazy cards, click, crosshair | slice 6 (already) |
| A-Wall-21 | Empty card texts | slice 6 (already) |
| A-PS-1 | Universe or cohort selector and its options | excluded: project-specific |
| A-PS-2 | Stated-label options and hover line | excluded: project-specific |
| A-PS-3 | Benchmarks shown as a pseudo fund group | excluded: project-specific |
| A-PS-4 | Review marking on tiles, cards and rows (the separate review page is D-J-1) | slice 13h (13h.15) |
| A-PS-5 | Currency-specific wording | excluded: project-specific |
| A-PS-6 | Group colour by universe | excluded: project-specific |
| A-PS-7 | Dual data packs and share-class selection | excluded: project-specific |
| A-PS-8 | Cohort coverage header (generic counts: D-A-2) | excluded: project-specific |
| B-Groups-1 | Shared legend chips | slice 6 (already) |
| B-Groups-2 | History control | slice 6 (already) |
| B-Groups-3 | Log-scale toggle | slice 13d (13d.9) |
| B-Groups-4 | Growth of 100 | slice 6 (already) |
| B-Groups-5 | Relative to the reference | slice 6 (already) |
| B-Groups-6 | Drawdown from peak | slice 6 (already) |
| B-Groups-7 | Line hover and sorted crosshair | slice 6 (already) |
| B-Groups-8 | Rolling 52-week statistic panel | slice 13d (13d.10) |
| B-Groups-9 | Excess z-score heat over time | slice 13d (13d.11) |
| B-Groups-10 | Set selector with Alpha | slice 13d (13d.12) |
| B-Groups-11 | Returns set (5Y, 10Y, rel 1Y) | slice 13d (13d.12) |
| B-Groups-12 | Risk set on the group index | slice 13d (13d.12) |
| B-Groups-13 | Alpha set | slice 13d (13d.12; data 13e.2) |
| B-Groups-14 | Breadth set with median z | slice 13d (13d.12; data 13a.7) |
| B-Groups-15 | Row click opens the group | excluded: decision (group marks hover only, PRODUCT 3) |
| B-Groups-16 | Inside each group | slice 6 (already) |
| B-Rotation-1 | Method intro | slice 7 (already) |
| B-Rotation-2 | Group quadrant chart | slice 7 (already) |
| B-Rotation-3 | Tail slider | slice 7 (already) |
| B-Rotation-4 | End-week slider | slice 7 (already) |
| B-Rotation-5 | Axis domain | slice 7 (already) |
| B-Rotation-6 | Stale heads | slice 13c (13c.7) |
| B-Rotation-7 | Head click selects the group | slice 13c (13c.7) |
| B-Rotation-8 | ETF rotation inside a group | slice 13c (13c.8) |
| B-Rotation-9 | Leadership ranks by return period | slice 13c (13c.9) |
| B-Rotation-10 | Leadership table | slice 7 (already) |
| B-Rotation-11 | Monthly returns by group heat | slice 13c (13c.10) |
| B-Correlation-1 | Windows 1Y daily, 3Y weekly | slice 9 (already) |
| B-Correlation-2 | 3Y residual window | slice 13e (13e.5) |
| B-Correlation-3 | Clustered or group order | slice 9 (already) |
| B-Correlation-4 | Matrix rendering and hover | slice 9 (already) |
| B-Correlation-5 | Sub-matrix under a group filter | slice 9 (already) |
| B-Correlation-6 | Group correlation matrix | slice 13e (13e.8) |
| B-Correlation-7 | Correlation regime | slice 13e (13e.9) |
| B-Correlation-8 | Rolling correlation to the reference | slice 13e (13e.10) |
| B-Correlation-9 | Closest and furthest bars | slice 9 (already) |
| B-Correlation-10 | ETF select | slice 13e (13e.6) |
| B-Correlation-11 | Pair lists | slice 9 (already) |
| B-Correlation-12 | Pairs mode and group columns | slice 13e (13e.7) |
| B-Structure-1 | Rolling PC1 to PC3 | slice 9 (already) |
| B-Structure-2 | Top-5 absorption ratio | slice 13e (13e.3, 13e.12) |
| B-Structure-3 | PC1 ex-market | slice 13e (13e.3, 13e.12) |
| B-Structure-4 | Scree, raw | slice 9 (already) |
| B-Structure-5 | Scree, market-residual | slice 13e (13e.13) |
| B-Structure-6 | Colour by computed cluster | slice 13e (13e.14) |
| B-Structure-7 | Group outlines toggle | slice 13e (13e.14) |
| B-Structure-8 | Shared legend for both scatters | slice 13e (13e.14) |
| B-Structure-9 | Residual map (MDS) | slice 13e (13e.4, 13e.15) |
| B-Structure-10 | Loadings, raw | slice 9 (already) |
| B-Structure-11 | Loadings, market-residual | slice 13e (13e.16) |
| B-Structure-12 | Alpha forest | slice 13e (13e.17) |
| B-Structure-13 | Alpha against residual volatility | slice 13e (13e.18) |
| B-Structure-14 | Alpha table | slice 13e (13e.18) |
| B-Structure-15 | Group vs computed cluster | slice 13e (13e.4, 13e.19) |
| B-Structure-16 | Filter behaviour | slice 9 (already) |
| B-PS-1 | Universe column and colour by universe | excluded: project-specific |
| B-PS-2 | Colour by stated label | excluded: project-specific |
| B-PS-3 | Currency-specific bond factor and price conversion (generic: `[factors]`, `[cash]`) | excluded: project-specific |
| B-PS-4 | Group-universe membership and its saved filter | excluded: project-specific |
| C-Risk-1 | Axis catalogue and dynamic title | slice 13f (13f.1) |
| C-Risk-2 | Axis return id writes the shared period | excluded: decision (PRODUCT 3: tab metrics never change shared settings) |
| C-Risk-3 | Colour by | slice 13f (13f.2) |
| C-Risk-4 | Dot size | slice 13f (13f.3) |
| C-Risk-5 | Label mode | slice 13f (13f.4) |
| C-Risk-6 | Group-index rings | slice 13f (13f.5) |
| C-Risk-7 | Legend chips | slice 13f (13f.4) |
| C-Risk-8 | Main scatter base | slice 8 (already) |
| C-Risk-9 | Reference crosshair | slice 13f (13f.6) |
| C-Risk-10 | Scatter hover | slice 13f (13f.7) |
| C-Risk-11 | Click | slice 8 (already) |
| C-Risk-12 | Up vs down capture scatter | slice 8 (already) |
| C-Risk-13 | Capture parity lines | slice 13f (13f.8) |
| C-Risk-14 | Capture hover with beta | slice 13f (13f.7) |
| C-Risk-15 | Distribution strip base | slice 8 (already) |
| C-Risk-16 | Strip metric select | slice 13f (13f.9) |
| C-Risk-17 | Strip reference line | slice 13f (13f.6) |
| C-Risk-18 | Drawdown watch base | slice 8 (already) |
| C-Risk-19 | Drawdown watch heat cells | slice 13f (13f.10) |
| C-Risk-20 | Layout | slice 8 (already) |
| C-Risk-21 | Drawdown, 52-week, volatility data | slice 7 (already) |
| C-Risk-22 | Capture data | slice 8 (already) |
| C-Risk-23 | Extended per-ETF metrics | slice 13a (13a.3) |
| C-Risk-24 | Factor regression | slice 13e (13e.2) |
| C-Risk-25 | Group-index metrics | slice 13a (13a.6) |
| C-Risk-26 | Benchmark metrics | slice 13a (13a.8) |
| C-Lab-1 | Benchmarks as series | slice 13h (13h.1) |
| C-Lab-2 | Horizon | slice 10 (already) |
| C-Lab-3 | Excess over | slice 13h (13h.2) |
| C-Lab-4 | z window | slice 10 (already) |
| C-Lab-5 | Rolling or expanding | slice 10 (already) |
| C-Lab-6 | Show follows the chart history | slice 10 (already) |
| C-Lab-7 | Pipeline strip | slice 13h (13h.3) |
| C-Lab-8 | Stats tiles | slice 13h (13h.4) |
| C-Lab-9 | Panel 1 index | slice 10 (already) |
| C-Lab-10 | Panel 2 excess | slice 10 (already) |
| C-Lab-11 | Panel 3 bands | slice 10 (already) |
| C-Lab-12 | Panel 4 z | slice 10 (already) |
| C-Lab-13 | Panel 5 percentile | slice 10 (already) |
| C-Lab-14 | Panel 6 histogram | slice 13h (13h.5) |
| C-Lab-15 | Percentile minimum for custom windows | slice 13h (13h.6) |
| C-Lab-16 | Cross-section with ETF and group bars | slice 13h (13h.7) |
| C-Lab-17 | Full-history statistics | slice 10 (already) |
| C-Lab-18 | Insufficient data | slice 10 (already) |
| C-Lab-19 | Layout | slice 10 (already) |
| C-Screener-1 | Column sets | slice 13f (13f.11) |
| C-Screener-2 | Period column first | slice 2 (already) |
| C-Screener-3 | Group filter | slice 2 (already) |
| C-Screener-4 | Search | slice 1 (already) |
| C-Screener-5 | Colour cells | slice 13f (13f.12) |
| C-Screener-6 | Include group indices | slice 13f (13f.13) |
| C-Screener-7 | Sparkline column | slice 13f (13f.14) |
| C-Screener-8 | ETF column | slice 1 (already) |
| C-Screener-9 | Group column | slice 1 (already) |
| C-Screener-10 | Cluster column | slice 13f (13f.15) |
| C-Screener-11 | Header hover | slice 13f (13f.16) |
| C-Screener-12 | Sort rule | slice 2 (already) |
| C-Screener-13 | Count line | slice 1 (already) |
| C-Screener-14 | Row click | slice 2 (already) |
| C-Screener-15 | Sticky scroll | slice 1 (already) |
| C-Screener-16 | Data for the sets | slice 13a (13a.3, 13a.4; factor ids 13e.2) |
| C-Calendar-1 | Series select | slice 7 (already) |
| C-Calendar-2 | Monthly grid | slice 7 (already) |
| C-Calendar-3 | History scope | slice 13h (13h.8) |
| C-Calendar-4 | Seasonality hover | slice 13h (13h.9) |
| C-Calendar-5 | Year-table initials and key | slice 13h (13h.10) |
| C-Calendar-6 | Seasonal paths | slice 13h (13h.11) |
| C-Calendar-7 | Fund-by-fund monthly heat | slice 13h (13h.12) |
| C-Calendar-8 | Monthly return rule | slice 7 (already) |
| C-Calendar-9 | Seasonality bars | slice 7 (already) |
| C-PS-1 | Risk colour by stated label | excluded: project-specific |
| C-PS-2 | Risk colour by universe | excluded: project-specific |
| C-PS-3 | Benchmarks and factor proxies as scatter rows (generic: 13f.6) | excluded: project-specific |
| C-PS-4 | Currency-converted levels and single-currency cash | excluded: project-specific |
| C-PS-5 | Lab index titled as a converted index | excluded: project-specific |
| C-PS-6 | Screener universe select | excluded: project-specific |
| C-PS-7 | Screener benchmark rows | excluded: project-specific |
| C-PS-8 | Factor proxies as seasonal-path series (generic: benchmarks, 13h.11) | excluded: project-specific |
| D-A-1 | Title and kicker | slice 1 (already) |
| D-A-2 | Cutoff line counts and hover | slice 13g (13g.1) |
| D-A-3 | As-of line: benchmark, priced, stale | slice 13g (13g.1) |
| D-A-4 | Market brief on every tab | slice 13g (13g.2) |
| D-A-5 | Scope note | slice 2 (already) |
| D-A-6 | Reload text while switching reference or universe | excluded: project-specific |
| D-B-1 | Benchmark select | slice 4 (already) |
| D-B-2 | Chart history 10Y | slice 13g (13g.3) |
| D-B-3 | Group picker | slice 2 (already) |
| D-B-4 | Collapsed settings summary | slice 13g (13g.4) |
| D-B-5 | Names | slice 2 (already) |
| D-B-6 | Review toggle | slice 12 (already) |
| D-B-7 | Return period 1W | slice 13a (13a.5) |
| D-B-8 | Period change moves local return metrics | slice 2 (already) |
| D-B-9 | Invalid saved settings | slice 2 (already) |
| D-B-10 | Hiding the open fund's group resets it | excluded: decision (UI 4 keeps `fund` with an outside-filter pill) |
| D-B-11 | Empty scope | slice 2 (already) |
| D-C-1 | Fifteen tabs | slice 1 (already) |
| D-C-2 | Arrow keys | slice 1 (already) |
| D-C-3 | Tab change scrolls to top | slice 13g (13g.5) |
| D-C-4 | Address hash | slice 2 (already) |
| D-C-5 | History entries | slice 2 (already) |
| D-C-6 | Switch by page reload with a view bridge | excluded: project-specific |
| D-C-7 | Coalesced redraw | slice 2 (already) |
| D-D-1 | Tooltip node | slice 2 (already) |
| D-D-2 | Crosshair line cap | slice 13a (13a.12) |
| D-D-3 | End labels with value and hover highlight | slice 13a (13a.12) |
| D-D-4 | Legend chips | slice 6 (already) |
| D-D-5 | Line chart options | slice 3 (already) |
| D-D-6 | Empty texts | slice 3 (already) |
| D-D-7 | Loading and failed series files | slice 13a (13a.13) |
| D-D-8 | Filtered-selection note | slice 4 (already) |
| D-D-9 | Horizontal bars | slice 5 (already) |
| D-D-10 | Table helper | slice 5 (already) |
| D-D-11 | Names | slice 2 (already) |
| D-D-12 | Colours | slice 2 (already) |
| D-D-13 | Lazy panels | slice 6 (already) |
| D-D-14 | Range control in the strip | slice 2 (already) |
| D-E-1 | Benchmark switch preserves state | slice 4 (already) |
| D-E-2 | Benchmark as Fund subject with role pill | slice 13g (13g.12) |
| D-F-1 | Range follows the history | slice 4 (already) |
| D-F-2 | Large-move note | slice 13g (13g.6) |
| D-F-3 | Intro | slice 4 (already) |
| D-F-4 | Breadth card | slice 4 (already) |
| D-F-5 | New highs and lows | slice 4 (already) |
| D-F-6 | Implied correlation | slice 4 (already) |
| D-F-7 | Dispersion scale toggle | slice 13g (13g.7) |
| D-F-8 | Equal weight vs reference card | slice 13g (13g.8; data 13c.6) |
| D-F-9 | Cash rate | slice 4 (already) |
| D-F-10 | Reference drawdown | slice 4 (already) |
| D-F-11 | Share up over 21 days | slice 13g (13g.9) |
| D-F-12 | Member squares hover | slice 13g (13g.10) |
| D-F-13 | Market cards under a group filter | slice 13g (13g.11) |
| D-F-14 | Market ends at the cutoff | slice 4 (already) |
| D-G-1 | Group index and benchmark subjects | slice 13g (13g.12) |
| D-G-2 | Range follows the history | slice 3 (already) |
| D-G-3 | Header name | slice 3 (already) |
| D-G-4 | Role and group-subject pills | slice 13g (13g.12) |
| D-G-5 | Flag chips | slice 13g (13g.13) |
| D-G-6 | Trades like pill | slice 13g (13g.14; data 13e.4) |
| D-G-7 | Identifier and share-class lines | excluded: project-specific |
| D-G-8 | Review actions under the header | slice 12 (already) |
| D-G-9 | KPI tiles: 5Y, alpha, t, z, 52-week | slice 13g (13g.15) |
| D-G-10 | Growth chart group line | slice 13g (13g.16) |
| D-G-11 | Drawdown chart benchmark line | slice 13g (13g.17) |
| D-G-12 | Rolling return and excess | slice 13g (13g.18) |
| D-G-13 | Rolling beta and correlation | slice 13g (13g.19) |
| D-G-14 | Monthly returns on the Fund tab | slice 13g (13g.20) |
| D-G-15 | Peers columns | slice 13g (13g.21) |
| D-G-16 | Most correlated ETFs | slice 13g (13g.22) |
| D-G-17 | Empty states | slice 3 (already) |
| D-H-1 | History timeline detail | slice 13h (13h.13) |
| D-H-2 | Coverage table | slice 11 (already) |
| D-H-3 | Search | slice 11 (already) |
| D-H-4 | Review queue | slice 12 (already) |
| D-H-5 | Method: returns and carry | slice 11 (already) |
| D-H-6 | Method: benchmarks | slice 13h (13h.14) |
| D-H-7 | Method: group indices | slice 13h (13h.14) |
| D-H-8 | Method: shared settings | slice 13h (13h.14) |
| D-H-9 | Method: metric list with live windows | slice 13h (13h.14) |
| D-H-10 | Method: adding ETFs | slice 13h (13h.14) |
| D-I-1 | Review toggle | slice 12 (already) |
| D-I-2 | Marker mode on every tab | slice 13h (13h.15) |
| D-I-3 | Note input | slice 13h (13h.15) |
| D-I-4 | Suspects only | slice 13h (13h.15) |
| D-I-5 | Counts | slice 12 (already) |
| D-I-6 | Export file | slice 12 (already) |
| D-I-7 | Discard pending | slice 13h (13h.15) |
| D-I-8 | Badges on rows, cards and tiles | slice 13h (13h.15) |
| D-I-9 | Review hover line | slice 12 (already) |
| D-I-10 | Pending, built and cleared layers | slice 12 (already) |
| D-I-11 | Marks table | slice 12 (already) |
| D-I-12 | Taxonomy-leaf move picker | excluded: project-specific |
| D-J-1 | Separate review page and its server | excluded: project-specific |
| D-K-1 | Print | slice 1 (already) |
| D-K-2 | Export | slice 12 (already) |
| D-K-3 | Keyboard | slice 2 (already) |
| D-K-4 | Detail drawer | slice 2 (already) |
| D-PS-1 | Universe select, per-universe groups, reload-based switch | excluded: project-specific |
| D-PS-2 | Currency label, converted histories, named rate | excluded: project-specific |
| D-PS-3 | Stated-label pill, identifier lines, fixed proxy ids | excluded: project-specific |
| D-PS-4 | Review commit endpoint and catalog-keyed marks | excluded: project-specific |
| D-PS-5 | Separate review page, its server and the settings link | excluded: project-specific |
| D-PS-6 | Catalog-import wording for adding funds | excluded: project-specific |
| D-PS-7 | Loader internals (binary packs, cache, prefetch) | excluded: project-specific |

Totals: 326 items; 127 built by slices 1 to 12 (already), 162 built by slice 13 (13a 11, 13e 19, 13b 21, 13c 21, 13d 21, 13f 19, 13g 26, 13h 24), 37 excluded (31 project-specific, 6 decision). A-Overview-1 and D-A-4 are one feature (13g.2), as are A-Overview-9 and A-Heatmap-7 (13b.2).
