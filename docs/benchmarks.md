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
| `cp437_from_unicode_ascii` | `kashi_cp437_from_unicode(0x41)` — the Unicode → cell map's ASCII fast path (1.1.0, ADR 0011). |
| `cp437_from_unicode_high` | `kashi_cp437_from_unicode(0xE7)` — a CP437 high-half codepoint: binary search of the 128-entry reverse index. |
| `font_row_builtin_cp` | `kashi_font_row(VGA, 0xE7, row)` — the library face on a built-in with a non-ASCII codepoint: map + core row fetch, per call. |
| `font_init` | One `kashi_font_init()` — packing the three built-in tables, the 9×16 derivation and the Unicode index (1.1.0). Init-time work, but not a one-off everywhere: dhancha calls it on every bitmap-text draw. |

## How to run

```sh
CYRIUS_DCE=1 cyrius bench tests/kashi.bcyr
```

`CYRIUS_DCE=1` matches the release/CI build (dead-code-eliminated). Run a
few times and take the steady-state figure; sub-microsecond ops use a
≥1e6-iteration batch to amortize `clock_gettime` overhead (see the
`tests/kashi.bcyr` header).

## Current — 1.2.2 (2026-10-29, cyrius 6.7.18)

Host: AMD Ryzen 7 5800H, x86_64 Linux (load average ~0.3). Two binaries,
both built `CYRIUS_DCE=1` on 6.7.18: **1.2.1** (the tag, rebuilt in a
scratch copy) and **1.2.2**. Nine interleaved rounds; the table gives the
median of the per-op means. 1.2.2's one code change is a guard in
`kashi_pcf_cp_to_idx`, which no benchmark calls.

| Benchmark | 1.2.2 | 1.2.1 | Δ | iters |
|---|---|---|---|---|
| `glyph_row` | 17.7 ns | 17.8 ns | −0.6 % | 1,000,000 |
| `glyph_ptr` | 6.9 ns | 7.1 ns | −2.6 % | 1,000,000 |
| `scan_vga_8x16` | 59.0 µs | 59.0 µs | 0.0 % | 10,000 |
| `font_row_builtin` | 22.7 ns | 22.7 ns | −0.3 % | 1,000,000 |
| `font_row_runtime` | 44.4 ns | 45.2 ns | −1.8 % | 1,000,000 |
| `font_row_runtime_cp` | 60.0 ns | 59.3 ns | +1.1 % | 1,000,000 |
| `cp437_from_unicode_ascii` | 3.5 ns | 3.5 ns | 0.0 % | 1,000,000 |
| `cp437_from_unicode_high` | 25.0 ns | 25.1 ns | −0.4 % | 1,000,000 |
| `font_row_builtin_cp` | 40.9 ns | 40.7 ns | +0.6 % | 1,000,000 |
| `font_init` | 21.1 µs | 21.1 µs | +0.1 % | 10,000 |

**Flat**: mixed signs; the largest, `glyph_ptr` −0.2 ns, is in code that
did not change.

## Previous — 1.2.1 (2026-10-29, cyrius 6.7.18)

Host: AMD Ryzen 7 5800H, x86_64 Linux (load average ~1.0). Two binaries,
both built `CYRIUS_DCE=1` from the same source: **1.2.0 on 6.7.9** (the
previous release, rebuilt in a scratch copy) and **1.2.1 on 6.7.18**. Nine
interleaved rounds; the table gives the median of the per-op means. 1.2.1
changes only the toolchain, so this table isolates cyrius 6.7.9 → 6.7.18.

| Benchmark | 1.2.1 / 6.7.18 | 1.2.0 / 6.7.9 | Δ | iters |
|---|---|---|---|---|
| `glyph_row` | 17.8 ns | 17.8 ns | −0.4 % | 1,000,000 |
| `glyph_ptr` | 7.1 ns | 7.1 ns | +0.4 % | 1,000,000 |
| `scan_vga_8x16` | 59.3 µs | 59.9 µs | −0.9 % | 10,000 |
| `font_row_builtin` | 22.8 ns | 22.7 ns | +0.2 % | 1,000,000 |
| `font_row_runtime` | 44.7 ns | 45.0 ns | −0.6 % | 1,000,000 |
| `font_row_runtime_cp` | 59.9 ns | 59.9 ns | 0.0 % | 1,000,000 |
| `cp437_from_unicode_ascii` | 3.6 ns | 3.6 ns | +0.2 % | 1,000,000 |
| `cp437_from_unicode_high` | 25.2 ns | 25.3 ns | −0.5 % | 1,000,000 |
| `font_row_builtin_cp` | 41.0 ns | 41.3 ns | −0.8 % | 1,000,000 |
| `font_init` | 21.2 µs | 21.1 µs | +0.2 % | 10,000 |

