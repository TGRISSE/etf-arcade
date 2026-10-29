# Known pitfalls

The 20 mistakes that cost the most time on earlier builds of this kind of dashboard, most painful first. Read this before slice 1 and again before each slice.

1. **Build and browser disagree on a number:** compute each metric once in Python; if the browser recomputes it, add a test that both give the same value on the same input.
2. **Copied constants drift:** every window, minimum and threshold lives once in `constants.py`, and the browser reads it from `data.js`.
3. **Missing becomes zero:** a missing price, cash rate or correlation gives null with a reason, and every share divides by the eligible count only.
4. **No rule for the last, incomplete week:** drop a week whose Friday is after the last close, in every weekly statistic (METRICS section 1).
5. **Mixed dates in one cross-section:** every group-level and cross-sectional figure ends at the one coverage cutoff, and the header shows that date.
6. **Carry missing or too generous:** carry a level at most 7 days, once, never past the series' last close, and never form a return across a carried value.
7. **Indicators run on after a fund stops pricing:** cut every derived series at the fund's real last close.
8. **Rebasing before the data starts:** rebase at the first valid observation inside the shown range, never at an empty date.
9. **PCA on mismatched samples:** use one complete-case sample and fix every component's sign with a written rule.
10. **Edge cases give infinity or crashes:** a zero denominator is null, floats are compared against a tolerance, and one bad fund never aborts the build.
11. **Filtered views show whole-universe numbers:** recompute for the visible groups or say "unavailable", never show a stale full-universe figure.
12. **State lost on navigation:** benchmark, history and tab switches keep fund, filters, sort and custom metrics; keep view state in `tabState`, not on DOM nodes.
13. **Labels overlap or clip:** measure text, cap the count, drop colliding labels, and run the `check=1` self-check (UI.md section 8) on every tab.
14. **Colours and captions don't match the view:** every group-coloured chart has a legend, a card shows the metric it is sorted by, and captions come from the current settings.
15. **Stale data after a rebuild:** series files carry the `generated` stamp and the page reloads on a mismatch.
16. **A failed build replaces the good one:** write to a temporary file, validate, then swap; on error keep the old `data.js` and exit 2.
17. **One bad print distorts everything:** apply the spike rule from slice 11 with its test, flag rather than delete, and let review marks override.
18. **Windows and headless tools misbehave:** use absolute screenshot paths, UTF-8 output, check 390 px inside an iframe, and put timeouts on browser runs.
19. **Docs drift from the code:** update STATUS.md and the "as built" notes in the same change as the code.
20. **Destructive commands on real data:** never run `sample --force` on a project with real prices, and keep network fetching opt-in.
