# 0009 — PCF import

**Status**: Accepted
**Date**: 2026-05-28
**Refines / extends**: [ADR 0002](0002-runtime-font-registry.md) (uses the
same runtime registry), [ADR 0003](0003-codepoint-addressing-runtime-fonts.md)
(uses the same codepoint→glyph map shape), [ADR 0008](0008-bdf-import.md)
(strict-metrics policy is the analog of strict-BBX)

## Context

0.7.0 added [BDF import](0008-bdf-import.md), the X11 *source* bitmap
font format. The companion format is **PCF** (Portable Compiled
Format) — what `bdftopcf` produces, what X servers actually load at
runtime. PCF is binary, denser than BDF, and the only bitmap font
format some distributions ship (older Linux desktops, BSD console
fonts, Solaris X11). 0.7.1 adds it.

PCF is materially more complex than PSF or BDF:

1. **Table-of-contents driven.** The file starts with an 8-byte
   header (magic + table count), then a TOC of `(type, format, size,
   offset)` quadruples, then the tables themselves at the indicated
   offsets. Tables can appear in any order; not every table type is
   present in every PCF.
2. **Per-table format byte.** Each table carries its own format word
   declaring its endianness (byte order), bit order within bytes,
   glyph row padding (1/2/4/8 bytes), scan unit (1/2/4/8 bytes), and
   for the METRICS table, whether metrics are compressed (signed-8
   per-field) or uncompressed (i16 per-field). All four byte-order
   × bit-order combinations exist in the wild — Linux/x86 produces
   LSB-byte / MSB-bit; older BSD/Sparc produces MSB-byte / MSB-bit
   with 4-byte scan units.
3. **Multiple tables needed.** A renderable PCF font requires (at
   minimum) `PCF_METRICS` (glyph dimensions), `PCF_BITMAPS` (the
   pixel data plus per-glyph offsets), and `PCF_BDF_ENCODINGS` (the
   codepoint → glyph index map). Many other tables exist
   (`PCF_PROPERTIES`, `PCF_ACCELERATORS`, `PCF_INK_METRICS`, …) but
   kashi doesn't need them.

The freestanding core (`src/font_data.cyr`) stays untouched — PCF is
library-face only, with a self-contained heapless parser
(`src/font_pcf.cyr`), the same posture as `font_psf.cyr` and
`font_bdf.cyr`.

## Decision

**A heapless PCF parser in `src/font_pcf.cyr` that canonicalizes
glyph bitmaps to MSB-bit / 1-byte-stride form on load; the library
face owns allocation, codepoint enumeration, and the registry.**

### Strict uniform metrics (analog of ADR 0008's strict BBX)

