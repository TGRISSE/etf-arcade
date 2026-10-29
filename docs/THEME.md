# Theme: neon arcade

Dark only. Every colour, font and spacing value lives in `web/theme.css` as a CSS custom property on `:root`. That file is the single source; nothing else hard-codes a colour.

`web/theme.css` ships with this repository and is not rebuilt: it holds the tokens below, the base elements and the component classes slice 1 needs (listed in `docs/reference/UI.md` section 16). Slices never edit it (a plan update may replace it): from slice 2 on, every new component class goes in `web/app.css` (created by slice 2, linked after `theme.css` and before `project.css`, tokens only). A project may add `<project>/theme.css` that overrides token values only (same names); `build` copies it to `web/project.css`, which `index.html` links last.

## Tokens

| Token | Value | Use |
| --- | --- | --- |
| `--bg` | `#0B0221` | page background |
| `--panel` | `#140A33` | panels |
| `--raised` | `#1D0F47` | table headers, row hover |
| `--input` | `#1A0D40` | form controls |
| `--border` | `#3A1C78` | panel and table borders |
| `--text` | `#F2F0FF` | body text |
| `--muted` | `#A89BD9` | secondary text |
| `--subtle` | `#7E6FB8` | decorative only (contrast 4.3:1, below text level) |
| `--accent` | `#FF2BD6` | active tab, titles, rules |
| `--accent-2` | `#00F0FF` | links, kickers, focus |
| `--positive` | `#B6FF00` | gains |
| `--negative` | `#FF3864` | losses |
| `--warning` | `#FFB000` | stale data |
| `--info` | `#3D7BFF` | neutral highlights |
| `--on-accent` | `#0B0221` | text on accent fills |
| `--chart-grid` / `--chart-axis` / `--chart-ref` | `#2A1760` / `#6A58B0` / `#C9C2F0` | gridlines, axis lines and ticks, reference lines |
| `--g1` to `--g12` | magenta, cyan, acid green, amber, violet, orange, mint, blue, red, yellow, pink, aqua | group palette, cycled by group order |

`--glow-accent` and `--glow-cyan` are the two glow recipes.

## Fonts

Loaded from Google Fonts (SIL OFL) with system fallbacks, so the page works offline.

- Display, `--font-display`: Orbitron. Titles, tab labels, KPI numbers.
- Body, `--font-body`: Chakra Petch. UI text.
- Mono, `--font-mono`: JetBrains Mono. Tables and numbers, with tabular figures.

## Glow rules

`text-shadow` and `box-shadow` glow is allowed only on titles, the active tab, focused or hovered controls, and a panel's hover border. Table and body text never glows. Contrast of `--text` and `--muted` on `--panel` must stay at or above 4.5:1. `--subtle` is decoration only, never essential text; the em dash for missing values uses `--muted`.

## No effects

No scanlines, no CRT curvature, no flicker, no animation. The only motion is a 150ms hover transition (`--transition`). It is disabled under `prefers-reduced-motion` (one rule at the end of `theme.css`).

## Chart colour rules

- Every chart colour comes from a token, read at runtime with `getComputedStyle(document.documentElement)`. No literal colours in chart code.
- Gridlines use `--chart-grid`, axis lines and tick marks `--chart-axis`, tick and axis label text `--muted`, reference or benchmark lines `--chart-ref`.
- Series that belong to a group use that group's `--gN` colour; positive and negative values use `--positive` and `--negative`.
- Never encode meaning by colour alone: keep hover text and labels, and make sure legend and axis labels fit inside their panel.

## Layout

Maximum width 1800px. One analytical panel per row, or at most two side by side; below 900px they stack. No horizontal page scroll: wide tables scroll inside their panel. Use short ETF names by default, with the full name and ticker in hover text.
