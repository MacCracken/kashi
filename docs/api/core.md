# Freestanding core API (`src/font_data.cyr`)

> **Frozen as of 0.9.0**; extended additively in 1.1.0
> (`kashi_cp437_from_unicode` + the extension cells —
> [ADR 0011](../adr/0011-unicode-to-cp437-and-extension-glyphs.md)).
> Consumed by the agnos kernel via
> `[deps.kashi] modules=["src/font_data.cyr"]`. Uses only
> `store8`/`load8` intrinsics + arithmetic — no stdlib, no heap,
> no syscalls. `cyrius vet src/font_data.cyr` reports
> "no dependencies".

## Built-in font IDs

The freestanding core carries three built-in fonts, addressable by id:

| Constant | Value | Font |
|---|---|---|
| `KASHI_FONT_VGA_8X16` | `0` | IBM VGA BIOS 8×16 ROM font (224 CP437 glyphs + the 2 extension cells; PD source = Linux's `font_8x16.c`). |
| `KASHI_FONT_CGA_8X8` | `1` | Hand-drawn AGNOS-original 8×8 ASCII low half (`0x20..0x7F`) + Linux PD `font_8x8.c` high half (`0x80..0xFF`). 224 CP437 glyphs + the 2 extension cells. |
| `KASHI_FONT_VGA_9X16` | `2` | VGA 9×16 derived at init from `KASHI_FONT_VGA_8X16` + the VGA col-9 replication rule for `0xC0..0xDF` ([ADR 0006](../adr/0006-vga-9x16-derived-builtin.md)). No new byte tables; computed at boot. |

All three cover the CP437 cells `0x20..0xFF` (printable ASCII +
CP437 high half) and, since 1.1.0, the two extension cells after them.
Runtime-loaded fonts get ids ≥ `KASHI_RT_FONT_BASE`
(see [`codes.md`](codes.md)).

## Cells vs codepoints

Every per-glyph accessor in this file takes a **cell**, not a Unicode
codepoint:

| Cells | What they are |
|---|---|
| `0x20..0xFF` | The CP437 code page — the bytes agnos and every byte-indexed consumer render. **Cells `0x80..0xFF` are not Latin-1**: cell `0xE7` is 'τ'; U+00E7 'ç' is cell `0x87`. This range never changes meaning. |
| `0x100..0x101` | The **extension block** (1.1.0): glyphs CP437 has no cell for — `0x100` U+2026 '…', `0x101` U+FFFD '�'. No byte value names them. |

Unicode text reaches a cell through
[`kashi_cp437_from_unicode(cp)`](#kashi_cp437_from_unicodecp), which answers
`0` — a cell every accessor treats as out of range — when no built-in glyph
exists. Treat the extension cell numbers as kashi-internal: get them from
`kashi_cp437_from_unicode`, or enumerate `KASHI_EXT_FIRST..KASHI_EXT_LAST`;
don't spell `0x100`.

## Cell range constants

The CP437 range (unchanged since 0.4.0 — these never describe the extension):

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_GLYPH_FIRST` | `0x20` | First encoded codepoint (space). |
| `KASHI_GLYPH_LAST` | `0xFF` | Last encoded codepoint (last CP437 cell). |
| `KASHI_GLYPH_COUNT` | `224` | `KASHI_GLYPH_LAST - KASHI_GLYPH_FIRST + 1`. |

The extension block (`KashiExtRange`, added in 1.1.0):

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_EXT_FIRST` | `0x100` | First extension cell; `== KASHI_GLYPH_LAST + 1`, so storage is contiguous. |
| `KASHI_EXT_LAST` | `0x101` | Last extension cell. |
| `KASHI_EXT_COUNT` | `2` | `KASHI_EXT_LAST - KASHI_EXT_FIRST + 1`. |

A built-in font has glyph data for every cell in both ranges. Cells
outside them return safe sentinels from the accessors (row `0`, ptr `0`).

## Init

### `kashi_font_init()`

```cyrius
fn kashi_font_init();
```

Packs the built-in glyph tables into their (zero-filled) buffers, derives the VGA 9×16
table from the 8×16 table, and builds the 128-entry Unicode reverse
index `kashi_cp437_from_unicode` searches. **Must run once at boot**,
before the first call to any accessor. Idempotent — calling twice is
safe and produces byte-identical results: every store writes a final
value to a fixed address (the index source is pre-sorted; nothing is
reset and refilled), so a reader racing a *re*-init never sees a wrong
value. The *first* call must finish before another CPU reads — kashi
has no synchronisation of its own.

**Cost**: ~21 µs per call warm (1.2.1, cyrius 6.7.18; the process's
first call costs ~30 – 38 µs, cold caches; 1.1.0: +1.2 µs over 1.0.10 for the extension
glyphs and the index fill; `font_init` in
[`docs/benchmarks.md`](../benchmarks.md)). It is boot work: a consumer
that calls it on every draw pays that per draw — check
`kashi_font_is_ready()` first instead.

For freestanding consumers: this is the only init step. No
allocations, no syscalls.

```cyrius
# Boot init (do this once):
kashi_font_init();

# Then accessors are ready:
var bits = kashi_glyph_row(KASHI_FONT_VGA_8X16, 0x41, 7);  # 'A' row 7
```

### `kashi_font_is_ready()`

```cyrius
fn kashi_font_is_ready();   # -> 1 if init has run, 0 otherwise
```

Reads back the init-complete flag. Useful for a freestanding
consumer to assert `kashi_font_init` ran before the first render.

```cyrius
if (kashi_font_is_ready() == 0) {
    kashi_font_init();
}
```

## Per-glyph accessors

All accessors are **bounds-safe for the full `i64` input domain**.
Out-of-range `(font_id, ch, row, byte_idx)` returns safe sentinels:
- Functions returning a byte → `0` (blank row).
- Functions returning a pointer → `0` (null).
- Metadata functions → `0` (unknown font).

Never a wild deref. Audit-verified across the freestanding surface
([`docs/audit/2026-05-27-audit.md`](../audit/2026-05-27-audit.md)).

### `kashi_glyph_row(font_id, ch, row)`

```cyrius
fn kashi_glyph_row(font_id, ch, row);   # -> u8 (0..255)
```

One row byte of glyph `ch` in `font_id`. For 8-wide fonts, bit 7
is the leftmost pixel. For wider fonts (e.g. `KASHI_FONT_VGA_9X16`,
which is 9 wide / stride 2), this returns the **leading byte**
(byte 0) of the row — use `kashi_glyph_row_byte` for the other
bytes.

| Param | Type | Range / meaning |
|---|---|---|
| `font_id` | `i64` | Built-in font id, 0..2 only. Any other id, runtime ids included, returns 0; read runtime fonts through `kashi_font_row` / `kashi_rt_glyph_row`. |
| `ch` | `i64` | Cell (`0x20..0xFF` CP437, `0x100..0x101` extension). Out-of-range → 0. |
| `row` | `i64` | Row index 0..height-1. Out-of-range → 0. |

```cyrius
# Render 'A' from VGA 8x16, row by row.
var row = 0;
while (row < kashi_font_height(KASHI_FONT_VGA_8X16)) {
    var bits = kashi_glyph_row(KASHI_FONT_VGA_8X16, 0x41, row);
    # blit `bits` (bit 7 = leftmost pixel) at (col, row + base_y) ...
    row = row + 1;
}
```

**Performance**: ~18 ns/call (see
[`docs/benchmarks.md`](../benchmarks.md)). For a hot render path,
prefer the `kashi_glyph_ptr` + `load8` pattern (~7 ns per glyph +
~1 ns per row).

### `kashi_glyph_row_byte(font_id, ch, row, byte_idx)`

```cyrius
fn kashi_glyph_row_byte(font_id, ch, row, byte_idx);  # -> u8
```

One byte at column-byte position `byte_idx` within a row. `byte_idx
0` is the leftmost 8 px; `byte_idx 1` is pixels 8–15; etc. Bit 7 of
each byte is the leftmost pixel of those 8.

For stride-1 fonts (`kashi_font_stride` returns 1) only `byte_idx 0`
is meaningful; other indices return 0.

```cyrius
# Render a wide font row, byte by byte.
var stride = kashi_font_stride(KASHI_FONT_VGA_9X16);
var bi = 0;
while (bi < stride) {
    var b = kashi_glyph_row_byte(KASHI_FONT_VGA_9X16, 0xDB, 5, bi);
    # blit `b` at column (bi * 8) ...
    bi = bi + 1;
}
```

### `kashi_glyph_ptr(font_id, ch)`

```cyrius
fn kashi_glyph_ptr(font_id, ch);   # -> ptr or 0 (null)
```

Base pointer to glyph `ch`'s row bytes (`ch` a cell). The `i`th row
is at `load8(ptr + i * stride)`. Returns `0` for unknown font or
out-of-range cell.

```cyrius
# Resolve once, blit all rows in a tight loop.
var p = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, 0x41);
if (p != 0) {
    var row = 0;
    while (row < 16) {
        var bits = load8(p + row);   # stride 1 for VGA 8x16
        # blit `bits` ...
        row = row + 1;
    }
}
```

**This is the hot-path pattern**: one `kashi_glyph_ptr` call per
glyph, then direct `load8` reads per row. ~7 ns + ~1 ns/row vs
~18 ns/call through `kashi_glyph_row`.

### `kashi_glyph_encoded(ch)`

```cyrius
fn kashi_glyph_encoded(ch);   # -> 1 if ch is in [FIRST, LAST] or [EXT_FIRST, EXT_LAST], else 0
```

Whether cell `ch` has a built-in glyph: the CP437 range or (since
1.1.0) the extension block. Pure predicate; doesn't read glyph data.
Every per-glyph accessor gates on it.

```cyrius
if (kashi_glyph_encoded(ch) == 0) {
    # Render a replacement (e.g., 0x3F '?').
    ch = 0x3F;
}
```

1.1.0 changed its answer for exactly two inputs: `0x100` and `0x101`
were out of range and are now the extension cells. Its answer for
every value `≤ 0xFF` is unchanged.

### `kashi_cp437_from_unicode(cp)`

```cyrius
fn kashi_cp437_from_unicode(cp);   # -> cell for kashi_glyph_*, or 0 (no built-in glyph)
```

Added in 1.1.0 ([ADR 0011](../adr/0011-unicode-to-cp437-and-extension-glyphs.md)).
The cell that draws Unicode codepoint `cp` in the built-in fonts:

| `cp` | Result |
|---|---|
| U+0020..U+007E | itself |
| the 128 codepoints of the CP437 high half | `0x80..0xFF`, per the standard CP437 mapping (Python's `cp437` codec / the Unicode Consortium's `CP437.TXT`): U+00E7 'ç' → `0x87`, U+2500 '─' → `0xC4`, U+00A0 NBSP → `0xFF` |
| U+2302 HOUSE '⌂' | `0x7F` (the glyph kashi's VGA cell `0x7F` draws; the CGA 8×8 draws that cell blank) |
| U+007F DELETE | `0` — a control character, like U+0000..U+001F and U+0080..U+009F |
| U+2026 HORIZONTAL ELLIPSIS | `KASHI_EXT_FIRST` |
| U+FFFD REPLACEMENT CHARACTER | `KASHI_EXT_FIRST + 1` |
| anything else (incl. negatives, past U+10FFFF) | `0` |

Exactly 226 codepoints map. Font-independent — it names a cell, and
every built-in font has every cell. `0` renders blank / null through
every accessor; substitute `kashi_cp437_from_unicode(0xFFFD)` if you
want a visible mark instead. Allocation-free: an ASCII fast path, then
a binary search (≤ 8 probes) of the reverse index `kashi_font_init`
builds. **Call `kashi_font_init` first** — before it only ASCII,
U+2302 and the extension resolve, and the high half answers `0`.

```cyrius
# Draw a decoded Unicode scalar `u` (your UTF-8 decoder's output).
var cell = kashi_cp437_from_unicode(u);
if (cell == 0) { cell = kashi_cp437_from_unicode(0xFFFD); }   # optional: show '�'
var p = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, cell);           # resolve once per glyph
# ... load8(p + row) per row, as above ...

# Truncate with a real ellipsis:
var ell = kashi_cp437_from_unicode(0x2026);
```

**Performance**: ~4 ns for ASCII; a high-half codepoint costs
~9 – 33 ns by how deep the binary search goes (1 – 8 probes over the
128-entry index; the bench's ~25 ns is one 6-probe hit, and a miss inside
U+00A0..U+25A0 costs ~31 – 35 ns) — see
[`docs/benchmarks.md`](../benchmarks.md). Once per glyph, not per row.
The search is bounded by the index's 128 slots whatever the count global
holds (1.3.0, audit 2026-10-30 C1).

## Per-font metadata

All return `0` for an unknown `font_id`.

### `kashi_font_width(font_id)`

```cyrius
fn kashi_font_width(font_id);   # -> pixel width per glyph
```

Pixel width per glyph. Built-ins: 8 (VGA 8×16, CGA 8×8) or 9 (VGA 9×16).

### `kashi_font_height(font_id)`

```cyrius
fn kashi_font_height(font_id);   # -> pixel height per glyph (= rows)
```

Pixel height per glyph = number of rows.

### `kashi_font_first(font_id)`

```cyrius
fn kashi_font_first(font_id);   # -> first encoded codepoint (== KASHI_GLYPH_FIRST)
```

First encoded codepoint. All built-ins return `KASHI_GLYPH_FIRST`
(`0x20`); included for symmetry with future fonts that might cover a
different range.

### `kashi_font_count(font_id)`

```cyrius
fn kashi_font_count(font_id);   # -> number of CP437 glyphs
```

Number of CP437 glyphs (`KASHI_GLYPH_COUNT`, 224, for built-ins), so
`first .. first + count` enumerates the CP437 range. The extension
block is **not** counted — enumerate `KASHI_EXT_FIRST..KASHI_EXT_LAST`
separately (as `docs/examples/glyph_sheet.cyr` does).

### `kashi_font_stride(font_id)`

```cyrius
fn kashi_font_stride(font_id);   # -> bytes per row (= ceil(width/8))
```

Bytes per row. Returns 1 for 8-wide fonts, 2 for 9–16 wide, etc.
Added in 0.5.0 ([ADR 0005](../adr/0005-wide-glyph-and-ligatures.md)).

## Stability

The freestanding-core surface is the most rigid in kashi — it's the
agnos kernel's load-bearing dependency. Specifically:

- **Function signatures, parameter order, return semantics** are
  frozen.
- **Constants** (font IDs, range bounds): values are frozen — with
  one stated exception, the extension block's upper end (next bullet).
  A future cut may add new font IDs (e.g., `KASHI_FONT_VGA_10X20 = 3`)
  but won't renumber existing ones.
- **`KASHI_GLYPH_FIRST` / `KASHI_GLYPH_LAST` / `KASHI_GLYPH_COUNT`**
  describe the CP437 cells and stay `0x20` / `0xFF` / `224`. The cell
  space grows past them only through the extension block (1.1.0): a
  later minor may append cells after the current last one, raising
  `KASHI_EXT_LAST` / `KASHI_EXT_COUNT` (`KASHI_EXT_FIRST` stays
  `0x100`) — read the constants, don't hard-code `0x101`. Existing
  extension cells keep their glyphs. Unicode reaches cells only through
  `kashi_cp437_from_unicode` — the core does not become
  codepoint-addressed, because `0x80..0xFF` are CP437 cells, not
  Latin-1.
- **Glyph bytes for `0x20..0xFF`** are frozen (the glyph-sheet pin's
  first 672 lines).
- **Byte order in glyph rows** (bit 7 = leftmost pixel) is frozen.

The freestanding core does NOT use the `_kashi_` underscore prefix
convention used by the library face's internal helpers — every
function in `font_data.cyr` is part of the public surface above
EXCEPT `kashi_fset8` / `kashi_fset16`, which are init-time table
packers and should not be called externally. The 1.1.0 index helpers
(`_kashi_cp437_ent`, `_kashi_cp437_rev8`) do carry the prefix and are
internal, as are the `kashi_cp437_rev` / `kashi_cp437_n` globals.
