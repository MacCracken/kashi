# Loading API (`src/lib.cyr`)

> **Frozen as of 0.9.0**. Library-face runtime font loading.
> Allocates from the stdlib bump allocator; not available to
> freestanding consumers.

## Return-value convention

All seven loading functions follow the same convention:

- **Success**: a `font_id` in the range `[KASHI_RT_FONT_BASE, …)`.
  `KASHI_RT_FONT_BASE` is `3` as of 0.9.0 (built-ins occupy 0, 1, 2).
  Future built-ins may bump this; consumers should reference the
  constant, not the value.
- **Failure**: `0 - <KashiResult>` — a non-positive value where the
  magnitude is one of `KASHI_EINVAL` (102) or `KASHI_EFORMAT` (103).
  Test with `if (id < 0)`. See [`codes.md`](codes.md).

```cyrius
var id = kashi_load_psf_file("/path/font.psf");
if (id < 0) {
    # 0 - KASHI_EINVAL = bad args / unreadable file
    # 0 - KASHI_EFORMAT = malformed / unsupported font
    return 1;
}
# id is a runtime font id; use it via the accessors.
```

## Format-specific load functions

Each `kashi_load_<fmt>` parses bytes already in memory; each
`kashi_load_<fmt>_file` reads a path into a capped buffer and
delegates to its in-memory sibling.

### `kashi_load_psf(buf, len)` + `kashi_load_psf_file(path)`

```cyrius
fn kashi_load_psf(buf, len);       # buf: ptr, len: i64 -> font_id or 0 - <code>
fn kashi_load_psf_file(path);      # path: cstr -> font_id or 0 - <code>
```

