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
| `scan_vga_8x16` | A full 224-glyph × 16-row sweep of the VGA font — a full-screen-repaint proxy. (Was 96 glyphs pre-0.4.0; see workload note below.) |
| `font_row_builtin` | `kashi_font_row` against a built-in id (the unified dispatcher's fast path). |
| `font_row_runtime` | `kashi_font_row` against a runtime-loaded font without a Unicode map (identity-index fallback). |
| `font_row_runtime_cp` | `kashi_font_row` against a runtime font WITH a Unicode map — binary search through the codepoint→glyph table. |

## How to run

```sh
CYRIUS_DCE=1 cyrius bench tests/kashi.bcyr
```

`CYRIUS_DCE=1` matches the release/CI build (dead-code-eliminated). Run a
few times and take the steady-state figure; sub-microsecond ops use a
≥1e6-iteration batch to amortize `clock_gettime` overhead (see the
`tests/kashi.bcyr` header).

## Current — 0.8.0 (2026-05-28)

Host: AMD Ryzen 7 5800H, x86_64 Linux. Indicative single-host numbers, not
a cross-machine guarantee.

| Benchmark | avg | iters |
|---|---|---|
| `glyph_row` | 18 ns | 1,000,000 |
| `glyph_ptr` | 7 ns | 1,000,000 |
| `scan_vga_8x16` | 63 µs | 10,000 |
| `font_row_builtin` | 20 ns | 1,000,000 |
| `font_row_runtime` | 43 ns | 1,000,000 |
| `font_row_runtime_cp` | 65 ns | 1,000,000 |

`scan_vga_8x16` at 63 µs over 3,584 row reads (224 glyphs × 16 rows)
≈ 17.6 ns/row, consistent with the standalone `glyph_row` figure — the
whole-font sweep carries no per-glyph overhead beyond the accessor itself.

## 0.x trend

Numbers from `benchmarks/history.csv`. All times in ns. Same host
(AMD Ryzen 7 5800H, x86_64 Linux). Empty cells indicate the benchmark
didn't exist yet at that version. Variation within ±2 ns is system
noise — only structural shifts are called out below.

| Version | `glyph_row` | `glyph_ptr` | `scan_vga_8x16` | `font_row_builtin` | `font_row_runtime` | `font_row_runtime_cp` |
|---|---|---|---|---|---|---|
| **0.1.0** | 17 | 7 | 27,000 | — | — | — |
| **0.2.0** | 18 | 7 | 27,000 | 19 | 28 | — |
| **0.3.0** | 18 | 6 | 26,000 | 19 | 43 | 61 |
| **0.4.0** | 16 | 6 | 60,000 | 19 | 42 | 61 |
| **0.5.0** | 17 | 6 | 61,000 | 19 | 42 | 63 |
| **0.5.1** | 18 | 7 | 62,000 | 19 | 42 | 64 |
| **0.5.2** | 17 | 6 | 62,000 | 19 | 43 | 61 |
| **0.6.0** | 18 | 7 | 62,000 | 19 | 43 | 62 |
| **0.8.0** | 18 | 7 | 63,000 | 20 | 43 | 65 |

> The 0.7.x cuts (BDF, PCF, sidecar tables) added parsers / load paths
> but **did not touch the hot accessors**, so they aren't separately
> measured — figures would be identical-within-noise to 0.6.0.
> Similarly the 0.8.0 audit fixes are all on parse-time error paths;
> the small rise vs 0.6.0 is sample variance, not a regression.

## Structural shifts called out

These are the only non-noise transitions in the table above:

- **0.2.0 → 0.3.0 (`font_row_runtime` 28 → 43 ns)**: M2's codepoint-
  addressing change (ADR 0003) added a resolve step. Before 0.3.0 the
  runtime accessor was a raw glyph-index lookup; after, it resolves
  the codepoint via the per-font map (or identity fallback if no map).
  Amortize with the `kashi_font_ptr`-once pattern in render hot paths.
- **0.3.0 → 0.4.0 (`scan_vga_8x16` 26,000 → 60,000)**: workload change,
  not a regression. The CP437 widening (ADR 0004) made the sweep
  iterate 224 glyphs instead of 96 — ≈ 2.3×, matching the
  glyph-count ratio. Per-row cost is unchanged.
- **0.4.0 → 0.5.0 (`glyph_row` 16 → 17 ns)**: the stride-aware
  accessor (`row * stride` vs `row * 1`) added 1 ns. For stride-1
  fonts (every built-in except `KASHI_FONT_VGA_9X16` in 0.5.1) the
  multiplication is by 1 — but Cyrius's optimizer doesn't currently
  collapse it. Acceptable; the new wide-glyph contract (ADR 0005) is
  more valuable than 1 ns/row.

Everything else is within ±2 ns / ±1,000 ns of the preceding row —
sample variance on a single host. The hot path has been **stable
within noise from 0.4.0 onward**.

## Hot-path render pattern

For a console rendering bytes at the per-row floor (kashi's intended
shape), resolve once, read rows:

```cyrius
var p = kashi_font_ptr(id, codepoint);    # one lookup
if (p != 0) {
    var row = 0;
    var h = kashi_font_height(id);
    while (row < h) {
        var bits = load8(p + row);         # 1 ns/row direct
        # ... blit bits ...
        row = row + 1;
    }
}
```

This skirts the `kashi_font_row` overhead per row and is what
agnos's framebuffer console does.

## Updating

Each release (or whenever an accessor's cost changes), append rows
to `benchmarks/history.csv` for the new version and refresh the
"Current" + "0.x trend" tables above. `scan_vga_8x16` is recorded
in ns in the CSV (27 µs → 27,000) so all columns share a unit; the
table renders the natural unit.

## Notes

- The audit fixes (`docs/audit/2026-05-27-audit.md`,
  `2026-05-28-audit.md`, `2026-05-28-audit-0.8.0.md`) touch only
  one-time init and error paths — never the hot accessors. The
  baseline is unchanged across them.
- `kashi_pcf_decode_glyph` (one-shot per glyph at load time) is not
  benched — it's amortized over the font lifetime, not per-render.
  Load-path numbers would be a worthwhile addition in a future
  release if a consumer reports load latency as a bottleneck.
