# Loading PSF fonts at runtime

> **Last Updated**: 2026-05-27 (M1, 0.2.0)

The **library face** (`src/lib.cyr`) can import [PC Screen Font][psf] files
(PSF1 and PSF2) at runtime and serve their glyphs through the same accessor
shape as the built-in fonts. This is the stdlib-using path — a freestanding
kernel that only `include`s `src/font_data.cyr` does **not** get it (see
[ADR 0001](../adr/0001-freestanding-font-data-core.md)). The design is in
[ADR 0002](../adr/0002-runtime-font-registry.md).

[psf]: https://en.wikipedia.org/wiki/PC_Screen_Font

## Font ids

| id | font |
|---|---|
| `0` | built-in VGA 8×16 (`KASHI_FONT_VGA_8X16`) |
| `1` | built-in CGA 8×8 (`KASHI_FONT_CGA_8X8`) |
| `≥ 2` | runtime-loaded fonts, in load order (`KASHI_RT_FONT_BASE = 2`) |

## Loading

```cyrius
include "src/lib.cyr"

# From a file:
var id = kashi_load_psf_file("/usr/share/kbd/consolefonts/default8x16.psf");
if (id < 0) {
    # 0 - KASHI_EFORMAT / 0 - KASHI_EINVAL — malformed, unsupported, or unreadable
    return 1;
}

# Or from a buffer you already hold (e.g. an embedded asset):
var id2 = kashi_load_psf(buf, len);

# Or register a raw 8-wide glyph table directly (bytes are copied):
var id3 = kashi_register_font(8, 16, glyph_bytes, 256);
```

**Return convention**: success is a `font_id ≥ 2`; failure is a **negative**
`0 - <code>` (`KASHI_EINVAL` = bad args/unreadable, `KASHI_EFORMAT` =
malformed/unsupported/truncated). Always check `if (id < 0)`.

## Reading glyphs

Use the **unified accessors** — they work for built-in *and* runtime ids:

```cyrius
var rows = kashi_rt_font_height(id);     # glyph height (runtime fonts)
var w    = kashi_rt_font_width(id);      # always 8 in M1
var n    = kashi_rt_font_count(id);      # number of glyphs

var row = 0;
while (row < rows) {
    var bits = kashi_font_row(id, glyph_index, row);  # one byte; bit 7 = leftmost
    # ... blit bits ...
    row = row + 1;
}
```

> **Addressing (M1)**: for the **built-in** fonts the second argument is an
> ASCII codepoint (`0x20`–`0x7F`); for **runtime** fonts it is a raw glyph
> **index** `[0, count)`. The PSF Unicode table is validated but not yet
> mapped, so kashi does not (yet) translate codepoints to glyph indices for
> loaded fonts — that is a future milestone. Out-of-range glyph/row always
> returns `0` (blank), never a wild read.

## Limits (M1)

- **Width ≤ 8** (one byte per row). PSF1 is always 8 wide; a PSF2 font wider
  than 8 px is rejected with `KASHI_EFORMAT`.
- **Validation-first**: magic, version, header size, geometry, glyph count,
  and total length are all checked before any glyph byte is read. The parser
  (`src/font_psf.cyr`) is fuzzed in `tests/kashi.fcyr`.
- Loaded fonts own a copy of their glyph bytes, so you may free the source
  buffer after loading. Stores are not individually freed (bump allocator).
