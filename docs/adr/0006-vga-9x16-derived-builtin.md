# 0006 — VGA 9×16 derived built-in (col-9 replication rule)

**Status**: Accepted
**Date**: 2026-05-27
**Refines / extends**: [ADR 0001](0001-freestanding-font-data-core.md)
(adds a third freestanding built-in font),
[ADR 0005](0005-wide-glyph-and-ligatures.md) (first wide-glyph consumer in
the freestanding core).

## Context

ADR 0005 (0.5.0) shipped wide-glyph infrastructure — `kashi_font_stride`,
`kashi_glyph_row_byte`, multi-byte rows — but the freestanding-core
built-ins stayed 8 wide; runtime PSF fonts were the only wide consumers.
The roadmap listed a 9×16 box-drawing built-in as the natural next built-in
addition, since VGA text-mode kernels often want 9-pixel cells (so
box-drawing line characters connect across cell boundaries).

The IBM/VGA hardware itself doesn't have a separate 9×16 font ROM — the
8×16 ROM font is "stretched" to 9 columns by the CRT controller: for
codepoints in the CP437 box-drawing range `0xC0..0xDF`, the 9th column
repeats the rightmost (bit 0) pixel of the underlying 8-wide row; for all
other codepoints, the 9th column is blank. That single rule turns a
universally-available 8×16 ROM font into a 9×16 display.

## Decision

**Ship the VGA 9×16 built-in as a deterministically-derived font, not a
new byte table.** A new font_id `KASHI_FONT_VGA_9X16 = 2` is added to the
freestanding core; the byte buffer (`var kashi_font9_16[7168]`, 224 × 16 ×
2 bytes) is filled at `kashi_font_init` time by walking the existing VGA
8×16 and applying the rule:

```
byte0 = byte from font16
byte1 = (cp in [0xC0, 0xDF]) ? ((byte0 & 0x01) << 7) : 0
```

No new glyph table is committed; the new font is a function of the existing
one. The derivation runs once at init alongside the existing `fset*` calls
and stays in the freestanding core (pure arithmetic + `load8`/`store8`;
`cyaudit vet` keeps reporting "no dependencies").

**Consequential change — `KASHI_RT_FONT_BASE: 2 → 3`.** The library face's
unified dispatchers route by `font_id < KASHI_RT_FONT_BASE` (built-in) vs
`≥` (runtime registry). With the new built-in at id 2, the boundary moves
to 3 so that id 2 routes to the core. Runtime ids assigned by
`kashi_register_font`/`kashi_load_psf` now start at 3 instead of 2.

**`kashi_glyph_row` becomes stride-aware** (one-line generalization,
multiply offset by `kashi_font_stride(font_id)`). For stride-1 fonts
`row*1 == row`, behavior unchanged; for the new 9×16 it returns the
LEADING byte of the row (matching the ADR 0005 documented semantics for
wide fonts).

## Consequences

- **Positive** — kernel consoles wanting a 9-pixel cell get
  visually-correct box-drawing (lines connect cell-to-cell) out of the
  box, with one PD source (the existing VGA 8×16) and zero new
  transcribed bytes.
- **Positive** — the wide-glyph path in the freestanding core finally has
  a real built-in consumer (was previously infrastructure-only); the
  test suite exercises the multi-byte accessors against a concrete font.
- **Positive** — boundary intact: vet still "no dependencies"; the
  derivation is pure arithmetic.
- **Breaking (pre-1.0)** — `KASHI_RT_FONT_BASE` value changed `2 → 3`.
  Consumers that read the constant symbolically are unaffected; any
  consumer that hard-coded `2` would be off. Flagged in CHANGELOG.
- **BSS grows** by 7168 bytes used (8× reservation = 57 344 bytes) for
  the new font buffer. Modest for a kernel; the freestanding core is
  still small.
- **Init does more work** — one extra loop over 224×16 (3584 iterations)
  on `kashi_font_init`. One-time at boot; trivial cost.
- **Note** — the col-9 rule applies only to `0xC0..0xDF`. For other
  glyphs, the 9th column is blank — matching real VGA hardware. This
  means non-box-drawing glyphs effectively render at 8-pixel width in a
  9-pixel cell (with a blank 9th column), which is what every IBM-style
  text console does.

## Alternatives considered

- **Ship a separately-sourced 9×16 PD font.** Considered — there are
  several PD 9×16 fonts (some BDF / Linux KBD packages carry them). But
  there's no single canonical 9×16 PD source the way `font_8x16.c` is for
  8×16; sourcing one risks license confusion and would mix unrelated
  authoring styles with the existing VGA aesthetic. The VGA-rule
  derivation matches the established VGA-display behavior consumers
  expect.
- **Hand-roll a 9×16 in the AGNOS style** (like the CGA 8×8). Rejected:
  authoring 224 glyphs at 9×16 = ~7 KB of careful art for a font whose
  primary value is matching standard VGA box-drawing behavior — which is
  precisely what the derived approach gives for free.
- **Compute the 9th column on every accessor call** (lazy, no extra BSS).
  Rejected: the per-call branch cost across the render hot path is worse
  than a one-time 3.5K init pass, and the BSS overhead is small enough
  not to matter. Pre-computing keeps the accessor a clean `load8(base +
  row*stride + byte_idx)`.
