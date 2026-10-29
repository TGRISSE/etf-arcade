# Europe example

A European project for etf-arcade: 141 UCITS ETFs in 31 groups, one row per distinct ISIN in `data/universe_ucits.csv` whose match is not `none`. It is a mirror list of facts (names, tickers, listing currency), not market advice.

- `ticker` is the Yahoo ticker, `name` the UCITS name, `currency` the listing currency (EUR for most; a few USD, GBP, DKK, SEK). Closes stay in listing currency; there is no FX conversion.
- `group` and `family` come from the first US ETF in `data/universe.csv` that maps to the fund. Matches are `same_index` or `similar` in the source file; several US ETFs can share one UCITS ETF, and `similar` rows track a different index. Re-check a row before relying on it.
- `settings.toml` sets the title, the universe label (`UCITS ETFs`) and an ordered list of three benchmarks (MSCI World, Euro Govt Bond, a 60/40 blend of the two; the first is the default and the page lets the reader switch). `EUNL.DE` is also a universe row, so its series is shared; `EUNH.DE` is not, so `fetch` and `sample` handle it as an extra ticker (slice 4). No `[cash]` is set: Yahoo has no clean EUR yield series, so cash-dependent metrics are blank (reason `cash_missing`), never zero. There is no `theme.css`, so the default neon theme applies.

Once your coding agent has built slices 1 and 1b (see `docs/BUILD_SLICES.md`), this project runs with:

```
python -m etf_arcade --project examples/europe sample && python -m etf_arcade --project examples/europe build
```

Sample prices are SYNTHETIC. For real prices use `fetch` instead of `sample` (needs yfinance). Run `python -m etf_arcade sample --force && python -m etf_arcade build` afterwards to restore the default US project, because `web/data.js` holds one project at a time.