**Flat**: every row is within ±0.9 %, with mixed signs. The binaries are
the same size as 6.7.9's but not byte-identical (codegen and stdlib
offsets moved), and none of that shows up in an accessor.

## Earlier — 1.2.0 (2026-10-29, cyrius 6.7.9)

Host: AMD Ryzen 7 5800H, x86_64 Linux (load average ~1.4 at the start).
Two binaries, both built `CYRIUS_DCE=1` from the same source: **1.1.1 on
6.6.20** (the previous release, rebuilt in a scratch copy) and **1.2.0 on
6.7.9**. Nine interleaved rounds; the table gives the median of the per-op
means. 1.2.0 changes only the toolchain, so this table isolates cyrius
6.6.20 → 6.7.9.

| Benchmark | 1.2.0 / 6.7.9 | 1.1.1 / 6.6.20 | Δ | iters |
|---|---|---|---|---|
| `glyph_row` | 17.8 ns | 17.9 ns | −0.6 % | 1,000,000 |
| `glyph_ptr` | 7.5 ns | 7.4 ns | +0.9 % | 1,000,000 |
| `scan_vga_8x16` | 61.8 µs | 61.9 µs | −0.2 % | 10,000 |
| `font_row_builtin` | 23.3 ns | 23.7 ns | −1.6 % | 1,000,000 |
| `font_row_runtime` | 46.1 ns | 46.5 ns | −0.9 % | 1,000,000 |
| `font_row_runtime_cp` | 61.3 ns | 61.3 ns | +0.1 % | 1,000,000 |
| `cp437_from_unicode_ascii` | 3.8 ns | 3.7 ns | +1.4 % | 1,000,000 |
| `cp437_from_unicode_high` | 26.1 ns | 25.8 ns | +1.3 % | 1,000,000 |
| `font_row_builtin_cp` | 42.2 ns | 42.2 ns | 0.0 % | 1,000,000 |
| `font_init` | 21.7 µs | 22.1 µs | −1.5 % | 10,000 |

**Flat**: every row is within ±1.6 %, and the signs are mixed, so the
toolchain moved nothing measurable. Absolute figures sit 1–3.5 % below the
2026-10-13 session on the same host, for both binaries alike. That is
session-to-session drift, which is why the old pin is re-measured in the
same session.

## Earlier — 1.1.1 (2026-10-13, cyrius 6.6.20)

Host: AMD Ryzen 7 5800H, x86_64 Linux (load not recorded; only a read-only
file search ran alongside). Three binaries, built `CYRIUS_DCE=1` and run
interleaved, nine rounds; medians of the per-op means. **1.1.0 on 6.6.6**
is this session's baseline; **1.1.0 on 6.6.20** isolates the toolchain;
**1.1.1 on 6.6.20** adds the storage fix.

| Benchmark | 1.1.1 / 6.6.20 | 1.1.0 / 6.6.20 | 1.1.0 / 6.6.6 | iters |
|---|---|---|---|---|
| `glyph_row` | 18.1 ns | 18.2 ns | 17.8 ns | 1,000,000 |
| `glyph_ptr` | 7.5 ns | 7.5 ns | 7.6 ns | 1,000,000 |
| `scan_vga_8x16` | 63.0 µs | 62.9 µs | 63.3 µs | 10,000 |
| `font_row_builtin` | 24.0 ns | 24.0 ns | 23.9 ns | 1,000,000 |
| `font_row_runtime` | 47.0 ns | 47.1 ns | 46.9 ns | 1,000,000 |
| `font_row_runtime_cp` | 63.2 ns | 62.8 ns | 63.1 ns | 1,000,000 |
| `cp437_from_unicode_ascii` | 3.8 ns | 3.8 ns | 3.8 ns | 1,000,000 |
| `cp437_from_unicode_high` | 26.7 ns | 26.6 ns | 26.6 ns | 1,000,000 |
| `font_row_builtin_cp` | 43.5 ns | 43.2 ns | 43.3 ns | 1,000,000 |
| `font_init` | 22.5 µs | 22.3 µs | 22.4 µs | 10,000 |

**Flat**: every row is within ±2 % across the three — neither the
fourteen toolchain releases nor the storage fix moved an accessor. The fix
changes only how much storage is reserved; every access is the same
`&buffer + offset` load or store, and the bytes used were already
contiguous at the front of each buffer. What it does move is size —
−88,576 B per binary (`docs/development/state.md`, *Build & size*).

## Earlier — 1.1.0 (2026-10-09, cyrius 6.6.6)

