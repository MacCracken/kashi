# kashi — Benchmarks

> **Baseline captured**: 2026-05-27 (0.1.0). Raw data:
> [`benchmarks/history.csv`](benchmarks/history.csv) — one row per
> `(version, benchmark)`, appended each release for version-over-version
> comparison. This file is the human-readable view; the CSV is the source
> of truth.

## What's measured

The freestanding core is the hot path of any framebuffer console:
`kashi_glyph_row` runs once per pixel-row per glyph during text render, so
its cost is the per-character render floor. The suite (`tests/kashi.bcyr`)
measures the accessors in isolation:

| Benchmark | What it measures |
|---|---|
| `glyph_row` | One `kashi_glyph_row(font, ch, row)` — the per-pixel-row accessor (a few branches + one `load8`). |
| `glyph_ptr` | One `kashi_glyph_ptr(font, ch)` — base-pointer fetch (branches + pointer arithmetic, no load). |
| `scan_vga_8x16` | A full 96-glyph × 16-row sweep of the VGA font — a full-screen-repaint proxy. |

## How to run

```sh
CYRIUS_DCE=1 cyrius bench tests/kashi.bcyr
```

`CYRIUS_DCE=1` matches the release/CI build (dead-code-eliminated). Run a
few times and take the steady-state figure; sub-microsecond ops use a
≥1e6-iteration batch to amortize `clock_gettime` overhead (see the
`tests/kashi.bcyr` header).

## Baseline — 0.1.0 (2026-05-27)

Host: AMD Ryzen 7 5800H, x86_64 Linux. Indicative single-host numbers, not
a cross-machine guarantee.

| Benchmark | avg | iters |
|---|---|---|
| `glyph_row` | 17 ns | 1,000,000 |
| `glyph_ptr` | 7 ns | 1,000,000 |
| `scan_vga_8x16` | 27 µs | 10,000 |

`scan_vga_8x16` at 27 µs over 1,536 row reads ≈ 17.6 ns/row, consistent
with the standalone `glyph_row` figure — the whole-font sweep carries no
per-glyph overhead beyond the accessor itself.

### Unified-dispatcher overhead (M1, unreleased)

The M1 `kashi_font_row` dispatcher routes built-in vs runtime ids. Measured
on the same host (not yet a release row in the CSV):

| Benchmark | avg | iters | vs direct |
|---|---|---|---|
| `font_row_builtin` | 20 ns | 1,000,000 | +~2 ns over `glyph_row` (one compare + branch) |
| `font_row_runtime` | 28 ns | 1,000,000 | +~10 ns (registry lookup + bounds + load) |

## Updating

Each release (or whenever an accessor's cost changes), append rows to
`benchmarks/history.csv` for the new version and refresh the table above.
`scan_vga_8x16` is recorded in ns in the CSV (27 µs → 27000) so all columns
share a unit; the table renders the natural unit.

## Notes

- The 0.1.0→post-audit hardening (audit 2026-05-27: `kashi_fset16/fset8`
  bounds guards, the `kashi_font_is_ready` reader) touches only the
  one-time init path and a new query, **not** the hot accessors — the
  baseline above is unchanged by it.
