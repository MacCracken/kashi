# 0003 — Codepoint addressing for runtime fonts via the PSF Unicode table

**Status**: Accepted
**Date**: 2026-05-27
**Refines**: [ADR 0002](0002-runtime-font-registry.md)

## Context

ADR 0002 (M1) shipped PSF import addressing runtime glyphs by **raw glyph
index**, and explicitly named the resulting wart: the unified
`kashi_font_row(id, ch, row)` treated `ch` as an ASCII codepoint for the
built-in fonts but as a glyph index for runtime fonts. PSF files carry an
optional Unicode→glyph table that M1 validated-but-ignored. M2 builds that
table into a map so runtime fonts can be addressed by codepoint like the
built-ins — what a real text console needs.

## Decision

**Parse the PSF Unicode table at load time into a per-font
codepoint→glyph map, and make the unified accessors codepoint-addressed.**

- **Parsing** stays in the pure, heapless `src/font_psf.cyr`:
  `kashi_psf_parse` now reports the table's byte offset (`KASHI_PH_UNIOFF`,
  0 if none), and `kashi_psf_uni_token` decodes one table unit at a time
  (PSF1 little-endian u16; PSF2 UTF-8), always bounds-checked — a malformed
  or truncated table yields `KASHI_TOK_END`, never an over-read.
- **Map build** lives in the library face (`src/lib.cyr`, needs the heap):
  walk the table with `kashi_psf_uni_token`, collect single-codepoint
  mappings (sequence entries after a `0xFFFE`/`0xFE` separator are skipped),
  and store a `{codepoint, glyph}` pair array sorted by codepoint in the
  runtime record (`RT_UMAP`/`RT_UCOUNT`). Lookup is a binary search.
- **Unified accessors are codepoint-addressed**:
  `kashi_font_ptr(id, cp)` / `kashi_font_row(id, cp, row)` — built-ins:
  `cp` is an ASCII codepoint (unchanged); runtime **with** a map: `cp`
  resolved via the map (unmapped → null/0, blank); runtime **without** a
  table: `cp` falls back to a raw glyph index (identity) so table-less fonts
  still work.
- **Raw glyph-index access** is preserved as its own pair,
  `kashi_rt_glyph_ptr(id, idx)` / `kashi_rt_glyph_row(id, idx, row)`, for
  tooling, enumeration, and explicit index addressing.

## Consequences

- **Positive** — the ADR 0002 wart is resolved for the common case: a PSF
  console font (which ships a Unicode table) is addressed by codepoint
  through the same accessor as the built-ins. Non-ASCII glyphs (e.g.
  U+00E9) resolve correctly.
- **Breaking (pre-1.0)** — `kashi_font_row`/`_ptr` runtime semantics change
  from "glyph index" (M1/0.2.0) to "codepoint" when the font has a table.
  A 0.2.0 consumer that loaded a table-bearing font and passed glyph indices
  must switch to `kashi_rt_glyph_*`. Flagged in CHANGELOG **Changed**;
  acceptable under the pre-1.0 "surface still moving" note.
- **Residual wart** — the addressing is still not perfectly uniform: a
  runtime font *without* a Unicode table is addressed by index through
  `kashi_font_*` (identity fallback). This is the honest behavior (a
  table-less font carries no codepoint information) and is documented.
- **Performance** — codepoint resolution is one binary search per glyph
  (~log₂(map) comparisons). The hot path stays cheap by resolving once via
  `kashi_font_ptr` then reading rows with `load8(ptr+row)`; the per-call
  `kashi_font_row(id, cp, row)` re-resolves and is the convenience form
  (see `docs/benchmarks.md`).

## Alternatives considered

- **Keep index addressing; add a separate `*_cp` accessor.** Rejected: the
  whole point of the unified accessor is one console-facing entry point;
  splitting it perpetuates the wart instead of resolving it.
- **A direct codepoint→glyph lookup table (array indexed by codepoint).**
  Rejected: a full BMP table is 64 Ki entries per font; console fonts map a
  sparse handful of codepoints, so a sorted pair array + binary search is
  far smaller and fast enough.
- **Map ligature/sequence entries (`0xFFFE`/`0xFE`).** Deferred: kashi hands
  back single glyph bitmaps; multi-codepoint sequences need a shaping layer
  the consumer (or a future lib) owns. Sequences are parsed-and-skipped.
