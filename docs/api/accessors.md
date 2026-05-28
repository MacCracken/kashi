# Accessor API (`src/lib.cyr`)

> **Frozen as of 0.9.0**. Library-face read accessors that span
> built-in (ids 0,1,2) and runtime (ids ≥ `KASHI_RT_FONT_BASE`)
> fonts through one set of entry points.

All accessors are **bounds-safe for the full `i64` input domain**.
Out-of-range inputs return safe sentinels (0 / null), never a wild
deref.

## Unified codepoint-addressed read

These three functions are kashi's main read surface. They route
built-in ids to the freestanding core and runtime ids to the
library-face registry. Codepoints resolve via the font's
codepoint→glyph map (built at load time) or fall back to identity
for fonts loaded without a map (e.g., raw `kashi_register_font`).

### `kashi_font_ptr(font_id, cp)`

```cyrius
fn kashi_font_ptr(font_id, cp);   # -> ptr or 0 (null)
```

Base pointer to glyph row bytes for codepoint `cp` in `font_id`.
Returns `0` for unknown font, unmapped codepoint, or any other
out-of-range input.

This is the **hot-path resolve** — call once per glyph, then read
rows directly via `load8(ptr + row * stride + byte_idx)`.

```cyrius
var p = kashi_font_ptr(id, codepoint);
if (p != 0) {
    var rows = kashi_rt_font_height(id);
    var stride = kashi_rt_font_stride(id);   # built-in: kashi_font_stride
    var row = 0;
    while (row < rows) {
        var b = load8(p + row * stride);   # leading byte of the row
        # blit ...
        row = row + 1;
    }
}
```

### `kashi_font_row(font_id, cp, row)`

```cyrius
fn kashi_font_row(font_id, cp, row);   # -> u8 (0..255)
```

One row byte of the glyph at codepoint `cp` in `font_id`. For wide
fonts (stride > 1), returns the **leading byte** (byte 0) of the
row; use `kashi_font_row_byte` for the other bytes.

```cyrius
var bits = kashi_font_row(id, 0x41, 7);   # 'A' row 7
```

Convenient for non-hot-path use; the hot path should resolve once
via `kashi_font_ptr`.

### `kashi_font_row_byte(font_id, cp, row, byte_idx)`

```cyrius
fn kashi_font_row_byte(font_id, cp, row, byte_idx);   # -> u8
```

One byte at column position `byte_idx` within row `row`. `byte_idx
0` is the leftmost 8 px (bit 7 = leftmost pixel); `byte_idx 1` is
pixels 8–15; etc. For stride-1 fonts only `byte_idx 0` is
meaningful — other indices return 0.

Added in 0.5.0 ([ADR 0005](../adr/0005-wide-glyph-and-ligatures.md)).

```cyrius
# Render a wide-glyph row.
var stride = kashi_rt_font_stride(id);
var bi = 0;
while (bi < stride) {
    var b = kashi_font_row_byte(id, cp, row, bi);
    # blit `b` at column (bi * 8) ...
    bi = bi + 1;
}
```

## Raw glyph-index access (runtime fonts only)

For tooling, enumeration, table-less fonts, or PCF-style "render
glyph N" access — bypasses the codepoint map.

### `kashi_rt_glyph_ptr(font_id, idx)`

```cyrius
fn kashi_rt_glyph_ptr(font_id, idx);   # -> ptr or 0
```

Base pointer to runtime font `font_id`'s glyph at raw index `idx`
(`0 ≤ idx < kashi_rt_font_count(font_id)`). Returns 0 for unknown
runtime font or out-of-range index.

**Built-in fonts have no separate index space** — call
`kashi_glyph_ptr` (from [`core.md`](core.md)) for those.

### `kashi_rt_glyph_row(font_id, idx, row)`

```cyrius
fn kashi_rt_glyph_row(font_id, idx, row);   # -> u8 (0..255)
```

One row byte of runtime font glyph at raw index `idx`. Leading byte
for wide fonts.

### `kashi_rt_glyph_row_byte(font_id, idx, row, byte_idx)`

```cyrius
fn kashi_rt_glyph_row_byte(font_id, idx, row, byte_idx);   # -> u8
```

One byte at column position `byte_idx` of runtime font glyph `idx`.

## Runtime-font metadata

