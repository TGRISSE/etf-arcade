# etf-arcade metric reference

Lookup sheet for every formula in etf-arcade. Slice numbers refer to the slice plan (1 skeleton and Screener, 2 chrome and settings, 3 Fund tab and charts, 4 benchmarks, cash and Market, 5 Overview and Heatmap, 6 Chart wall and Groups, 7 Leadership, Rotation and Calendar, 8 Risk and return, 9 Correlation and PCA, 10 Signal lab, 11 Data tab and data-quality screen, 12 review marks, 13a to 13h parity; build order 1, 1b, 2, 3, 11, 12, 4 to 10, then 13a, 13e, 13b, 13c, 13d, 13f, 13g, 13h). Slice-13 formulas are in `docs/reference/PARITY.md` (section 13 below lists their ids).

Rows marked **S1** are the slice-1 metrics. Their definitions here are exact: slice-1 code must follow this page.

## 1. Global conventions

| Rule | Value |
| --- | --- |
| Price input | `adj_close` per ticker from `<project>/var/prices.csv` (long format: date, ticker, adj_close). Nothing else feeds the core. |
| Currency | Each fund's closes are in its own listing currency (`universe.csv` `currency`, a label). No FX conversion exists, and no metric mixes currencies except through a benchmark the project chose. |
| Level `L_t`, return `r_t` | Level is the adjusted close. Return is a decimal simple return `L_t / L_prev - 1`. Display in percent. |
| Missing | `null` in JSON, em dash in the UI, reason in hover text. Never 0, never an interpolated number. |
| Carry | A missing close is never filled into a return, an index or a statistic, with two bounded exceptions. (1) The cash rate (section 4). (2) Alignment: where a statistic reads a level on a date grid (the W-FRI weekly buckets, the business-day grid of `vs_200d`, a benchmark level on a fund's dates), the grid date takes the last close at most `CARRY_DAYS` (7) calendar days earlier and never past the series' last close; older is null. A longer gap stays a gap. |
| No splicing | Two tickers are never joined into one history, even when one replaced the other. |
| Sample statistics | Standard deviation and covariance use `ddof = 1`. |
| Calendar | Weekly buckets are W-FRI: the last close in each Saturday-to-Friday bucket, labelled with that Friday. |
| Partial week | A bucket whose Friday label is after the ticker's own last close is dropped before any weekly return is formed; a fund whose last close is a Wednesday ends its weekly series at the previous Friday. For group-level and cross-sectional weekly outputs the limit is the cutoff `C` instead. A week is never reported half-finished. This applies to every weekly statistic (`vol_1y`, `sharpe_3y`, `beta_3y`, `up_cap`, `down_cap`, the signal of section 8, the weekly arrays of section 9) and to the weekly cash return. |
| Weekly return | `W_k / W_{k-1} - 1`, null if either weekly level is null. A missing week nulls two returns and is not bridged. |
| Daily return | `L_t / L_prev - 1` only when the previous observed close is at most 7 calendar days earlier (covers weekends and holidays). Otherwise null. |
| Annualisation | 52 weekly periods per year (`ANNUAL_WEEKS`). Volatility and Sharpe use `sqrt(52)`. |
| Anchor | Each ticker is evaluated at its own last close `t_end` (weekly statistics at its last complete week, see Partial week). Every group-level and cross-sectional output is evaluated at the coverage cutoff `C` (section 10), so a fund that reports late or early never shifts a group figure. Staleness against the universe is shown separately (`days_stale`), never hidden. |
| Panel `as_of` | Latest close date across all tickers in the universe. |
| Rounding | Compute at full precision. Round only for display (percent 1 dp, ratios 2 dp, correlations 2 dp). The one stored exception: correlation matrices, rounded to 3 dp (slice 9). |
| Units | Every stored number is a decimal fraction: returns, `rel_*` differences, shares, rates (`0.052` = 5.2%). Beta, Sharpe, z, capture ratios (1 = parity), correlations and the `rs` / `rrg_*` index levels (100 = parity) are plain numbers. The UI formats. |
| Synthetic data | When `source` is `synthetic`, every output carries `synthetic: true`. The header names the source; no banner. |

## 2. Constants

Every window, minimum and span in this document is a named constant in `etf_arcade/constants.py` (listed in DATA.md section 12). Main values: 52 annual weeks, 7-day anchor tolerance, volatility 52 slots / 40 minimum, summary 156 / 104, rolling 52 / 40, signal horizon 13 with 156 / 104 / 52, MAs 200 and 50, daily correlation 252 / 200, PCA 104 weeks (fixed) and 52 (rolling) with 8 ETFs minimum, rotation EMA 10 / 40, lag 4, 60 levels minimum.

## 3. Per-ETF metrics

### 3.1 Return anchors

`back(months)`: target date `T = t_end - DateOffset(months)`. The anchor level is the last close dated on or before `T`, used only if `T - date <= 7` calendar days. Otherwise null. Null therefore covers both "history does not reach the anchor" and "the last close before the anchor is stale".

Return horizons: 1M, 3M, 6M, 1Y are 1, 3, 6, 12 calendar months; 3Y is 36 months; YTD anchors at 31 December of the prior year. From slice 13a: 1W anchors at `t_end - 7` calendar days (same tolerance), 5Y and 10Y are 60 and 120 months, annualised like 3Y.

### 3.2 Table

| id | Definition | Window | Freq | Min obs | Null when | Tab / slice |
| --- | --- | --- | --- | --- | --- | --- |
| `r_1m` **S1** | `P / back(1M) - 1`, `P` = last close | 1 month | daily levels | anchor exists | no anchor within tolerance | Screener, Heatmap, Overview, Market (S1, S5) |
| `r_3m` **S1**, `r_6m` **S1** | same with 3 and 6 months | 3 / 6 months | daily levels | anchor | as above | as above |
| `r_1y` **S1** | `P / back(12M) - 1` | 12 months | daily levels | anchor | as above | as above |
| `r_ytd` **S1** | `P / L0 - 1`, `L0` = last close on or before 31 Dec of the prior year | year to date | daily levels | `L0` exists | `L0` more than 7 days before 31 Dec | as above |
| `r_3y_ann` **S1** | `(P / back(36M))^(1/3) - 1`, annualised | 36 months | daily levels | anchor | no anchor, so less than 3 years of history | Screener (S1), Heatmap |
| `r_si` | `(P / P0)^(1/Y) - 1`, `P0` first close, `Y = days / 365.25` | full history | daily levels | `Y > 0.5` | `Y <= 0.5` | Fund (S3) |
| `vol_1y` **S1** | `std(weekly returns, ddof 1) * sqrt(52)` | last 52 weekly slots up to `t_end` | weekly | 40 finite returns | fewer than 40 finite in the 52 slots | Screener (S1), Risk (S8) |
| `dd_now` | `L_t / max(L_s, s <= t) - 1` at `t_end` | full history | daily | 1 close | no closes | Risk, Fund (S3, S8) |
| `maxdd` | `min over t of (L_t / running max - 1)` | full history | daily | 2 closes | fewer than 2 closes | Risk, Fund |
| `maxdd_1y` **S1** | `min over t in window of (L_t / running max inside the window - 1)`; the running max starts at the first close in the window | closes dated on or after `t_end - 12 months` | daily | `DD_MIN_CLOSES` = 200 closes in the window | fewer than 200 closes in the window, or the first close in the window is more than 7 days after `t_end - 12 months` | Screener (S1) |
| `first_date` **S1** | first date with a close | full | n/a | 1 close | no closes | Screener (S1), Data (S11) |
| `last_date` **S1** | `t_end`, last date with a close | full | n/a | 1 close | no closes | Screener (S1) |
| `days_stale` **S1** | `as_of - last_date` in calendar days, `as_of` from section 1 | n/a | n/a | 1 close | no closes | Screener (S1), Data (S11) |
| `from_52wh` | `P / max(closes in the last calendar year) - 1` (at most 0) | last calendar year | daily | 100 closes in the window | fewer than 100 closes in the window | Leadership (S7) |
| `from_52wl` | `P / min(closes in the last calendar year) - 1` (at least 0) | last calendar year | daily | 100 closes | as above | Leadership |
| `vs_200d` | `P / mean(levels on the last 200 grid dates) - 1`; the grid is every business day (Monday to Friday) plus every observed date up to `t_end`, each grid date reading the last close at most `CARRY_DAYS` earlier (section 1, Carry) | 200 grid dates (`MARKET_MA_LONG`) | daily | 200 grid levels | fewer than 200 grid levels; reason `insufficient_history` | Overview, Market |
| `mom_12_1` | `back(1M-anchored level) / back(12M-anchored level) - 1`, momentum skipping the last month | 12M minus 1M | daily levels | both anchors | either anchor null; reason `no_history_to_anchor` | Heatmap, Signal lab |

Slice-1 notes:
- `vol_1y` counts 52 weekly slots, not 52 finite values; a null week lowers the count toward the 40 minimum.
- `maxdd_1y` is a slice-1 contract choice. `maxdd` (full history, no window) is the general metric. They must have different ids and labels ("Max DD 1Y" and "Max drawdown").
- Until slice 11 lands no spike or gap screening applies (section 10): figures are raw. Slice 11 adds the screen and the Data tab that flags suspect histories.
- `back(3M)` uses a calendar offset, not 63 trading days. A 30-day approximation is not allowed.

## 4. Benchmarks and cash (slice 4)

Benchmarks are declared in the project's `settings.toml` as an ordered list `[[benchmarks]]` (`id`, `label`, `components = {ticker = weight}`; DATA.md section 2b). The first entry is the default; the UI has a benchmark switch (UI.md section 4) and every reference-dependent output is computed once per benchmark and stored keyed by benchmark id (DATA.md section 9). In the formulas below `REF` is the weekly level or return series of any one configured benchmark. Component tickers are ordinary series in `prices.csv`; a component may also be a universe member and is then the very same series, with no special case. The default project configures `SPY`, `AGG` and a 60/40 blend of the two; `examples/europe` configures its own.

| id | Definition | Freq | Null when |
| --- | --- | --- | --- |
| Single-component benchmark | The adjusted close of its one ticker, as a level rebased to 100 at its first close, computed by the same weekly-rebalance chain as a blend (one component, weight 1). | daily levels | missing close |
| Blend | The weights of `components` (summing to 1), rebalanced weekly (below). Level starts at 100. | daily levels, weekly rebalancing | any component missing in the bucket |

Weekly-rebalanced blend (any number of components, weights `w_c`):
1. Rebalance date of each W-FRI bucket = the last date in the bucket where every component has a close.
2. Weekly return `b_k = sum_c w_c * r_c,k` using weekly returns between consecutive rebalance dates. All components required, no renormalising of weights.
3. Level `B_k = B_{k-1} * (1 + b_k)`, `B_0 = 100`. Within a week the daily path is buy-and-hold: `B_rebalance * sum_c w_c * P_c / P_c,rebalance`.
4. A bucket missing any component ends the chain at that point and the level stays null for the rest of the series. The chain is never restarted, because a restart would splice two histories.

Cash:
- Source: the `[cash]` series of the project (`ticker`, `label`, `kind = "annual_yield_pct"`): an annual yield in percent per annum, for example the 13-week T-bill discount rate. The UI and the docs name it by `cash.label` only; it is not a total-return cash index.
- Its values in `prices.csv` may be zero or negative; only the cash ticker is exempt from the `adj_close > 0` rule.
- The published cash series is the annual rate as a decimal fraction (`y / 100`).
- Daily rate series: each fixing is carried to following calendar days for at most 7 days and never past the last observation. The bound is the alignment carry of section 1 (`CARRY_DAYS`).
- Weekly cash return for Friday `d`: `c_d = product over days j in [d-7, d-1] of (1 + y_j / 100 / 360) - 1` (act/360). The rate of day `j` earns `j` to `j+1`.
- If any of those 7 days has no bounded rate, `c_d` is null. Cash is never zero-filled.
- Any metric needing cash is null when any cash value in its window is null (`cash_incomplete`), and null with `cash_missing` when the project has no `[cash]` or its ticker has no rows.

Reference-dependent outputs (each stored as an object keyed by benchmark id): per fund `rel_3m rel_1y beta_3y up_cap down_cap rel_13w z_13w pct_13w z_rank` and, from slice 13a, `corr_3y rel_1w rel_1m rel_6m rel_ytd rel_3y_ann`; per group `beat_ref_3m ew_ratio ew_tells`, from slice 13a `med_z_13w` and the reference-dependent entries of `index_metrics`, from slice 13c `ew_dir ew_days ew_rel_3m`; the per-benchmark parts of `rotation` and the weekly `rel_13w`, `z_13w` arrays of the series files. `sharpe_3y` depends on cash only and is a plain number.

## 5. Benchmark-relative and cash-relative metrics (slice 4, extended later)

Let `r` be an ETF weekly return, `m` the weekly return of `REF` (each configured benchmark in turn), `c` the weekly cash return. All `REF` rows are computed once per benchmark.

| id | Definition | Window | Freq | Min obs | Null when | Tab / slice |
| --- | --- | --- | --- | --- | --- | --- |
| `rel_3m`, `rel_1y` | `P / back(k) - REF_{t_end} / REF_s`, the ETF return minus the reference return (decimal fractions). `s` = the ETF's anchor date, `REF` looked up on that exact date | 3 / 12 months | daily | both anchors | any leg null or `REF` missing on `s` (`reference_missing`) | Market, Heatmap, Groups (S4, S5) |
| `excess_cash` | `x = r - c` weekly | per week | weekly | n/a | `r` or `c` null | feeds Sharpe |
| `sharpe_3y` | `mean(x) / sd(r) * sqrt(52)`; independent of the benchmark | 156 weeks | weekly | 104 finite pairs of `r`, `c` | no `[cash]` (`cash_missing`); any null among the window slots (`cash_incomplete`); `sd <= 1e-12` (`zero_variance`); fewer than 104 | Risk, Screener (S4, S8) |
| `beta_3y` | `cov(r, m) / var(m)` on pairs where both are finite | 156 weeks | weekly | 104 pairs | fewer than 104 pairs; `var(m) = 0` | Market, Risk (S4, S8) |
| `up_cap` | `mean(r where m > 0) / mean(m where m > 0)` | 156 weeks | weekly | 104 pairs and a non-empty regime | empty regime | Risk |
| `down_cap` | same with `m < 0` | 156 weeks | weekly | as above | empty regime | Risk |

Rules:
- `m` is `REF` weekly return, masked wherever the ETF's `r` is null.
- Null reasons are the DATA.md section 9 codes: `reference_missing` (also: no benchmark configured), `cash_missing`, `cash_incomplete`, `insufficient_pairs` (fewer pairs than the minimum), `zero_variance`.
- Excluded and broken-price funds (review marks) are skipped by every aggregate that uses these metrics.

As built (slice 4):
- `rel_*` read the benchmark level at `t_end` and at the anchor `s` with the alignment carry of section 1 (at most `CARRY_DAYS`, never past the benchmark's last level); beyond that the fund is `reference_missing`. The benchmark level exists only on dates where every component has a close.
- `sharpe_3y` and `beta_3y` use the fund's own last 156 weekly slots (`REF_WEEKS`). A cash week that is null in a slot where the fund has a return gives `cash_incomplete` (slots where the fund has no return do not need cash); the 104-pair minimum then applies to the finite fund returns (`insufficient_pairs`). Order of the reasons: `cash_missing`, `cash_incomplete`, `insufficient_pairs`, `zero_variance`.
- The blend chain starts at the first date on which every component has a close (level 100) and is rebalanced on the last common date of each W-FRI bucket; a single component is therefore its closes rebased to 100 at its first close.
- The cash carry stops at the last observation: a Friday after it has no weekly cash return, so a fund with a return in that week is `cash_incomplete`.

As built (slice 8):
- `up_cap` and `down_cap` use the same masked pairs as `beta_3y` (the fund's last 156 weekly slots, both returns finite). The regime is `m > 0` (up) or `m < 0` (down); a week with `m = 0` is in neither. An empty regime, or fewer than 104 pairs, is null with `insufficient_pairs` (there is no separate reason code for an empty regime); a benchmark without levels is `reference_missing`.

## 6. Group aggregates

A group is the `group` column of `universe.csv`. A family is a separate, coarser label used for filters only.

| id | Definition | Min obs | Null when | Tab / slice |
| --- | --- | --- | --- | --- |
| `group_return` | For each of the six standard return periods (stored as `group_return[period id]`; nine ids from slice 13a, adding `r_1w r_5y_ann r_10y_ann`): the return of the group's `group_index` over that period, anchored as in 3.1 with `t_end = C` (the coverage cutoff), so `r_3y_ann` is the annualised index return. `r_ytd` is null when `as_of` falls in a later calendar year than `C`. One step of the index chain (below) is the mean of the members' returns on that step; members without a return are skipped, not zeroed. | 1 member | the index has no level at `C` or at the anchor | Overview, Heatmap, Groups (S5, S6) |
| `group_index` | Level 100 at the first date with any member return; level `*= (1 + group_return)` for each step. If a period has no valid group return but later members reappear, resume with the mean of `L_t / L_d` over members finite on both dates, where `d` is the last date the group had a valid level. If none qualify, factor is 1 and a chain-break counter is incremented. | 1 member | no member has data | Groups (S6) |
| `med_<metric>` | Median of member values (finite only) for `r_1m`, `r_3m`, `r_1y`, `vol_1y`, `maxdd_1y`, `z_13w` (`med_z_13w` published per benchmark from slice 13a) | 1 member | no finite member | Overview, Groups |
| Member values | Every member value entering `med_*`, `breadth_200d`, `beat_ref_3m` and `dispersion_1y` is the member's metric evaluated with `t_end` = its last close on or before `C` (section 1, Anchor); the published per-fund value stays at its own last close | n/a | n/a | all group aggregates |
| `breadth_200d` | Share (fraction) of finite `vs_200d` that are above 0 | 1 member | none finite | Overview (S5) |
| `beat_ref_3m` | Share (fraction) of finite `rel_3m` above 0, per benchmark | 1 member | none finite | Groups |
| `dispersion_1y` | Sample standard deviation of members' `r_1y` | 2 members | fewer than 2 | Groups |

As built (slice 6): `beat_ref_3m` reads `rel_3m[benchmark id]` of each member (a member whose entry is null is skipped, never counted as a loser); `med_vol_1y` and `med_maxdd_1y` are member medians of `vol_1y` and `maxdd_1y`; `dispersion_1y` uses `ddof=1`. The browser draws the Groups relative and drawdown charts itself from `group_index` and the benchmark series (relative = index level divided by the reference level on the same date, no filling; drawdown from the running peak of the full history).

As built (slice 5): the chain runs on the market axis (DATA 9 `market.dates`), one step per axis date, so `group_index.dates` is that axis. Level 100 at the first date on which any member has a close. At date `t` the factor is the mean of `W_t / W_d` over members with a close on both `t` and `d` (`d` = the previous date with a level; a member without a close on the previous axis date is skipped for that step, not zeroed); when no member qualifies the factor is 1 and the break counter grows. A date on which no member has a close has a `null` level and `d` stays. `group_return` over a period is the return of `group_index` ending at the cutoff `C` (section 1, Anchor), never a mean of the members' own-date returns; the index itself is cut at `C`. Every table that shows a group count shows `n of N`: members with a value for the shown metric, of all members; a group with no priced member is still listed, with dashes. `breadth_200d`, `med_*` and `group_return` take their members from the member filter of DATA 8. `breadth_200d` in the Heatmap table is coloured around 50%, not around zero.

Scope rules:
- Group visibility is a shared setting. Group-level values (medians, `breadth_200d`, group indices) are recomputed for the visible groups. Counts of finite members (for hover) come from the browser.
- Aggregates fitted to the full universe are **never** presented as recalculated for a filtered selection. These are market breadth, participation, implied correlation, rolling PCA, scree and MDS maps, and fixed loadings. Under a filter they show an "Unavailable for a filtered selection" panel; nothing is recomputed.

## 7. Structure, correlation and rotation (slices 7 to 9)

### 7.1 Correlation (slice 9)

Both correlation windows and the PCA windows (7.2) end at the coverage cutoff (section 10).

| id | Definition | Window | Freq | Min obs | Null when |
| --- | --- | --- | --- | --- | --- |
| `corr_1y_daily` | Pearson of daily returns, pairwise-complete | last 252 daily rows | daily | 200 pairs | fewer than 200 pairs; zero variance |
| `corr_3y_weekly` | Pearson of weekly returns, pairwise-complete | last 156 weekly rows | weekly | 104 pairs | fewer than 104 pairs |

Display order of a matrix is hierarchical clustering only: `D = clip(1 - R, 0, 2)` with a zero diagonal, average linkage, leaf order. Missing pairs are blank (never 0). For ordering only, a missing pair uses `D = 1` (as if uncorrelated). Average linkage is implemented in numpy (no scipy). Leaf order is deterministic: of two merged branches, the one holding the lower smallest member index comes first; one order per window serves every view. Order does not change any value. Pair lists (15 highest, 15 lowest) break ties by ticker.

As built (slice 9): only member funds with at least the minimum number of finite returns in the window enter a matrix (`n`; fewer than `PCA_MIN_FUNDS` gives `insufficient_data` and no matrix); a pair needs its own minimum of overlapping returns, otherwise it is blank (`null`). Daily rows are the dates up to the cutoff on which at least `MARKET_MIN_COVERAGE` of the funds have a close (the market-axis rule of section 11, restricted to the cutoff); a fund's daily return needs its own previous close at most 7 days earlier, so a fund missing a row has no return there. Weekly rows are the W-FRI returns of section 1 up to the last Friday on or before the cutoff. Zero variance on a pair is blank. The leaf order is a plain O(n^3) numpy average-linkage over the included funds; the closest pair is the first in row-major order on ties. Under a group filter the browser takes the sub-matrix of the visible funds and keeps this order.

### 7.2 PCA (slice 9)

| id | Definition | Window | Min obs | Null / status |
| --- | --- | --- | --- | --- |
| `pca_fixed` | Eigen-decomposition of the correlation matrix of weekly returns for complete-case ETFs. Complete-case means all 104 of the last 104 weekly returns are finite and the standard deviation exceeds 1e-8. Share `i` = `lambda_i / sum(lambda)` (a fraction). Loadings `v_i * sqrt(max(lambda_i, 0))`. | last 104 weeks (2Y) | 8 complete ETFs | fewer than 8: status `insufficient data`, no shares, no loadings. |
| `pca_status` | `ok`; `insufficient_data`; `zero_dispersion` if `sum(lambda) <= 0`; `indefinite_correlation` if the smallest eigenvalue is below `-1e-10 * max(1, max abs(lambda))`. Non-ok statuses publish no repaired numbers. | n/a | n/a | n/a |
| `pca_orient` | PC1 sign chosen so its mean loading is positive. If that mean is exactly 0, use the sign of the largest-magnitude element. | n/a | n/a | n/a |
| `pca_rolling` | Same eigen-decomposition on each 52-week window; report PC1 to PC3 shares. | 52 weeks, step 1 week | 8 complete varying ETFs | fewer than 8 in a window: that week is a null row (the line breaks). |
| `pca_n` | Number of ETFs in the sample, and the count excluded because of missing data. Always displayed with the shares. | n/a | n/a | n/a |

All PCA outputs are full-universe aggregates (section 6). Under a group filter they are not recomputed.

As built (slice 9): the sample is the member funds (DATA 8) whose last 104 weekly returns (W-FRI up to the last Friday on or before the cutoff) are all finite with a standard deviation above `PCA_STD_MIN`; fewer than 104 weekly rows means no fund is complete. `pca_n` is `{used, excluded}`. PC2 and PC3 signs are not specified by the plan; the largest-magnitude loading is made positive so the output is reproducible. `pca_rolling` covers every weekly return date up to the cutoff; each window is complete-case on its own 52 rows, and for speed (equal values) uses the 52 x 52 Gram matrix of the standardised returns when a window has more funds than weeks. Constants: `PCA_ROLL_WEEKS`, `PCA_COMPONENTS`, `PCA_STD_MIN`, `CORR_DECIMALS` (DATA 12).

### 7.3 Rotation (slice 7)

Weekly data to the coverage cutoff (section 10).

| id | Definition | Min obs | Null when |
| --- | --- | --- | --- |
| `rs` | `W_id / W_REF * 100`, weekly levels, per benchmark | both finite | either null |
| `rrg_ratio` | `100 * EMA10(rs) / EMA40(rs)`, EMA with span `s` and `adjust = false` | 60 weekly levels | fewer than 60 finite levels; `rs` null that week |
| `rrg_mom` | `100 * rrg_ratio / rrg_ratio(4 weeks earlier)` | 4 prior weeks | prior null |
| `rrg_quadrant` | `ratio >= 100`: Leading if `mom >= 100`, else Weakening. `ratio < 100`: Improving if `mom >= 100`, else Lagging. | n/a | ratio or momentum null |

Show the last 156 weekly points for groups and 52 for ETFs. Axes are symmetric around 100.

As built (slice 7): weekly levels are the last close of each W-FRI bucket, null for a week without a close; a bucket labelled after the cutoff is dropped, so the last week is the last Friday on or before the cutoff. `rs` of a group uses its `group_index` level. The EMAs run over the weekly grid (`ewm(adjust=False)`, a missing week keeps its slot); `rrg_ratio` is null unless the week's `rs` is finite and at least 60 finite levels exist up to that week (so the replay is causal). Funds are the member funds only (DATA 8), and a fund is published only when it has a ratio in its 52-week tail. A group with no ratio in the whole 156-week tail is omitted (listed once in the footnote).

### 7.4 Leadership (slice 7)

| id | Definition | Min obs | Null when |
| --- | --- | --- | --- |
| `lead_x` | `1 + from_52wl`, "gain factor from 52-week low" | both extremes | `from_52wl` null |
| `lead_y` | `from_52wh`, "percent below 52-week high" | extremes | null |
| `rank_trail` | At each month end, rank 1 = highest trailing return over the chosen horizon (anchor rules of 3.1); ties by display order | one finite return | that month is skipped in the line |
| `ew_ratio` | `group_index / REF`, daily, per benchmark | 200 observations | missing leg |
| `ew_tells` | Count of six tests on the last date (a missing input makes a test false): ratio below its 200-day mean; 50-day mean below 200-day mean; 200-day mean below its value 20 positions earlier; ratio below its value 63 and 21 rows before the last row (two tests; `EW_BACK_LONG`, `EW_BACK_SHORT`); group level below its 200-day mean. Warn at 4 or more. | 200 observations | 200-day mean or its prior value null |

As built (slice 7): `from_52wh` / `from_52wl` use the closes dated on or after `t_end - 12 months`. `ew_ratio` is the last value of `group_index / reference level` on the dates where both exist; the six tests read observation rows of that ratio (the sixth reads the group index itself), a missing input counts as false, and `ew_tells` needs 200 + 20 rows (`ew_ratio` 200). The tests use all dates to the last common one, not the cutoff. `rank_trail` has one point per calendar month that has a date on or before the cutoff, so its last point is the last date before the cutoff, and only the 3-month horizon is published in `ret` and `rank`; slice 13c adds every return period in `rank_trail.by_period` (PARITY 13c.9). `lead_x`, `lead_y` are computed in the browser from the two fund fields.

### 7.5 Calendar (slice 7)

| id | Definition | Null when |
| --- | --- | --- |
| `monthly_return` | Last close in each calendar month, then simple return against the previous month's last close. A month with no close is null and the next month is also null (no bridging). | month without a close |
| `year_return` | `product(1 + monthly_return_m) - 1` over the months with data. A year with fewer than 12 months is marked with an asterisk and the count ("n of 12"). | no month with data |
| `season_mean`, `season_median` | Mean and median of finite `monthly_return` for one calendar month across shown years. | no finite value |
| `season_hit` | Share (fraction) of finite values above 0. | no finite value |

Part years are excluded from seasonal statistics.

## 8. Signal lab (slice 10)

Weekly data. Canonical signal: 13-week excess return against `REF`, z-scored in a rolling 156-week window.

| id | Definition | Window | Min obs | Null when |
| --- | --- | --- | --- | --- |
| `rel_13w` | `product of (1 + r_i) over 13 weeks - product of (1 + m_i) over 13 weeks`, a difference of decimal returns | 13 weeks | all 13 returns of both series | any of the 26 returns null |
| `z_13w` | `(rel - mean(rel)) / sd(rel)` over the trailing 156 values of `rel`, current included, sample SD | 156 weeks | 104 finite `rel` values and a finite current value | fewer than 104; `sd <= 0`; current null |
| `pct_13w` | Share of finite `rel` values in the trailing 156 window that are less than or equal to the current value (a fraction; ties inclusive, current included) | 156 weeks | 52 finite values and a finite current | fewer than 52; current null |
| `z_rank` | Share of finite `z` in the shown history that are less than or equal to the current z | shown range | one finite value | current null |

Custom horizon `h` or window `win` (Signal lab controls): the relative signal uses `h` weeks. The z minimum becomes `max(26, round(0.67 * win))` when `win` is not 156. An expanding window uses all history up to the current week, with the canonical minimum of 104. From slice 13h the percentile minimum is `SIGNAL_PCT_MIN` (52) for the canonical setting only; any custom horizon, window or comparison uses that setting's z minimum, an expanding window 104; and the comparison may be the reference, the own group index or nothing (PARITY 13h.2, 13h.6). The latest-value summary is always taken at the final date, even when null.

As built (slice 10):
- Grid: the fund's own weekly slots from `weekly_arrays` (METRICS 9 as built); the final slot is its last close, so a fund is evaluated at its last complete week (section 1, Partial week); the group signals and the cross-section of groups end at the cutoff `C`. The reference weekly level of a slot is the last level of the same W-FRI bucket; a fund ending after the reference's last level has `reference_missing` in its final week.
- `rel` is computed from log weekly returns (`expm1` of the 13-week sum), which equals the product form and is null unless all 26 returns are finite. The trailing window holds up to 156 slots; slots with no `rel` stay out of the count and out of the mean and SD. `sd` at or below `SIGNAL_SD_MIN` (1e-12) is `zero_variance`.
- `z_rank` over the full history is published per fund; the browser recomputes it over the shown range (METRICS says "shown history", DATA publishes one number per fund).
- Groups have no published signal: the browser computes it from `group_index` (daily, last non-null level per W-FRI bucket) and the benchmark series file with the same functions as above (`ARCADE.lab`, checked against the published fund arrays: maximum difference 5e-5, the 4-decimal rounding).
- Custom horizon or window: the Lab recomputes for the chosen series only. The cross-section of every fund always shows the published canonical values, because a custom signal for every fund would need every series file.

## 9. Rolling windows (slices 3 and 6)

All rolling statistics use weekly returns and a window independent of chart history. Compute on full history first, then slice for display.

| id | Definition | Window | Min obs | Null when |
| --- | --- | --- | --- | --- |
| `roll_ret` | `W_k / W_{k-52} - 1` | 52 weeks | both levels finite | either null |
| `drawdown_t` | `L_t / max(L_s, s <= t) - 1`, peak taken over the full history before the shown range, so a shortened chart does not reset the peak | full | n/a | level null (the line breaks) |

As built (slice 6): the weekly level is the last close of each `W-FRI` week; `k - 52` counts slots of the fund's own weekly grid from its first week to its last (a week without a close is a null slot, so a gap before the window nulls `roll_ret`, and `drawdown_t` stays null on the empty week while the peak carries across it). Gaps break lines. A rolling value at a week is never filled from the week before. The Fund tab computes its daily drawdown in the browser from `adj_close` by the same rule.

## 10. Data-quality rules (slice 11)

`quality.screen` applies these rules before `metrics.py` runs (DATA.md section 11). They affect analytics only; stored prices are never edited.

| Rule | Definition |
| --- | --- |
| Spike quarantine | For interior point `i`: candidate if `abs(v_i / v_{i-1} - 1) > SPIKE_MOVE` and `abs(v_{i+1} / v_{i-1} - 1) <= SPIKE_REVERT`. A flagged point is excluded from analytics (treated as missing, not interpolated). Consecutive points are not flagged twice (`q_i = candidate_i and not q_{i-1}`). |
| Gap break | If two consecutive closes are more than `GAP_BREAK_DAYS` calendar days apart, analytics use only the segment after the last such gap. The earlier segment is not spliced on. `first_date` shows the segment start and the Data tab lists the ETF. |
| Coverage cutoff | Defined and computed once by the slice-11 screen (`quality.screen`, DATA.md section 11) and passed to `metrics.py` by `build`; `metrics.py` never imports `quality.py`. Every group-level and cross-sectional output ends at the cutoff date `C`: `group_index` and everything read from it (`group_return`, early-warning tells, rotation, rank trail, group signals), the Market statistics, correlation matrices and PCA. `C` is the last date with any ETF observation, minus `CUTOFF_LAG_BDAYS` business days, then moved back to the last observed date on or before it. Per-ETF figures keep their own last close. |
| Staleness | `days_stale = as_of - last_date`. Flag at more than `STALE_DAYS` (14, DATA.md section 12) calendar days. A stale ETF is shown with its own date and is never rolled forward. |
| Publication safety | `build` writes `data.js` to a temporary file and replaces the live file only after validation. On any error the previous `data.js` stays and the exit code is 2. |

## 11. Market statistics (slice 4)

Full-universe aggregates (section 6 scope rule). Shares (fractions) exclude ETFs without a full window from numerator and denominator.

| id | Definition | Window |
| --- | --- | --- |
| `above_50d`, `above_200d` | Share of ETFs with close above their own 50 / 200 observation mean | 50 / 200 |
| `new_hi`, `new_lo` | Share at or above the rolling 252-row maximum / at or below the minimum | 252 |
| `dispersion_21d` | Cross-sectional sample SD of 21-row returns, a decimal fraction (needs 2 ETFs) | 21 rows |
| `impl_corr_63d` | `rho = (V - D) / ((sum s_j)^2 - D)`: `s_j` sample SD of ETF `j` over 63 daily returns, `D = sum(s_j^2)`, `V` sample variance of the per-date cross-sectional sum. Admit ETFs with 50 finite returns on 50 common dates, at least 2 varying. | 63 rows |

As built (slice 4): all four windows count each fund's own observation rows; a statistic on date `d` uses the funds that have a close on `d`. The dates are the market axis of DATA.md section 9 (`MARKET_MIN_COVERAGE`), so thin dates (month ends, one-market holidays) do not produce a share of a handful of funds. For `impl_corr_63d` the 63 returns are between consecutive axis dates (null where either close is missing, never filled); the "common dates" are the `MARKET_CORR_MIN` (50) dates of the window with the most finite returns across funds (ties: most recent), admitted funds are those finite on all of them, and at least two must vary. Requiring 50 common dates among all admitted funds literally would leave 30 on a real shelf with differing holidays.

`impl_corr_63d` is null if fewer than two ETFs qualify, the denominator is at most `1e-12 * D`, or `abs(rho) > 1 + 1e-10`; otherwise clip to [-1, 1].

## 12. Review marks (slice 12)

Review marks (DATA.md section 14) change inputs and member lists, not formulas.

| Rule | Definition |
| --- | --- |
| Broken range | Closes in `[from, to]` are removed before the quality screen (slice 11), never filled. Anchors follow section 3.1 (null `no_history_to_anchor` if the hole swallows the anchor); an empty week nulls two weekly returns (not bridged) and `vol_1y` keeps its 40-return minimum. Drawdown peaks (`dd_now`, `maxdd`, `maxdd_1y`, `drawdown_t`) use remaining closes only. A hole over `GAP_BREAK_DAYS` is a gap break (section 10). |
| Whole series | No closes: every metric null, reason `review_broken_price`; treated like `no_data` in aggregates, correlation and PCA. |
| Excluded fund | Left out of the members behind `group_return`, `group_index`, `med_*`, `breadth_*`, `beat_ref_3m`, `dispersion_1y`, market statistics (section 11), correlation, PCA and rotation. Its own metrics are still computed. A group left empty keeps its row, aggregates null. |
| Moved fund | Counted in the target group only. |
| Cleared marks | A `clear` mark removes earlier marks of its ticker before any of the rules above apply, so a cleared fund is computed as if the cleared marks never existed. |
| Unknown ticker | A mark for a ticker that is not in the universe is ignored (a warning, not an error). |
| Standing rule for aggregates | Every aggregate built in slices 4 to 9 takes its members through the single member filter of DATA.md section 8, so `review_excluded`, `review_broken` and `no_data` funds never enter it, whichever slice builds the aggregate. |

## 13. Parity metrics (slice 13)

Defined in `docs/reference/PARITY.md`, item by item, with window, minimum, null rule and evaluation date. Ids:

| Sub-slice | Ids |
| --- | --- |
| 13a | `r_1w r_5y_ann r_10y_ann years vol_3y sortino_3y hit_1y corr_3y rel_1w rel_1m rel_6m rel_ytd rel_3y_ann`; `index_metrics` (per-ETF ids on `group_index` at `C`); `benchmarks[].metrics`; `med_z_13w`; `cash_w`; `avg_corr within across corr_ref`; browser `roll_ex roll_vol roll_beta roll_corr roll_sharpe` |
| 13e | `resid` (intermediate); `alpha_3y alpha_t_3y beta_equity_3y beta_bond_3y res_vol_3y appraisal_3y alpha_full alpha_t_full`; group `factor_fit`; `pca_resid rpc1 rpc2 ar5 pc1_resid`; `mds_x mds_y cluster`; `corr_3y_residual`; group correlation `corr_3y_weekly corr_1y_daily` |
| 13c | `ew_dir ew_days ew_rel_3m`; `ew_index groups_above_200d`; `rank_trail` per period |
| 13d | browser `rel_ref_t rel_grp_t pct13_t g_rel_h g_z_h` |
| 13g | `pos_21d`; browser as-of counts and market brief |
| 13h | browser `rel_h_cmp` |