Host: AMD Ryzen 7 5800H, x86_64 Linux, quiet (load average < 1).
Indicative single-host numbers, not a cross-machine guarantee. Absolute
figures sat ~5–8 % above the 2026-09-21 record on the same host, so the
**1.0.10 binary was re-measured in the same session** (the same
`font_init` bench appended to its suite) — that column is the one to
compare against. Medians of 9 interleaved runs (`CYRIUS_DCE=1`); a second,
loaded-host session (load average 3–6) gave the same deltas.

| Benchmark | 1.1.0 | 1.0.10, same session | iters |
|---|---|---|---|
| `glyph_row` | 17.7 ns | 17.2 ns | 1,000,000 |
| `glyph_ptr` | 7.4 ns | 7.4 ns | 1,000,000 |
| `scan_vga_8x16` | 62.1 µs | 60.3 µs | 10,000 |
| `font_row_builtin` | 23.7 ns | 20.7 ns | 1,000,000 |
| `font_row_runtime` | 46.5 ns | 46.6 ns | 1,000,000 |
| `font_row_runtime_cp` | 62.3 ns | 62.2 ns | 1,000,000 |
| `cp437_from_unicode_ascii` | 3.9 ns | — | 1,000,000 |
| `cp437_from_unicode_high` | 26.3 ns | — | 1,000,000 |
| `font_row_builtin_cp` | 42.6 ns | — | 1,000,000 |
| `font_init` | 22.0 µs | 20.8 µs | 10,000 |

What moved, and why:

- **`font_row_builtin` +3 ns** — the library face now maps a built-in's
  codepoint through `kashi_cp437_from_unicode` (the 1.1.0 fix), and
  `kashi_font_row` resolves on every call, i.e. per row. ASCII takes the
  4 ns fast path; a high-half codepoint costs ~26 ns (`font_row_builtin_cp`
  = 43 ns). Resolve once per glyph with `kashi_font_ptr` (or
  `kashi_cp437_from_unicode` + `kashi_glyph_ptr`) and the mapping is paid
  once, not 16 times.
- **`font_init` +1.2 µs (+6 %)** — about 0.2 µs for the two extension
  glyphs in each font and their 9×16 derivation, about 1.0 µs for filling
  the 128-entry Unicode index (measured by building the same tree without
  the fill). The index source is listed already sorted by codepoint and
  written with straight stores — no sort runs at init (ADR 0011 records
  why: sorting at init doubled the function). A consumer that calls
  `kashi_font_init()` per draw (dhancha) pays the 1.2 µs per draw;
  guarding the call with `kashi_font_is_ready()` makes it once.
- **`scan_vga_8x16` +3 %** — code placement, not work. The scan's only
  source change is `kashi_glyph_encoded`'s upper-bound constant
  (`0xFF` → `0x101`); a 1.1.0 build with that constant put back at `0xFF`
  measured the same (63.0 µs against 63.2 µs, 1.0.10 61.4 µs, in a
  separate 7-run session), so it follows the moved code, not the compare.
- **`glyph_row`** 17.2 → 17.7 ns here and 18.1 → 18.0 ns in the loaded
  session: flat within noise. Its only source change is the same bound.

`scan_vga_8x16` at ~62 µs over 3,584 row reads (224 glyphs × 16 rows)
≈ 17 ns/row, consistent with the standalone `glyph_row` figure — the
whole-font sweep carries no per-glyph overhead beyond the accessor itself.

## Trend

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
| **1.0.0** | 17 | 7 | 64,000 | 19 | 43 | 66 |
| **1.0.9** | 16 | 7 | 56,193 | 19 | 43 | 58 |
| **1.0.10** | 17 | 7 | 55,900 | 19 | 43 | 58 |
| 1.0.10 (re-run 2026-10-09) | 17 | 7 | 60,277 | 21 | 47 | 62 |
| **1.1.0** (2026-10-09) | 18 | 7 | 62,147 | 24 | 46 | 62 |
| 1.1.0 (re-run 2026-10-13, 6.6.6) | 18 | 8 | 63,253 | 24 | 47 | 63 |
| 1.1.0 on 6.6.20 (2026-10-13) | 18 | 8 | 62,884 | 24 | 47 | 63 |
| **1.1.1** (2026-10-13, 6.6.20) | 18 | 8 | 62,962 | 24 | 47 | 63 |
| 1.1.1 on 6.6.20 (re-run 2026-10-29) | 18 | 7 | 61,881 | 24 | 46 | 61 |
| **1.2.0** (2026-10-29, 6.7.9) | 18 | 8 | 61,787 | 23 | 46 | 61 |
| 1.2.0 on 6.7.9 (re-run 2026-10-29) | 18 | 7 | 59,858 | 23 | 45 | 60 |
| **1.2.1** (2026-10-29, 6.7.18) | 18 | 7 | 59,314 | 23 | 45 | 60 |
| 1.2.1 (re-run 2026-10-29) | 18 | 7 | 59,015 | 23 | 45 | 59 |
| **1.2.2** (2026-10-29, 6.7.18) | 18 | 7 | 59,025 | 23 | 44 | 60 |

