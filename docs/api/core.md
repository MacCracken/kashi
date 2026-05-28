# Freestanding core API (`src/font_data.cyr`)

> **Frozen as of 0.9.0**. Consumed by the agnos kernel via
> `[deps.kashi] modules=["src/font_data.cyr"]`. Uses only
> `store8`/`load8` intrinsics + arithmetic — no stdlib, no heap,
> no syscalls. `cyaudit vet src/font_data.cyr` reports
> "no dependencies".

## Built-in font IDs

The freestanding core carries three built-in fonts, addressable by id:

| Constant | Value | Font |
|---|---|---|
| `KASHI_FONT_VGA_8X16` | `0` | IBM VGA BIOS 8×16 ROM font (224 glyphs, full CP437; PD source = Linux's `font_8x16.c`). |
| `KASHI_FONT_CGA_8X8` | `1` | Hand-drawn AGNOS-original 8×8 ASCII low half (`0x20..0x7F`) + Linux PD `font_8x8.c` high half (`0x80..0xFF`). 224 glyphs. |
| `KASHI_FONT_VGA_9X16` | `2` | VGA 9×16 derived at init from `KASHI_FONT_VGA_8X16` + the VGA col-9 replication rule for `0xC0..0xDF` ([ADR 0006](../adr/0006-vga-9x16-derived-builtin.md)). No new byte tables; computed at boot. |

All three cover the codepoint range `0x20..0xFF` (printable ASCII +
CP437 high half). Runtime-loaded fonts get ids ≥ `KASHI_RT_FONT_BASE`
(see [`codes.md`](codes.md)).

## Codepoint range constants

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_GLYPH_FIRST` | `0x20` | First encoded codepoint (space). |
| `KASHI_GLYPH_LAST` | `0xFF` | Last encoded codepoint (last CP437 cell). |
| `KASHI_GLYPH_COUNT` | `224` | `KASHI_GLYPH_LAST - KASHI_GLYPH_FIRST + 1`. |

A built-in font has glyph data for every codepoint in this range.
Codepoints outside the range return safe sentinels from the accessors
(row `0`, ptr `0`).

## Init

### `kashi_font_init()`

```cyrius
fn kashi_font_init();
```

Packs the built-in glyph tables into BSS and derives the VGA 9×16
table from the 8×16 table. **Must run once at boot**, before the
first call to any accessor. Idempotent — calling twice is safe and
produces byte-identical results.

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
| `font_id` | `i64` | Font id (built-in 0..2; runtime ≥ 3). Unknown → returns 0. |
| `ch` | `i64` | Codepoint. Out-of-range → 0. |
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

**Performance**: ~17 ns/call (see
[`docs/benchmarks.md`](../benchmarks.md)). For a hot render path,
prefer the `kashi_glyph_ptr` + `load8` pattern (~7 ns per row).

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

Base pointer to glyph `ch`'s row bytes. The `i`th row is at
`load8(ptr + i * stride)`. Returns `0` for unknown font or
out-of-range codepoint.

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
~17 ns/call through `kashi_glyph_row`.

### `kashi_glyph_encoded(ch)`

```cyrius
fn kashi_glyph_encoded(ch);   # -> 1 if ch is in [FIRST, LAST], else 0
```

Whether codepoint `ch` has an encoded glyph in the built-in range.
Pure predicate; doesn't read glyph data.

```cyrius
if (kashi_glyph_encoded(ch) == 0) {
    # Render a replacement (e.g., 0x3F '?').
    ch = 0x3F;
}
```

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
fn kashi_font_count(font_id);   # -> number of encoded glyphs
```

Number of encoded glyphs (`KASHI_GLYPH_COUNT` for built-ins).

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
- **Constants** (font IDs, range bounds): values are frozen. A
  future cut may add new font IDs (e.g., `KASHI_FONT_VGA_10X20 = 3`)
  but won't renumber existing ones.
- **`KASHI_GLYPH_FIRST` / `KASHI_GLYPH_LAST`**: a future cut may
  widen the range (e.g., to U+0000..U+FFFF) but won't narrow it.
  Consumers reading the constants are forward-safe; consumers that
  hard-coded `0x20`/`0xFF` are off in that case.
- **Byte order in glyph rows** (bit 7 = leftmost pixel) is frozen.

The freestanding core does NOT use the `_kashi_` underscore prefix
convention used by the library face's internal helpers — every
function in `font_data.cyr` is part of the public surface above
EXCEPT `kashi_fset8` / `kashi_fset16`, which are init-time table
packers and should not be called externally.
