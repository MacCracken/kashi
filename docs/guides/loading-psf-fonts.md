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
| `2` | built-in VGA 9×16, derived (`KASHI_FONT_VGA_9X16`; 0.5.1, ADR 0006) |
| `≥ 3` | runtime-loaded fonts, in load order (`KASHI_RT_FONT_BASE = 3`) |

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
    var bits = kashi_font_row(id, codepoint, row);  # one byte; bit 7 = leftmost
    # ... blit bits ...
    row = row + 1;
}
```

For the render hot path, resolve once and read rows directly — one codepoint
lookup per glyph instead of per row:

```cyrius
var p = kashi_font_ptr(id, codepoint);   # one lookup
if (p != 0) {
    var row = 0;
    while (row < rows) { blit(load8(p + row)); row = row + 1; }
}
```

> **Addressing**: `kashi_font_row` / `kashi_font_ptr` are **codepoint**-addressed.
> Built-in fonts take an ASCII codepoint (`0x20`–`0x7F`). A runtime font that
> carried a PSF Unicode table resolves the codepoint through that table; a
> runtime font *without* a table falls back to treating the codepoint as a
> raw glyph index. An unmapped codepoint (or out-of-range row) returns `0`
> (blank), never a wild read.
>
> For explicit raw glyph-**index** access into a runtime font (tooling,
> enumeration), use `kashi_rt_glyph_ptr(id, idx)` / `kashi_rt_glyph_row(id,
> idx, row)`.

## Wide-glyph fonts (0.5.0+)

For fonts wider than 8 pixels, each row spans multiple bytes
(`stride = ceil(width/8)`). Use the byte-position accessors:

```cyrius
var stride = kashi_rt_font_stride(id);   # bytes per row (built-ins: 1)
var row = 0;
while (row < rows) {
    var bi = 0;
    while (bi < stride) {
        var b = kashi_font_row_byte(id, codepoint, row, bi);  # codepoint-addressed
        # ... blit b at column (bi*8) ...
        bi = bi + 1;
    }
    row = row + 1;
}
```

`kashi_font_row(id, cp, row)` keeps returning a single byte — for wide
fonts that's the **leading byte** (`byte_idx 0`). Stride-1 fonts are
unchanged.

## Ligature lookup (0.5.0+)

PSF Unicode tables can record multi-codepoint→glyph sequences (e.g. an
"fi" ligature). `kashi_font_seq_glyph` resolves them:

```cyrius
var cps[16];                        # u64 per codepoint
store64(&cps + 0, 0x66);            # 'f'
store64(&cps + 8, 0x69);            # 'i'
var g = kashi_font_seq_glyph(id, &cps, 2);
if (g >= 0) {
    # render glyph index `g` via kashi_rt_glyph_row(id, g, row) ...
}
```

Returns `0 - 1` for no match, built-in ids, fonts without a sequence
table, null cps_ptr, or non-positive `len`. kashi exposes the data; the
shaping policy (when to look up a sequence vs render singles) belongs to
the consumer.

## Limits

- **Width 1–32** (up to 4 bytes per row); PSF2 wider than 32 → `KASHI_EFORMAT`.
- **Unicode sequences** are now harvested into the lookup table above;
  kashi itself does not do shaping — that stays the consumer's job.
- **Validation-first**: magic, version, header size, geometry, glyph count,
  and total length are all checked before any glyph byte is read. The parser
  (`src/font_psf.cyr`) is fuzzed in `tests/kashi.fcyr`.
- Loaded fonts own a copy of their glyph bytes, so you may free the source
  buffer after loading. Stores are not individually freed (bump allocator).
