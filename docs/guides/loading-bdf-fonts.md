# Loading BDF fonts at runtime

> **Last Updated**: 2026-05-28 (0.7.0)

The **library face** (`src/lib.cyr`) can import [BDF][bdf] (Bitmap
Distribution Format) source files at runtime alongside the PSF1/PSF2
path — same runtime registry, same codepoint-addressed accessors. BDF
is the de-facto X11 source format for bitmap fonts (Terminus, Spleen,
Tamzen, Cozette, Unifont all ship as BDF first). The freestanding core
(`src/font_data.cyr`) does **not** get this path; only the library
face does. Design rationale: [ADR 0008](../adr/0008-bdf-import.md).

[bdf]: https://en.wikipedia.org/wiki/Glyph_Bitmap_Distribution_Format

## Loading

```cyrius
include "src/lib.cyr"

# From a file:
var id = kashi_load_bdf_file("/usr/share/fonts/X11/misc/terminus-bold-12.bdf");
if (id < 0) {
    # 0 - KASHI_EFORMAT / 0 - KASHI_EINVAL — malformed, unsupported, or unreadable
    return 1;
}

# Or from a buffer you already hold (e.g. an embedded asset):
var id2 = kashi_load_bdf(buf, len);
```

**Return convention**: success is a `font_id ≥ KASHI_RT_FONT_BASE`;
failure is a **negative** `0 - <code>` (`KASHI_EINVAL` for bad args /
unreadable file, `KASHI_EFORMAT` for malformed / unsupported /
truncated BDF). Always check `if (id < 0)`.

The runtime registry that holds BDF-loaded fonts is the same one PSF
populates — loaded ids are interleaved across formats in load order.

## Reading glyphs

BDF fonts plug into the same codepoint-addressed accessors as PSF:

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

For wide fonts (`width > 8`), use the byte-position accessor exactly as
the PSF guide describes — `kashi_rt_font_stride(id)` reports bytes/row;
`kashi_font_row_byte(id, cp, row, bi)` reads one byte of a wider row.
See [loading-psf-fonts.md](loading-psf-fonts.md#wide-glyph-fonts-050)
for the pattern (it's identical for BDF).

> **Addressing**: `kashi_font_row` / `kashi_font_ptr` resolve the
> codepoint through the BDF font's `ENCODING` map (built at load time).
> An unencoded codepoint returns `0` (blank), never a wild read.

## What kashi accepts (the BDF subset)

kashi's BDF parser handles the **console-font subset** — uniform-cell
bitmap fonts. Specifically:

- **`STARTFONT`** at the first non-blank line (any 2.x version).
- **`FONTBOUNDINGBOX w h xoff yoff`** with `1 ≤ w ≤ 32`, `1 ≤ h ≤ 32`.
- **`CHARS n`** with `1 ≤ n ≤ 65536`.
- A **`STARTPROPERTIES`/`ENDPROPERTIES`** block, if present, is walked
  past without parsing the body. Properties aren't needed for the
  bitmap data.
- One **`STARTCHAR..ENDCHAR`** block per glyph, each carrying
  `ENCODING <int>`, `BBX w h xoff yoff`, `BITMAP` + `h` hex-byte rows,
  `ENDCHAR`. Order within the block is flexible.
- Any unrecognized lines inside or outside glyph blocks (`FONT`,
  `SIZE`, `SWIDTH`, `DWIDTH`, `COMMENT`, `FONT_NAME`, …) are skipped.

### Strict BBX (uniform geometry)

Every glyph's `BBX w h xoff yoff` must match the font's
`FONTBOUNDINGBOX w h xoff yoff` exactly. A mismatch → `KASHI_EFORMAT`.
This covers the console-bitmap corpus (Terminus, Spleen, Tamzen,
Cozette, Unifont's 8×16 and 16×16 ranges). Display BDFs with variable
per-glyph geometry (italic overhangs, descender outlines) are out of
scope for 0.7.0; see [ADR 0008](../adr/0008-bdf-import.md) for the
rationale and the path to lifting the restriction.

### `ENCODING -1` glyphs are skipped

A BDF may carry named glyphs with no codepoint (`ENCODING -1`). kashi
drops these entirely — they're neither in the glyph store nor in the
codepoint→glyph map. The runtime font's `kashi_rt_font_count` reports
the **kept** count, which may be less than the declared `CHARS n`.

### File-size cap

`kashi_load_bdf_file` reads at most **4 MiB** of source
(`KASHI_BDF_FILE_CAP`). BDF is verbose text (~8–16× a PSF of the same
font), and 4 MiB covers every console BDF in the wild plus most
display BDFs. Files larger than the cap are rejected.

## Minimal example BDF

The smallest BDF kashi accepts:

```
STARTFONT 2.1
FONTBOUNDINGBOX 8 8 0 0
CHARS 1
STARTCHAR A
ENCODING 65
BBX 8 8 0 0
BITMAP
18
24
42
7E
42
42
42
00
ENDCHAR
ENDFONT
```

After `kashi_load_bdf` of the above:

```cyrius
kashi_font_row(id, 0x41, 0)   # -> 0x18
kashi_font_row(id, 0x41, 3)   # -> 0x7E
kashi_font_row(id, 0x41, 7)   # -> 0x00
kashi_font_row(id, 0x42, 0)   # -> 0 (unmapped — 'B' isn't in this font)
```

## Limits and posture

- **Width 1–32, height 1–32** (matching the PSF parser's caps).
- **Glyph count ≤ 65,536** (declared via `CHARS`).
- **Strict BBX** — see above.
- **Validation-first**: header keywords, geometry, glyph count, and
  each glyph's BBX are checked before the BITMAP hex is read. The
  parser (`src/font_bdf.cyr`) is heapless and dependency-free
  (`cyaudit vet` reports "no dependencies"), and is fuzzed in
  `tests/kashi.fcyr` (2000 rounds of random / templated /
  mutated-template / truncated-template inputs).
- Loaded fonts own a copy of their glyph bytes — the source buffer is
  free to reuse after loading. The runtime registry's bump allocator
  does not individually free per-font storage.

## Picking between PSF and BDF

| | PSF | BDF |
|---|---|---|
| **Format** | binary, packed | text, verbose |
| **Source size** | tight | ~10× of PSF for the same font |
| **Loaded via** | `kashi_load_psf*` | `kashi_load_bdf*` |
| **File cap** | 256 KiB | 4 MiB |
| **Unicode table** | PSF1 LE-u16 / PSF2 UTF-8 (separate table) | per-glyph `ENCODING <int>` lines |
| **Ligatures** | yes — `kashi_font_seq_glyph` (ADR 0005) | no |
| **Per-glyph BBX** | n/a (uniform) | strict-match required (ADR 0008) |
| **Use when** | shipping the font compiled-in or to a kernel | loading from an X11 source distribution |

If you have a font available in both formats, prefer PSF — smaller
files, slightly faster parse. Use BDF when the upstream only ships
source (Terminus distributes both; Unifont's primary form is BDF).