> The 0.7.x cuts (BDF, PCF, sidecar tables) added parsers / load paths
> but **did not touch the hot accessors**, so they aren't separately
> measured — figures would be identical-within-noise to 0.6.0.
> Similarly the 0.8.0 audit fixes are all on parse-time error paths;
> the small rise vs 0.6.0 is sample variance, not a regression.
> 1.0.1–1.0.8 were toolchain bumps with no source change and were not
> separately recorded; 1.0.9 (the 6.6.4 → 6.6.6 bump) and 1.0.10 (test
> and packaging changes only) are. For the 1.1.0 row the 1.0.10 binary was
> re-measured in the same session (the `1.0.10-rerun` rows in the CSV),
> because the host's absolute numbers had drifted ~5–8 % above the
> 2026-09-21 record: compare 1.1.0 with that row, not with the 2026-09-21
> one. 1.1.0's four new benchmarks (`cp437_from_unicode_*`,
> `font_row_builtin_cp`, `font_init`) are in the Current table and the CSV;
> `font_init` has a `1.0.10-rerun` row too. The three 2026-10-13 rows are
> one interleaved session (CSV versions `1.1.0-rerun`, `1.1.0-on-6.6.20`,
> `1.1.1`, all ten benchmarks): compare them with each other.
> `glyph_ptr` reads 8 there against 7 before — 7.5–7.6 ns rounding up,
> the same in all three. The 2026-10-29 rows are three interleaved
> sessions: `1.1.1-on-6.6.20` with `1.2.0`, then `1.2.0-on-6.7.9` with
> `1.2.1`, then `1.2.1-rerun` with `1.2.2`. Compare within a pair; the
> later pairs ran ~3 % lower across the board.

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
- **1.0.10 → 1.1.0 (`font_row_builtin` 21 → 24 ns, same session)**: the
  built-in path now maps the codepoint (ADR 0011) — a real, intended cost,
  per row on the convenience accessor; see "Hot-path render pattern" below
  for how to pay it once per glyph. `font_init` +1.2 µs (+6 %) is the extension glyphs plus the
  Unicode index fill; `scan_vga_8x16` +3 % is code placement (both shown
  above).
- **1.0.0 → 1.0.9 (`scan_vga_8x16` 64,000 → 56,193; `font_row_runtime_cp`
  66 → 58 ns)**: toolchain, not source. Nothing in `src/` changed between
  these rows; the pin moved 6.0.3 → 6.6.6 across 1.0.1–1.0.9, and the
  compiler's codegen got ~12 % faster on the sweep and the codepoint
  binary search. The glyph output is byte-identical across the bump (the
  glyph-sheet pin, `docs/examples/glyph_sheet.sha256`, is the proof), so
  this is a free improvement rather than a behaviour change.

Everything else is within ±2 ns / ±1,000 ns of the preceding row —
sample variance on a single host. The hot path has been **stable
within noise from 0.4.0 onward**, source-side; toolchain-side it has
only improved.

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
agnos's framebuffer console does. For a built-in font it also pays the
Unicode → cell mapping (1.1.0) once per glyph instead of once per row.

## Updating

Each release (or whenever an accessor's cost changes), append rows
to `benchmarks/history.csv` for the new version and refresh the
"Current" + "Trend" tables above. A pure toolchain bump is still a
release: record it, because the toolchain is the one thing that had
moved these numbers since 0.4.0 (1.1.0's mapping cost aside). Measure the
old and new binaries in one interleaved session — the host's absolute
figures drift by more than the deltas worth reporting. `scan_vga_8x16` is recorded
in ns in the CSV (27 µs → 27,000) so all columns share a unit; the
table renders the natural unit.

## Notes

- The audit fixes (`docs/audit/2026-05-27-audit.md`,
  `2026-05-28-audit.md`, `2026-05-28-audit-0.8.0.md`) touch only
  one-time init and error paths — never the hot accessors. The
  baseline is unchanged across them.
- `kashi_pcf_decode_glyph` (one-shot per glyph at load time) is not
  benched — it's amortized over the font lifetime, not per-render.
  Load-path benchmarks are booked for 1.3.x (`development/roadmap.md`),
  as the baseline for measuring the loaders' worst case.