The parsed METRICS table must report uniform per-glyph dimensions:
every glyph's `(left_sb, right_sb, char_width, ascent, descent)` must
match a single reference tuple (the first glyph's). A mismatch →
`KASHI_PCF_EFORMAT`. Glyph width = `right_sb - left_sb`, glyph height
= `ascent + descent`.

Console PCFs (Terminus, Spleen, Tamzen, Cozette PCFs, the bitmap
output of `bdftopcf` on a uniform BDF) all satisfy this. Display
PCFs with per-glyph kerning are out of scope for 0.7.1 — relaxing to
"pad varying metrics into a max cell" would need signed coordinate
arithmetic and is reserved for a later additive cut.

### Both compressed and uncompressed metrics layouts

PCF's METRICS table has two on-disk layouts:

- **Compressed** (format bit `0x100` set): one byte per field, biased
  by `+0x80` so byte `0x80` = 0, `0x7F` = -1, `0x81` = +1. 5 fields
  per glyph (no `attributes`) = 5 bytes/glyph.
- **Uncompressed** (format bit `0x100` clear): two bytes per field
  (LE or BE per the format byte's byte-order bit). 6 fields per
  glyph (last field is `attributes`) = 12 bytes/glyph.

Modern `bdftopcf` produces compressed; uncompressed exists in older
fonts. Both are supported; one extra layout dispatch in the parser.

### All four byte-order × bit-order combos, normalized to MSB-bit on load

PCF can be stored with byte order = LSB or MSB, and bit order = LSB
(bit 0 of a byte is the leftmost pixel) or MSB (bit 7 is leftmost).
kashi's internal convention is MSB-bit (bit 7 = leftmost) — every
existing accessor and built-in font assumes this.

On load, the parser canonicalizes:

1. **Scan-unit byte swap** if `byte_order != bit_order` (the libXfont
   "TwoByteSwap" / "FourByteSwap" passes). Reverses byte order
   within each scan-unit-byte group.
2. **Bit-reverse each byte** if `bit_order != MSBFirst`. Turns LSB-bit
   layout into MSB-bit.

After these passes, byte 0 of each glyph row holds the leftmost 8
pixels with bit 7 = leftmost, matching kashi's convention. The
canonicalization is done per-glyph at decode time into a small
function-local scratch (max 8 bytes per row at scan_unit=8/pad=8),
then the first `ceil(width/8)` bytes (= our_stride) are copied to the
owned runtime store.

### File-size cap = 4 MiB (`kashi_load_pcf_file`)

Same cap as BDF. PCF is binary and ~10× denser than BDF source — 4
MiB covers every console PCF in the wild plus full Unifont. Reusing
the BDF constant is tempting but keeping a per-format constant is
clearer; `KASHI_PCF_FILE_CAP = 4194304` lives in `font_pcf.cyr` for
grep-discoverability.

### `KIND = 4` in the parsed-header struct

The PSF parser writes `KIND = 1` (PSF1) or `KIND = 2` (PSF2);
`font_bdf.cyr` writes `KIND = 3` (BDF); `font_pcf.cyr` writes
`KIND = 4`. Same 56-byte parsed-header shape, with `WIDTH`,
`HEIGHT`, `COUNT`, `CHARSIZE` (= height × ceil(width/8)), `DATAOFF`
(= byte offset of the BITMAPS table's pixel data within the source
buffer — diagnostic, not used by lib.cyr), and `UNIOFF` = 0 (PCF has
no separate Unicode table; codepoints live in `PCF_BDF_ENCODINGS`).

### Required tables, ignored tables

Required (any missing → `KASHI_PCF_EFORMAT`):
- `PCF_METRICS` (0x04) — glyph dimensions.
- `PCF_BITMAPS` (0x08) — pixel data + per-glyph bitmap offsets.
- `PCF_BDF_ENCODINGS` (0x20) — codepoint → glyph index map.

Ignored (the TOC entries are walked past without parsing the bodies):
- `PCF_PROPERTIES` (0x01) — font metadata.
- `PCF_ACCELERATORS` (0x02) — rendering acceleration data.
- `PCF_INK_METRICS` (0x10) — ink metrics.
- `PCF_SWIDTHS` (0x40) — scalable widths.
- `PCF_GLYPH_NAMES` (0x80) — glyph names.
- `PCF_BDF_ACCELERATORS` (0x100) — accelerator-with-BDF-info.

Same posture as BDF's "STARTPROPERTIES walked past without parse"
decision (ADR 0008).

### Parser API (consumed by `src/lib.cyr`)

```
kashi_pcf_parse_header(buf, len, hdr, ctx)
  Validates magic, walks the TOC, validates the three required
  tables' format bytes and uniform metrics, computes width/height,
  fills hdr (56-byte parsed-header struct; KIND=4) and ctx (128-byte
  PCF context: glyph_count, width, height, source/target row strides,
  table offsets within `buf`, format flags, BDF_ENCODINGS range).

kashi_pcf_decode_glyph(buf, ctx, glyph_idx, dest, dest_size)
  Decodes glyph `glyph_idx` into `dest` (>= height * our_stride
  bytes). Applies scan-unit byte swap + bit-reverse to produce
  canonical MSB-bit/1-byte-stride output. Returns KASHI_PCF_OK or
  KASHI_PCF_EFORMAT.

kashi_pcf_cp_to_idx(buf, ctx, cp)
  Resolves a codepoint to a glyph index via the BDF_ENCODINGS table.
  Returns the glyph index (>= 0, < glyph_count) or 0 - 1 if
  unmapped (out of range, or sentinel 0xFFFF in the table).
```

The `_cp_to_idx` function is exposed so the load path can enumerate
encoded codepoints without re-parsing the BDF_ENCODINGS table.

## Consequences

- **Positive** — kashi now imports the X11 compiled-binary font
  format, completing the BDF / PCF pair. Many distributions ship
  bitmap fonts as PCF only.
- **Positive** — `cyaudit vet src/font_pcf.cyr` reports "no
  dependencies"; the freestanding boundary (`src/font_data.cyr`) is
  untouched.
- **Positive** — the canonicalize-on-load design means runtime glyph
  bytes are byte-identical regardless of the source PCF's byte/bit
  order. Existing kashi consumers see the same bit-7-leftmost layout
  PSF and BDF already provide.
- **Negative** — PCF parsing is the biggest untrusted-input surface
  kashi has so far. TOC offsets, signed metric biases, variable-
  length BDF_ENCODINGS arrays, format-byte combinatorics. The fuzz
  harness gets a dedicated 1500-round PCF path with the same four
  pick variants as BDF (random / template / mutated / truncated).
- **Negative** — strict uniform metrics rules out display PCFs with
  per-glyph kerning. Documented up front; a future "lenient with
  cell padding" cut can be additive (per ADR 0008's same tradeoff).
- **Negative** — supporting all four byte/bit combos adds the
  scan-unit byte swap and bit-reverse paths (~30 lines combined).
  Without them we'd reject ~half of real PCFs.
- **Neutral** — the runtime registry record layout (`RT_*` offsets in
  `lib.cyr`) does not change — PCF fonts use exactly the same record
  fields PSF and BDF already use. Sequence/ligature fields stay zero
  (PCF carries no ligature data — that's a BDF/PSF construct).
- **Neutral** — a new `KASHI_PH_KIND = 4` value appears; consumers
  branching on kind values must learn it. No existing consumer reads
  `KASHI_PH_KIND` (it's an internal parsed-header field), so this is
  a no-op in practice.

## Alternatives considered

- **Lenient metrics with cell padding.** Rejected for 0.7.1 — same
  reasoning as the BDF BBX call (ADR 0008). The console-bitmap corpus
  doesn't need it; deferring keeps the cut small.
- **Drop LSB-bit / drop scan-unit-swap support.** Considered, but
  would reject half of real PCFs (LSB-bit is common on PowerPC /
  ARM-built PCFs; scan-unit=4 with byte_order != bit_order is the
  classic Sparc/Solaris output). The transform code is ~30 lines and
  isolated to a per-row canonicalization helper.
- **Drop uncompressed metrics support.** Rejected — only ~10 lines
  of extra parsing dispatched off the compressed-bit. Older fonts
  use uncompressed; keeping both maximizes compatibility.
- **Pre-decode all glyphs into the parsed-header struct.** Rejected
  — that would require allocation from a pure parser, breaking the
  heapless posture. Decoding happens in lib.cyr's loop where alloc
  is already in scope.
- **Reuse BDF's KASHI_BDF_FILE_CAP constant.** Rejected — a
  per-format constant is clearer to grep and tweak independently.
  Both happen to land at 4 MiB; that's coincidence, not coupling.
- **Use the X11 `PCF_FILE_VERSION` u32 (`0x70636601` LE) as a
  combined magic check.** Equivalent to the four-byte `0x01 'f' 'c'
  'p'` check; spelled as four bytes for grep-discoverability in the
  parser.
