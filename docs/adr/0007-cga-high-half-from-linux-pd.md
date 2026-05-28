# 0007 — CGA 8×8 high half from Linux's public-domain `font_8x8.c`

**Status**: Accepted
**Date**: 2026-05-28
**Refines / extends**: [ADR 0004](0004-cp437-glyph-range.md) (CP437 range
widening was done for VGA 8×16; the CGA high half stayed blank then) and
[ADR 0001](0001-freestanding-font-data-core.md) (CGA 8×8 was originally
"hand-drawn AGNOS original").

## Context

0.4.0 widened the freestanding-core addressing range to full CP437
(`0x20..0xFF`, 224 glyphs) and populated the VGA 8×16 high half from
Linux's PD `lib/fonts/font_8x16.c`. The CGA 8×8 high half stayed
intentionally blank: it was hand-drawn AGNOS original, and authoring 128
more 8×8 glyphs at that aesthetic was deferred. Until 0.5.1 ran out of
remaining M2 deliverables, the CGA high half had to either get filled or
keep documenting the blank.

The Linux kernel ships an `lib/fonts/font_8x8.c` — the same PD IBM CGA 8×8
ROM bytes, GPL-2.0 file wrapping the PD data — which provides all 256
glyphs (the CP437 high half being mostly geometric: line/box drawing,
block shading, plus a long tail of accented letters / Greek / math).

## Decision

**Fill the CGA 8×8 high half (0x80..0xFF) with bytes from Linux's
`lib/fonts/font_8x8.c`.** kashi's CGA font becomes **dual-sourced**:
- Low half (`0x20..0x7F`) — hand-drawn AGNOS original (unchanged since
  0.1.0).
- High half (`0x80..0xFF`) — public-domain IBM CGA 8×8 from Linux PD
  (added 0.5.2).

Mechanically generated from the fetched source (one `awk` pass packing
8 bytes per glyph into the kashi `fset8` u64 form), then spot-fidelity
asserts in `src/test.cyr` lock the transcription against the source.

**License posture** is the same as the 0.4.0 VGA CP437 extension: the
byte values are facts about the IBM CGA ROM font (PD); the Linux file
wrapping them is GPL-2.0, but we're not copying the C wrapper — we're
re-expressing the same PD byte data in Cyrius `fset8` form. kashi's
license stays GPL-3.0-only.

## Consequences

- **Positive** — the CGA 8×8 built-in now covers full CP437 like the VGA
  8×16 does, so kashi-consuming kernels picking the CGA font get
  box-drawing, dithering, accented letters out of the box. No more
  "renders blank" caveat for the high half.
- **Positive** — boundary intact: vet still "no dependencies"; the new
  bytes are static `fset8` calls in the existing init path. BSS sizing
  was already provisioned (`var kashi_font8[1792]` covers all 224 slots
  since 0.4.0); just filling previously-zero memory.
- **Style mismatch** — the hand-drawn AGNOS ASCII half doesn't perfectly
  match the IBM PD high half in line weight or curve style. For the
  box-drawing range `0xB0..0xDF` the difference is invisible (these
  glyphs are purely geometric: lines, blocks, shading). For accented
  letters (`0x80..0xAF`, `0xE0..0xFF`) the high-half characters may look
  slightly different in stroke style from the ASCII ones. Acceptable;
  documented in the `font_data.cyr` comment above the CGA high-half
  block. Coverage beats blank.
- **No behavior change for ASCII consumers** — the low half is byte-for-
  byte unchanged; existing CGA `0x20..0x7F` fidelity asserts still hold.

## Alternatives considered

- **Hand-roll the 128 high-half glyphs in the AGNOS style.** Faithful to
  the original aesthetic but ~7 KB of careful art for a font whose
  primary value is being available, not visually unique. Rejected:
  authoring effort vs. payoff is poor when a known-good PD source exists.
- **Add a separate `KASHI_FONT_IBM_CGA_8X8 = 3` built-in** using Linux
  bytes for the full range, leaving the existing hand-drawn CGA as
  ASCII-only. Cleaner aesthetic separation, but adds another `font_id`,
  bumps `KASHI_RT_FONT_BASE` again (to 4), and gives consumers two
  fonts to pick between for what is essentially one CGA-style 8×8 — not
  worth the API surface for the slight aesthetic gain.
- **Replace the low half too with Linux bytes** — single style across
  the font. Rejected: the existing hand-drawn ASCII bytes are
  asserted in fidelity tests (would break them), and the AGNOS original
  is part of the project's history.
