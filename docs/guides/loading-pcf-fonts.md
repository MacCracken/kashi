# Loading PCF fonts at runtime

> **Last Updated**: 2026-05-28 (0.7.1)

The **library face** (`src/lib.cyr`) can import [PCF][pcf] files
(Portable Compiled Format — X11's compiled binary bitmap font). PCF
is what `bdftopcf` produces and what X servers actually load at
runtime; many distributions ship bitmap fonts (Terminus, Spleen,
Tamzen, Cozette, Unifont) as PCF only. The freestanding core
(`src/font_data.cyr`) does **not** get this path. Design rationale:
[ADR 0009](../adr/0009-pcf-import.md).

[pcf]: https://en.wikipedia.org/wiki/Portable_Compiled_Format

## Loading

```cyrius
include "src/lib.cyr"

# From a file:
var id = kashi_load_pcf_file("/usr/share/fonts/X11/misc/ter-u14n.pcf");
if (id < 0) {
    # 0 - KASHI_EFORMAT / 0 - KASHI_EINVAL — malformed, unsupported, or unreadable
    return 1;
}

# Or from a buffer you already hold (e.g. an embedded asset):
var id2 = kashi_load_pcf(buf, len);
```

**Return convention**: success is a `font_id ≥ KASHI_RT_FONT_BASE`;
failure is a **negative** `0 - <code>` (`KASHI_EINVAL` for bad args /
unreadable file, `KASHI_EFORMAT` for malformed / unsupported PCF).
Same shape as PSF and BDF.

## Reading glyphs

PCF fonts plug into the same codepoint-addressed accessors as PSF
and BDF:

```cyrius
var rows = kashi_rt_font_height(id);
var w    = kashi_rt_font_width(id);
var n    = kashi_rt_font_count(id);

var row = 0;
while (row < rows) {
    var bits = kashi_font_row(id, codepoint, row);   # leading row byte
    row = row + 1;
}
```

For wide fonts (`width > 8`), use the byte-position accessor exactly
as the PSF guide describes — `kashi_rt_font_stride(id)` reports
bytes/row; `kashi_font_row_byte(id, cp, row, bi)` reads one byte of
a wider row.

> **Addressing**: `kashi_font_row` / `kashi_font_ptr` resolve the
> codepoint through the font's `PCF_BDF_ENCODINGS` table (built at
> load time). An unencoded codepoint returns `0` (blank), never a
> wild read.

> **Canonical bytes**: regardless of the source PCF's byte/bit order,
> the runtime glyph store always holds the kashi convention — byte 0
> of each row = leftmost 8 pixels, bit 7 = leftmost pixel. The
> parser canonicalizes on load (scan-unit byte swap + bit reverse as
> needed).

## What kashi accepts (the PCF subset)

kashi's PCF parser handles the **console-bitmap-font subset** —
uniform-cell fonts. Specifically:

- **Magic** `0x01 'f' 'c' 'p'` (the X11 `PCF_FILE_VERSION` u32).
- **Table-of-contents** with up to 64 entries.
- **Required tables**: `PCF_METRICS` (0x04), `PCF_BITMAPS` (0x08),
  `PCF_BDF_ENCODINGS` (0x20). Any missing → `KASHI_EFORMAT`.
- **Ignored tables**: `PCF_PROPERTIES`, `PCF_ACCELERATORS`,
  `PCF_INK_METRICS`, `PCF_SWIDTHS`, `PCF_GLYPH_NAMES`,
  `PCF_BDF_ACCELERATORS`. Their TOC entries are walked past without
  parsing the bodies.

### Strict uniform metrics

Every glyph's `(left_sb, right_sb, char_width, ascent, descent)`
must match a single reference (the first glyph's). A mismatch →
`KASHI_EFORMAT`. This is the PCF analog of the BDF strict-BBX
policy. Glyph width = `right_sb - left_sb`, glyph height =
`ascent + descent`.

Per-glyph kerning fonts (display fonts with overhangs) are out of
scope for 0.7.1; see [ADR 0009](../adr/0009-pcf-import.md) for the
rationale and the path to lifting the restriction.

### Both compressed and uncompressed metrics

The METRICS table's two on-disk layouts are both supported:
- **Compressed** (`format & 0x100` set): 5 bytes/glyph, each field
  biased by `+0x80`. Modern `bdftopcf` produces this.
- **Uncompressed** (`format & 0x100` clear): 12 bytes/glyph,
  i16 fields in the table's byte order.

### All four byte-order × bit-order combos

PCF can be stored LSB-byte / MSB-byte and LSB-bit / MSB-bit
(format byte bits 2 and 3). On load, the parser canonicalizes to
MSB-bit / 1-byte-stride form:

1. **Scan-unit byte swap** if `byte_order != bit_order`.
2. **Bit-reverse each byte** if `bit_order != MSB`.

After this, byte 0 of each row holds the leftmost 8 pixels with bit
7 = leftmost. Matches kashi's existing convention for built-ins,
PSF, and BDF.

The parser requires `glyph_pad >= scan_unit` so the per-row
byte-swap stays within row boundaries; pathological PCFs with
smaller `glyph_pad` than `scan_unit` are rejected with
`KASHI_EFORMAT`. Real PCFs all satisfy this.

### Geometry caps

- **Width 1–32, height 1–32** (matching PSF and BDF caps).
- **Glyph count ≤ 65,536**.
- **TOC ≤ 64 entries** (real PCFs have 5–10).

### File-size cap

`kashi_load_pcf_file` reads at most **4 MiB** of source
(`KASHI_PCF_FILE_CAP`). PCF is binary and ~10× denser than BDF —
4 MiB covers full Unifont. Files larger than the cap are rejected.

## Limits and posture

- **Validation-first**: magic, TOC, table bounds, metric uniformity,
  glyph_count consistency across METRICS and BITMAPS — all checked
  before any glyph byte is read.
- The parser (`src/font_pcf.cyr`) is heapless and dependency-free
  (`cyaudit vet` → "no dependencies").
- Fuzzed in `tests/kashi.fcyr` (1500 rounds across four pick paths:
  random / valid template / mutated template / truncated template).
  PCF is kashi's biggest untrusted-input surface; the fuzz harness
  is sized accordingly.
- Loaded fonts own a copy of their canonicalized glyph bytes — the
  source buffer is free to reuse after loading. The runtime
  registry's bump allocator does not individually free per-font
  storage.

## Picking between PSF, BDF, and PCF

| | PSF | BDF | PCF |
|---|---|---|---|
| **Format** | binary, packed | text, verbose | binary, table-of-contents |
| **Source size** | tight | ~10× of PSF | similar to PSF |
| **Loaded via** | `kashi_load_psf*` | `kashi_load_bdf*` | `kashi_load_pcf*` |
| **File cap** | 256 KiB | 4 MiB | 4 MiB |
| **Codepoint map** | PSF1 LE-u16 / PSF2 UTF-8 | per-glyph `ENCODING <int>` | `PCF_BDF_ENCODINGS` 2D table |
| **Ligatures** | yes — `kashi_font_seq_glyph` (ADR 0005) | no | no |
| **Metrics** | uniform header | strict BBX match (ADR 0008) | strict uniform (ADR 0009) |
| **Use when** | shipping the font compiled-in or to a kernel | loading from an X11 source distribution | loading what X servers / pre-built bitmap fonts ship |

If a font is available in multiple formats, prefer PSF (smallest,
fastest parse). PCF is the closest alternative — same binary density
as PSF, no ligature concept, but the X11 ecosystem's primary runtime
form so most upstreams ship it.
