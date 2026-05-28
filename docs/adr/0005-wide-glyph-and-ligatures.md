# 0005 — Wide-glyph rows + PSF ligature lookup

**Status**: Accepted
**Date**: 2026-05-27
**Refines / extends**: [ADR 0002](0002-runtime-font-registry.md) (accessor
shape), [ADR 0003](0003-codepoint-addressing-runtime-fonts.md) (the
Unicode-table walk now also harvests sequences)

## Context

Through 0.4.0 every kashi accessor was 8-pixel-wide: `kashi_glyph_row` and
`kashi_font_row` returned **one byte** (= 8 px), and the PSF parser
rejected width > 8 outright with `KASHI_EFORMAT`. That was fine for the
freestanding built-ins (VGA 8×16, CGA 8×8) and the common ASCII PSF
fonts, but ruled out:

- PSF2 console fonts wider than 8 px (common in modern Linux consoles).
- PSF Unicode tables' **sequence/ligature** entries (multi-codepoint→glyph
  mappings, separated by `0xFFFE`/`0xFE`), which 0.3.0 parsed-and-skipped.

0.5.0 widens the contract. agnos isn't yet wired (booked at agnos 1.38.0),
so this is the right moment to evolve the freestanding-core accessor
shape; the changes are **backward-compatible** for any 8-wide consumer.

## Decision

### Multi-byte row access (additive)

Two new accessors per id-space; nothing existing changes shape:

- **`kashi_font_stride(font_id)`** (freestanding core) +
  **`kashi_rt_font_stride(font_id)`** (library, for runtime fonts) —
  bytes per row = `ceil(width / 8)`. Both built-ins return `1`.
- **`kashi_glyph_row_byte(font_id, ch, row, byte_idx)`** (freestanding) +
  **`kashi_rt_glyph_row_byte(font_id, idx, row, byte_idx)`** (library
  raw-index) + **`kashi_font_row_byte(font_id, cp, row, byte_idx)`**
  (library codepoint-addressed) — read one byte at a column-byte position
  within a row. `byte_idx 0` is the leftmost 8 px; for stride-1 fonts
  only `byte_idx 0` is meaningful and any other returns `0`.

The existing single-byte accessors (`kashi_glyph_row`, `kashi_rt_glyph_row`,
`kashi_font_row`) keep their shape; for wide fonts they return the
**leading byte** (`byte_idx 0`). Documented; no breaking change for the
8-wide world (which includes agnos's eventual consumer).

### Wider widths accepted in the parser + runtime registry

- `KASHI_PSF_MAX_WIDTH` rises from `8` to `32` (4 bytes/row max).
- The parser's strict `charsize == height` check becomes
  `charsize == height * ceil(width/8)` — correct for any width and still
  catches PSF files whose declared charsize disagrees with their geometry.
- `kashi_register_font` accepts width 1–32 and computes
  `charsize = height * stride` internally; `kashi_load_psf` derives stride
  from the parsed `charsize / height`.
- Runtime registry record grows 56 → 88 bytes (adds `RT_STRIDE`,
  `RT_SEQS`, `RT_SEQCOUNT`, `RT_SEQDATA`).
- Storage layout for wide glyphs: `glyph idx` lives at offset
  `idx * height * stride`; `row R` starts at `+ R * stride`; `byte B` of
  the row at `+ B`. The library accessors compute these directly.

### PSF Unicode sequence/ligature lookup

PSF Unicode tables can record multi-codepoint→single-glyph mappings —
they appear after a `0xFFFE` (PSF1) or `0xFE` (PSF2) separator and run
until the next separator or terminator. 0.3.0 parsed and *skipped* them.

0.5.0 collects them. The existing `_kashi_build_umap` (singles) stays as
is; a new sibling `_kashi_build_useqs` walks the same table in a two-pass
algorithm:

1. Pass 1: count sequences and total codepoint length.
2. Alloc a flat record array (`24 B × N`: `{cps_ptr, length, glyph}`) +
   a single backing store for all codepoints.
3. Pass 2: fill records and the backing store.

Public lookup: **`kashi_font_seq_glyph(font_id, cps_ptr, len)`** — linear
scan (sequences are typically few per font, no hot-path concern). Returns
the matched glyph index, or `0 - 1` for any miss (no match, built-in id,
no table, null ptr, non-positive len). Fuzzed.

kashi remains a **glyph-data provider**, not a shaping layer — the lookup
is the data the consumer's shaping logic needs; kashi itself doesn't do
shaping, BiDi, or complex-script handling (still out of scope per the
roadmap).

## Consequences

- **Positive** — runtime PSF fonts wider than 8 px now load; ligature
  data is accessible via a tiny additive API; the freestanding accessor
  contract evolves with no breaking change to existing 8-wide consumers.
- **Positive** — `cyaudit vet src/font_data.cyr` still reports
  "no dependencies"; `src/font_psf.cyr` stays self-contained
  ("no dependencies"). Boundary intact.
- **Neutral** — for the consumer there are now two parallel single-byte
  accessors (`*_row`, `*_row_byte`); the byte-index sibling is the
  "complete" one, the byte-only is the 8-wide convenience. Documented.
- **Neutral** — runtime record grows 56 → 88 B per font; bump-allocator
  storage rises a small amount per loaded font. Sequences also allocate
  a 24-B record per sequence plus a `8 × total_cps` codepoint store.
- **Note** — addressing for wide fonts uses the same codepoint-vs-index
  rules as ADR 0003: mapped via the Unicode map, identity-index for
  table-less fonts. No new wart introduced.

## Alternatives considered

- **Widen `kashi_glyph_row` to return `u64`** (8 packed row bytes).
  Cleaner single-call API for moderate widths, but a breaking semantic
  change to the existing 8-wide accessor (the byte vs u64 distinction
  matters at the consumer's blit site) and changes the agnos contract.
  Rejected; the byte-at-position model preserves backward compatibility.
- **Bake a 9×16 box-drawing built-in into this cut.** Considered, but
  the wide-glyph infrastructure is the load-bearing change; layering on
  a new built-in (whether hand-rolled or derived via the VGA col-9 rule)
  would balloon the cut. Deferred to a later release.
- **Sequence/ligature shaping inside kashi.** Out of scope — kashi
  hands back glyph bitmaps; shaping/BiDi belongs in the consumer (or a
  future shaping library). 0.5.0 exposes the *data* (lookup helper);
  shaping policy stays out.
- **Hash-table lookup for sequences.** Overkill — fonts carry at most a
  few dozen sequences; linear scan is fast enough and avoids a hash
  implementation in kashi.