Loads a PSF1 or PSF2 (PC Screen Font) bitmap font. Validates magic,
header, geometry, glyph count, and total length; copies glyph bytes
into an owned store; builds the codepoint→glyph map from the inline
Unicode table if present (else identity-index fallback). Sequence
(ligature) entries are also harvested (see `kashi_font_seq_glyph`
in [`accessors.md`](accessors.md)). How the maps are built is under
[Codepoint maps](#codepoint-maps) below.

- **File-size cap**: `kashi_load_psf_file` takes a regular file of at
  most **256 KiB**; see [Reading font files](#reading-font-files).
- **Width**: 1..32 (PSF1 is always 8; PSF2 can be wider, per
  [ADR 0005](../adr/0005-wide-glyph-and-ligatures.md)).
- **Detailed walkthrough**: [`docs/guides/loading-psf-fonts.md`](../guides/loading-psf-fonts.md).

```cyrius
var id = kashi_load_psf_file("/usr/share/kbd/consolefonts/default8x16.psf");
if (id < 0) { return 1; }
# Render 'A' at row 7:
var bits = kashi_font_row(id, 0x41, 7);
```

### `kashi_load_bdf(buf, len)` + `kashi_load_bdf_file(path)`

```cyrius
fn kashi_load_bdf(buf, len);       # -> font_id or 0 - <code>
fn kashi_load_bdf_file(path);      # -> font_id or 0 - <code>
```

Loads a BDF (Bitmap Distribution Format) source font — X11's text
format. Parses `STARTFONT` / `FONTBOUNDINGBOX` / `CHARS` / per-glyph
`STARTCHAR..ENDCHAR` blocks. Strict-BBX policy: every glyph's `BBX`
must match the font-wide `FONTBOUNDINGBOX`
([ADR 0008](../adr/0008-bdf-import.md)). `ENCODING -1` glyphs are
skipped.

- **File-size cap**: `kashi_load_bdf_file` takes a regular file of at
  most **4 MiB** (`KASHI_BDF_FILE_CAP`). BDF is verbose text; this covers
  all console BDFs plus most display BDFs.
- **Strictness since 1.3.0** (audit 2026-10-30, B1 – B5): a font must end
  with `ENDFONT`, which catches a truncated file. A glyph's `BITMAP` rows
  must be followed directly by `ENDCHAR`. A number must end at a field
  boundary, so `ENCODING 0x41` is malformed, not U+0000. `STARTFONT`,
  `FONTBOUNDINGBOX` and `CHARS` may each appear once. Each of these used to
  load. The glyph store is sized from what the input can hold, not from
  `CHARS` alone.
- **Width / height**: 1..32 each.
- **Detailed walkthrough**: [`docs/guides/loading-bdf-fonts.md`](../guides/loading-bdf-fonts.md).

```cyrius
var id = kashi_load_bdf_file("/usr/share/fonts/X11/misc/terminus-12.bdf");
```

### `kashi_load_pcf(buf, len)` + `kashi_load_pcf_file(path)`

```cyrius
fn kashi_load_pcf(buf, len);       # -> font_id or 0 - <code>
fn kashi_load_pcf_file(path);      # -> font_id or 0 - <code>
```

Loads a PCF (Portable Compiled Format) bitmap font — X11's compiled
binary, what `bdftopcf` produces. Validates the TOC, locates the
three required tables (`PCF_METRICS`, `PCF_BITMAPS`,
`PCF_BDF_ENCODINGS`), validates uniform metrics, canonicalizes glyph
bitmaps to MSB-bit form with unpadded rows of `ceil(width / 8)` bytes,
regardless of source endianness or bit order ([ADR 0009](../adr/0009-pcf-import.md)).

- **File-size cap**: `kashi_load_pcf_file` takes a regular file of at
  most **4 MiB** (`KASHI_PCF_FILE_CAP`).
- **Width / height**: 1..32 each.
- **Required tables**: `PCF_METRICS`, `PCF_BITMAPS`,
  `PCF_BDF_ENCODINGS`. Other table types are walked past without
  parsing.
- **Detailed walkthrough**: [`docs/guides/loading-pcf-fonts.md`](../guides/loading-pcf-fonts.md).

```cyrius
var id = kashi_load_pcf_file("/usr/share/fonts/X11/misc/ter-u14n.pcf");
```

## Raw-table registration

### `kashi_register_font(width, height, glyph_data, glyph_count)`

```cyrius
fn kashi_register_font(width, height, glyph_data, glyph_count);
#   width: 1..32, height: 1..32, glyph_data: ptr to packed bytes,
#   glyph_count: 1..65,536
#   -> font_id or 0 - KASHI_EINVAL
```

Registers a raw glyph table directly — no file parsing. The bytes
at `glyph_data` are interpreted as `glyph_count` consecutive glyphs,
each `height * ceil(width/8)` bytes (row-major, top-to-bottom,
bit 7 of each byte = leftmost pixel). The bytes are **copied** into
a store the registry owns, so the caller may free / reuse
`glyph_data` after the call returns.

The resulting font is **identity-addressed**: codepoint `cp` resolves
to glyph index `cp` if `cp < glyph_count`, else unmapped. To attach
a codepoint→glyph mapping, see [`attach.md`](attach.md).

```cyrius
# Build a 2-glyph font directly from bytes.
var data = alloc(2 * 8);   # 2 glyphs, 8x8
# glyph 0: pattern
store8(data + 0, 0x18); store8(data + 1, 0x24);
store8(data + 2, 0x42); store8(data + 3, 0x7E);
# ... etc ...
var id = kashi_register_font(8, 8, data, 2);
if (id < 0) { return 1; }
```

Typical use: embedded fonts at compile time, dynamically-generated
glyphs (programmatic patterns), test fixtures.

## What's NOT in this surface

- `kashi_psf_parse` / `kashi_bdf_parse_header` / `kashi_pcf_parse_header`
  and friends — those are the low-level parser primitives. They're
  useful if you want to validate-without-loading, walk the font byte
  by byte, or implement a custom storage scheme on top of kashi's
  byte-level guarantees. See [`parsers.md`](parsers.md).
- `kashi_attach_unicode_*` — those go on after loading, in
  [`attach.md`](attach.md).


## Reading font files

The five `*_file` entry points (`kashi_load_psf_file`,
`kashi_load_bdf_file`, `kashi_load_pcf_file`,
`kashi_attach_unicode_table_file`, `kashi_attach_unicode_text_file`) read
the path the same way (1.3.0, [ADR 0012](../adr/0012-loader-hardening-maps-and-file-reading.md)):

| Path is… | Result |
|---|---|
| a regular file of 1 .. cap bytes | read whole into a buffer sized to it, then parsed |
| a regular file larger than the cap | `0 - KASHI_EINVAL` (until 1.3.0: cut at the cap and loaded) |
| an empty file | `0 - KASHI_EFORMAT` |
| missing / unreadable | `0 - KASHI_EINVAL`, nothing allocated |
| a FIFO, tty, device or directory (Linux) | `0 - KASHI_EINVAL` at once (until 1.3.0 a FIFO or tty blocked forever) |

- **Linux.** On x86_64 and aarch64 the path is opened non-blocking and
  without acquiring a controlling terminal, then checked through `fstat` on
  the open descriptor, so a path swapped between check and open cannot slip
  through. A read interrupted by a signal is retried.
- **Other targets.** On agnos, Windows and macOS the file is read through
  the stdlib with the same cap rule, but a special file can still block
  there.
- **Validation order.** The two attach calls validate the font id and kind
  before they open the path.

## Codepoint maps

Every codepoint map (PSF inline table, BDF, PCF, either attach format) is
built the same way since 1.3.0 ([ADR 0012](../adr/0012-loader-hardening-maps-and-file-reading.md)):

- **Cost.** O(n log n) in the number of entries on any input. A table
  already in codepoint order is checked and not re-sorted. Until 1.3.0 a
  table in reverse order cost O(n²): ~200 s for a 256 KiB PSF.
- **Duplicates.** When a table lists one codepoint for several glyphs, the
  **highest glyph index** wins: the Linux console's rule, where a later
  entry overwrites an earlier one. Until 1.3.0 the winner depended on the
  map's size.
- **Table without single codepoints.** A font that carries a table is
  addressed by it even when the table maps no single codepoint (only
  sequences): every codepoint then misses. Only a font with no table at all
  is identity-addressed. Until 1.3.0 such a font fell back to identity, and
  U+00E7 drew glyph 0xE7.
- **Allocation failure.** The record is registered complete, maps included.
  An allocation failure while building them fails the load with
  `0 - KASHI_EFORMAT` rather than registering a font without its map.

## Stability

- Function signatures and return conventions are frozen.
- `KASHI_RT_FONT_BASE` numeric value: currently `3`. A future cut
  that adds a fourth built-in font ID would bump this to `4`.
  Consumers must reference the constant, not literal `3`.
- File-size caps (`KASHI_BDF_FILE_CAP`, `KASHI_PCF_FILE_CAP`,
  the PSF 256 KiB ceiling) are part of the contract; lowering them
  in a future release would be a breaking change for any user near
  the limit, so they will only ever raise. Since 1.3.0 a file over its cap
  is refused, not truncated.
- The PSF file cap (256 KiB) is an internal constant in `lib.cyr`
  (the literal `262144` in `kashi_load_psf_file` / the
  `KASHI_TAB_FILE_CAP` value used by the attach paths). The same
  256 KiB value covers both. Not currently exported.