All return `0` for an unknown `font_id` (incl. when a built-in id
is passed — these are runtime-only). For built-ins use the
`kashi_font_*` accessors in [`core.md`](core.md).

### `kashi_rt_font_width(font_id)`

Pixel width per glyph (1..32).

### `kashi_rt_font_height(font_id)`

Pixel height per glyph = rows (1..32).

### `kashi_rt_font_count(font_id)`

Number of glyphs in the runtime font's store. For BDF-loaded fonts,
this counts only the glyphs kept (post-`ENCODING -1` skipping).

### `kashi_rt_font_stride(font_id)`

Bytes per row = `ceil(width / 8)`. 1 for 8-wide; 2 for 9..16; etc.

## Cross-id helpers

### `kashi_font_total()`

```cyrius
fn kashi_font_total();   # -> total number of known fonts
```

`KASHI_RT_FONT_BASE + len(runtime_registry)`. Built-in count is
currently 3 (ids 0, 1, 2). Use to iterate all fonts:

```cyrius
var total = kashi_font_total();
var id = 0;
while (id < total) {
    # font with id `id` is loaded ...
    id = id + 1;
}
```

### `kashi_set_active_font(font_id)` / `kashi_active_font()`

```cyrius
fn kashi_set_active_font(font_id);   # -> KASHI_OK or 0 - KASHI_EINVAL
fn kashi_active_font();               # -> current active font_id (default 0)
```

Stash the consumer's "current font" selection in kashi rather than
plumbing it through every call site. Purely additive — the
accessors never consult this; the consumer reads it and passes the
id along. Default `0` (the VGA 8×16 built-in).

```cyrius
kashi_set_active_font(id);
# ... later, in a render callback ...
var f = kashi_active_font();
var bits = kashi_font_row(f, cp, row);
```

Bounds-checked: setting a font_id outside `[0, kashi_font_total())`
returns `0 - KASHI_EINVAL`.

## Ligature lookup (PSF only)

### `kashi_font_seq_glyph(font_id, cps_ptr, len)`

```cyrius
fn kashi_font_seq_glyph(font_id, cps_ptr, len);
#   font_id: runtime font (>= KASHI_RT_FONT_BASE)
#   cps_ptr: pointer to a u64-per-codepoint array
#   len: number of codepoints in the array (>= 1)
#   -> glyph_idx (>= 0) on match, 0 - 1 on miss
```

PSF Unicode tables may record multi-codepoint→single-glyph
mappings (e.g., an "fi" ligature: `f, i` → ligature glyph). This
function looks them up. Linear scan over the font's sequence list
(typically few per font, so no hot-path concern).

Returns `0 - 1` for any miss: no match, built-in id, font without
a sequence table (BDF / PCF / raw-registered fonts), null
`cps_ptr`, or non-positive `len`.

Added in 0.5.0 ([ADR 0005](../adr/0005-wide-glyph-and-ligatures.md)).

```cyrius
var cps[16];
store64(&cps + 0, 0x66);    # 'f'
store64(&cps + 8, 0x69);    # 'i'
var g = kashi_font_seq_glyph(id, &cps, 2);
if (g >= 0) {
    # Render glyph index `g` via kashi_rt_glyph_row(id, g, row).
}
```

kashi exposes the *data* for ligature lookup; the *shaping policy*
(when to look up sequences vs render singles) belongs to the
consumer.

## Stability

- All function signatures and return conventions are frozen.
- The codepoint-vs-index distinction is part of the contract:
  built-in fonts and runtime fonts with a map use codepoints;
  raw-registered or table-less fonts use identity-index. A future
  cut may add an explicit "codepoint-or-index" probe API; existing
  semantics won't change.
- The "bit 7 = leftmost pixel" convention is frozen across all
  read accessors.

## Performance notes

See [`docs/benchmarks.md`](../benchmarks.md). At a glance (x86_64 0.8.0):

- Hot path (`kashi_font_ptr` then `load8`): ~7 ns + ~1 ns/row.
- `kashi_font_row` (built-in): ~20 ns/call.
- `kashi_font_row` (runtime, no map): ~43 ns/call.
- `kashi_font_row` (runtime, with codepoint map): ~65 ns/call.

The runtime overhead is the registry lookup + codepoint resolve;
amortize via the hot-path pattern in render loops.
