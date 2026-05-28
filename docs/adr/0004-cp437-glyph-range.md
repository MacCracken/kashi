# 0004 — Extend the freestanding range to full CP437 (VGA 8×16)

**Status**: Accepted
**Date**: 2026-05-27
**Refines**: [ADR 0001](0001-freestanding-font-data-core.md) (widens the
freestanding addressing range)

## Context

Through 0.3.0 kashi's freestanding-core addressing range was
`0x20–0x7F` (96 glyphs, printable ASCII). A framebuffer console wants
more than ASCII — at minimum the CP437 high half's line/box drawing
(`0xB0–0xDF`), block shading (`0xDB`/`0xDF`), and a smattering of accented
letters. The IBM VGA 8×16 BIOS font (public domain, IBM 1981) already
defines all 256 glyphs; kashi's existing ASCII half is byte-for-byte the
same source. Widening to the full CP437 range simply finishes that table.

agnos isn't yet wired (booked at agnos 1.38.0), so this is the right
moment to make the breaking contract change — agnos integrates the widened
range from day one rather than churning later.

## Decision

**Extend the freestanding addressing range from `0x20–0x7F` to `0x20–0xFF`**
(96 → 224 glyphs). Populate the VGA 8×16 high half with the public-domain
bytes from Linux's `lib/fonts/font_8x16.c`; leave the CGA 8×8 high half
intentionally blank (deferred — extending it would require hand-rolling
128 more glyphs).

- `KashiGlyphRange` enum constants change: `KASHI_GLYPH_LAST = 0xFF` (was
  `0x7F`), `KASHI_GLYPH_COUNT = 224` (was `96`); `KASHI_GLYPH_FIRST = 0x20`
  unchanged. The accessors (`kashi_glyph_encoded`, `kashi_glyph_ptr`,
  `kashi_glyph_row`, `kashi_font_count`) all derive from these — no
  accessor-logic change.
- BSS buffers grow per architecture note 001's `var X[N] = N bytes used`
  shape: `var kashi_font16[3584]` (224×16) and `var kashi_font8[1792]`
  (224×8).
- VGA 8×16: 128 new `kashi_fset16` calls for `0x80–0xFF`, generated
  mechanically from the fetched Linux source (one `awk` pass packing
  16-byte glyphs into the kashi `hi`/`lo` u64 form). Spot-fidelity
  asserts in `src/test.cyr` lock the transcription against the source.
- CGA 8×8: the high half is zeroed BSS — the slots exist (ptr non-null),
  every row reads `0`, renders blank. Documented and tested.

**Sourcing & licensing.** The byte values are facts about the IBM VGA
BIOS font (IBM 1981, no copyright on the bitmaps); kashi already credited
"the same byte table Linux's `font_8x16.c` carries" for the ASCII half.
The Linux *file* is GPL-2.0; the BYTE DATA inside is the PD IBM font,
which we're re-expressing in Cyrius — we're not copying the C wrapper.
This matches kashi's existing posture for the ASCII bytes.

## Consequences

- **Positive** — a kashi-consuming kernel console gets box-drawing,
  shading, line-art, and the common accented letters out of the box,
  matching what real text consoles expect from a VGA-style font.
- **Positive** — the accessor logic didn't change; only constants and
  buffer sizes. The freestanding boundary holds (`cyaudit vet
  src/font_data.cyr` → "no dependencies").
- **Breaking (pre-1.0)** — `KASHI_GLYPH_LAST`/`COUNT` change values.
  Consumers that hard-coded `0x7F`/`96` will be off; consumers that use
  the accessors (`kashi_font_count(id)`, `kashi_glyph_encoded(ch)`) are
  unaffected. Flagged in CHANGELOG **Changed**.
- **BSS grows** ≈ 2.3× (1536+768 bytes = 2304 used → 3584+1792 = 5376
  used; including the architecture-note-001 8× reservation, the actual
  BSS footprint grows 18 432 → 43 008 bytes ≈ 24 KiB more). Trivial for
  a kernel; the freestanding core's whole point is to be small but
  this is still small.
- **CGA 8×8 high half blank** — not "wrong", just not authored. A future
  cut can roll those glyphs or import a PD CP437 8×8 (Linux's
  `font_8x8.c` also exists, GPL-2.0 file wrapping the same PD bytes).
- **`scan_vga_8x16` bench** grows ≈ 2.3× (96 → 224 glyphs scanned);
  captured in `docs/benchmarks.md` / `history.csv`.

## Alternatives considered

- **Add a third font_id (e.g. "VGA CP437")** instead of widening the
  existing VGA's range. Rejected: forces consumers to choose a different
  id for box-drawing, and the existing VGA's ASCII half is already a
  subset of the same source — splitting them would be artificial.
- **Per-font addressing ranges** (each font carries its own
  `FIRST/LAST/COUNT`). Considered for the longer term, but for one
  widened built-in and one with-a-blank-high-half it's needless
  complexity now; the accessors already take `font_id` for everything
  except `kashi_glyph_encoded`, which is a global range check.
- **Roll the CGA 8×8 high half by hand** to match. Rejected for 0.4.0
  scope: that's another ~128 hand-drawn glyphs (substantial authoring).
  Deferred — possibly into 0.5.0's font additions, or skipped if the
  kernel's primary font stays VGA 8×16.
- **Wait and bundle with the 0.5.0 wide-glyph breaking change.** Rejected:
  the wide-glyph work is its own big lift; shipping CP437 now in 0.4.0
  lets a kashi-consuming kernel get box-drawing immediately rather than
  waiting on a much larger contract evolution.
