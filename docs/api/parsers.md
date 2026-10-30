# Parser API (`src/font_psf.cyr`, `src/font_bdf.cyr`, `src/font_pcf.cyr`)

> **Frozen as of 0.9.0**. Low-level heapless parser primitives for
> advanced consumers — validate without loading, walk a font byte
> by byte, build a custom storage scheme on top of kashi's
> byte-level guarantees.

Each parser file is **dependency-free** (`cyrius vet` reports
"no dependencies"): only the load / store intrinsics + arithmetic (the
stores write the caller's header / context struct).
No allocations. Bounds-safe on any input, hostile or otherwise.

Most consumers should use the higher-level `kashi_load_*` API in
[`loading.md`](loading.md). This page is for callers who need
finer control.

## Shared struct shapes

### Parsed-header struct (56 bytes)

Used by PSF, BDF, and PCF parsers. Caller allocates `var hdr[56]`.

| Offset | Field | Type | Meaning |
|---|---|---|---|
| 0 | `KASHI_PH_KIND` | u64 | 1 = PSF1, 2 = PSF2, 3 = BDF, 4 = PCF. |
| 8 | `KASHI_PH_WIDTH` | u64 | Pixel width per glyph (1..32). |
| 16 | `KASHI_PH_HEIGHT` | u64 | Pixel height per glyph (1..32). |
| 24 | `KASHI_PH_COUNT` | u64 | Number of glyphs. |
| 32 | `KASHI_PH_CHARSIZE` | u64 | Bytes per glyph = `height * ceil(width/8)`. |
| 40 | `KASHI_PH_DATAOFF` | u64 | Byte offset of glyph data (or first STARTCHAR / bitmap pixel data) within the source buffer. |
| 48 | `KASHI_PH_UNIOFF` | u64 | Byte offset of the Unicode table (PSF), else 0. |

Each parser writes the same shape; the `KIND` field distinguishes
which family parsed it.

PSF defines the constants as `KASHI_PH_*` in `font_psf.cyr`. BDF
re-declares them as `BDF_PH_*` in `font_bdf.cyr` (same offsets, same
shape — duplicated to keep `cyrius vet` reporting "no dependencies"
for each parser). PCF uses `PCF_PH_*`. The values match across all
three; you can pick whichever spelling matches the parser you're
calling.

## PSF parser

### `kashi_psf_parse(buf, len, out)`

```cyrius
fn kashi_psf_parse(buf, len, out);   # -> KASHI_OK / KASHI_EINVAL / KASHI_EFORMAT
```

Detects and validates a PSF1 or PSF2 font in `buf` (`len` bytes).
On success fills the 56-byte `out` header struct. Returns:

- `KASHI_OK` (0) — valid PSF1 or PSF2.
- `KASHI_EINVAL` (102) — `buf == 0`, `out == 0`, `len < 0`.
- `KASHI_EFORMAT` (103) — bad magic, unsupported version,
  invalid header fields, geometry out of bounds, truncated, or
  unknown header bits set (audit F1/F2).

Does **not** read glyph data — only the fixed header — so safe to
call on any buffer, including hostile ones.

```cyrius
var hdr[56];
var rc = kashi_psf_parse(buf, len, &hdr);
if (rc == KASHI_OK) {
    var kind = load64(&hdr + KASHI_PH_KIND);     # 1 or 2
    var width = load64(&hdr + KASHI_PH_WIDTH);
    var height = load64(&hdr + KASHI_PH_HEIGHT);
    var count = load64(&hdr + KASHI_PH_COUNT);
    # ... glyph data at buf + KASHI_PH_DATAOFF, Unicode table at buf + KASHI_PH_UNIOFF if nonzero ...
}
```

### `kashi_psf_uni_token(buf, pos, end, kind, out)`

```cyrius
fn kashi_psf_uni_token(buf, pos, end, kind, out);   # -> 0
```

Decodes one token from a PSF Unicode table at byte offset `pos`
into the 24-byte `out` token struct. Caller-allocated:
`var tok[24]`.

Token struct:

| Offset | Field | Meaning |
|---|---|---|
| 0 | `KASHI_UT_TYPE` | One of `KASHI_TOK_CP` (single codepoint), `KASHI_TOK_SEQ` (sequence separator), `KASHI_TOK_TERM` (glyph terminator), `KASHI_TOK_END` (past `end` or malformed — caller should stop). |
| 8 | `KASHI_UT_CP` | Codepoint value (valid only when type = `KASHI_TOK_CP`). |
| 16 | `KASHI_UT_NEXT` | Byte offset of the next token. Caller advances `pos = next`. |

`kind` is `KASHI_PSF_KIND_PSF1` (LE-u16 entries; `0xFFFF` = TERM,
`0xFFFE` = SEQ) or `KASHI_PSF_KIND_PSF2` (UTF-8 codepoints; `0xFF`
= TERM, `0xFE` = SEQ).

Always bounds-safe — at/over `end`, or on a malformed/truncated
unit, returns `KASHI_TOK_END` with `next = end` so the caller's
walk terminates rather than reading past the buffer. Pure: only
`load8` + arithmetic, writing only `out`. Since 1.3.0 a null `out` returns
without writing, and a null `buf` reports `KASHI_TOK_END` (audit
2026-10-30, P2).

**Strict UTF-8 (PSF2)**: rejects codepoints > U+10FFFF, the
UTF-16 surrogate range U+D800..U+DFFF (audit F2, 0.6.0), and
overlong sequences per RFC 3629 (ADR 0010, 0.7.2). **PSF1** u16 entries in
U+D800..U+DFFF end the table too since 1.3.0 (audit 2026-10-30, P1).

```cyrius
# Walk a PSF Unicode table glyph by glyph.
var tok[24];
var pos = unioff;
var glyph = 0;
while (pos < len) {
    kashi_psf_uni_token(buf, pos, len, kind, &tok);
    var t = load64(&tok + KASHI_UT_TYPE);
    if (t == KASHI_TOK_END) { pos = len; }
    if (t != KASHI_TOK_END) {
        pos = load64(&tok + KASHI_UT_NEXT);
        if (t == KASHI_TOK_CP) {
            var cp = load64(&tok + KASHI_UT_CP);
            # cp maps to `glyph` ...
        }
        if (t == KASHI_TOK_TERM) { glyph = glyph + 1; }
    }
}
```

## BDF parser

### `kashi_bdf_parse_header(buf, len, hdr, ctx)`

```cyrius
fn kashi_bdf_parse_header(buf, len, hdr, ctx);
#   hdr: ptr to 56-byte parsed-header struct
#   ctx: ptr to 32-byte BDF context struct
#   -> KASHI_BDF_OK / KASHI_BDF_EINVAL / KASHI_BDF_EFORMAT
```

Validates `STARTFONT` / `FONTBOUNDINGBOX` / `CHARS` (each at most once
since 1.3.0, and every number must end at a field boundary: audit
2026-10-30, B4 / B5), skips `STARTPROPERTIES..ENDPROPERTIES` if present,
locates the first `STARTCHAR`. Fills both structs:

- `hdr` (56 bytes): `KIND = 3`, `WIDTH`, `HEIGHT`, declared `COUNT`,
  `CHARSIZE`, `DATAOFF` (offset of first STARTCHAR), `UNIOFF = 0`
  (BDF carries no separate Unicode table).
- `ctx` (32 bytes): `BDF_CTX_W`, `BDF_CTX_H`, `BDF_CTX_X`, `BDF_CTX_Y`
  — the parsed `FONTBOUNDINGBOX` for per-glyph BBX validation by
  `kashi_bdf_next_glyph`.

```cyrius
var hdr[56];
var ctx[32];
if (kashi_bdf_parse_header(buf, len, &hdr, &ctx) != KASHI_BDF_OK) {
    return malformed;
}
var w = load64(&hdr + BDF_PH_WIDTH);
var h = load64(&hdr + BDF_PH_HEIGHT);
# ... iterate glyphs via kashi_bdf_next_glyph from DATAOFF ...
```

### `kashi_bdf_next_glyph(buf, pos, end, ctx, dest, dest_size, gout)`

```cyrius
fn kashi_bdf_next_glyph(buf, pos, end, ctx, dest, dest_size, gout);
#   pos: cursor byte offset (initially = hdr's DATAOFF)
#   end: buffer length
#   ctx: 32-byte BDF context (from parse_header)
#   dest: per-glyph bitmap buffer (>= height * ceil(width/8) bytes)
#   dest_size: capacity of dest
#   gout: 24-byte glyph-out struct
#   -> KASHI_BDF_OK / KASHI_BDF_EINVAL / KASHI_BDF_EFORMAT
```

Cursor: from `pos`, scans to the next `STARTCHAR`, reads `ENCODING`
+ `BBX` (strict-match validated against `ctx`) + `BITMAP` hex rows,
decodes into `dest`, advances past `ENDCHAR`. After the declared rows only
`ENDCHAR` may follow; an extra row is `KASHI_BDF_EFORMAT` (1.3.0, B3). A
`ctx` whose width or height is outside 1..32 (a zeroed ctx from a failed
parse, or a forged one) is `KASHI_BDF_EINVAL` (1.3.0, B6). Writes the
result kind, codepoint, and next-position into `gout`:

| Offset | Field | Meaning |
|---|---|---|
| 0 | `BG_KIND` | `BG_KIND_OK` (0) = bitmap in dest, cp in CP. `BG_KIND_SKIP` (1) = `ENCODING -1` or out-of-range cp, skip. `BG_KIND_END` (2) = reached `ENDFONT` or buffer end (the cursor serves partial buffers; `kashi_load_bdf` additionally requires an `ENDFONT` line since 1.3.0, B1). |
| 8 | `BG_CP` | Codepoint (valid when KIND = OK). |
| 16 | `BG_NEXT` | Byte offset to resume scanning. |

```cyrius
var gout[24];
var dest[128];
var pos = load64(&hdr + BDF_PH_DATAOFF);
var done = 0;
while (done == 0) {
    var grc = kashi_bdf_next_glyph(buf, pos, len, &ctx, &dest, 128, &gout);
    if (grc != KASHI_BDF_OK) { done = 1; }
    if (done == 0) {
        var kind = load64(&gout + BG_KIND);
        if (kind == BG_KIND_END) { done = 1; }
        if (done == 0) {
            pos = load64(&gout + BG_NEXT);
            if (kind == BG_KIND_OK) {
                var cp = load64(&gout + BG_CP);
                # dest now holds glyph bytes for codepoint cp ...
            }
        }
    }
}
```

## PCF parser

### `kashi_pcf_parse_header(buf, len, hdr, ctx)`

```cyrius
fn kashi_pcf_parse_header(buf, len, hdr, ctx);
#   hdr: ptr to 56-byte parsed-header struct
#   ctx: ptr to 144-byte PCF context struct
#   -> KASHI_PCF_OK / KASHI_PCF_EINVAL / KASHI_PCF_EFORMAT
```

Validates magic + table count, walks the TOC, locates the three
required tables (`PCF_METRICS`, `PCF_BITMAPS`, `PCF_BDF_ENCODINGS`),
validates METRICS uniformity, computes width/height/glyph_count.

Fills:

- `hdr` (56 bytes): `KIND = 4`, `WIDTH`, `HEIGHT`, `COUNT`,
  `CHARSIZE`, `DATAOFF` (= bitmap pixel data offset, diagnostic),
  `UNIOFF = 0`.
- `ctx` (144 bytes): glyph_count, dims, source/target row strides,
  table offsets, format flags, BDF_ENCODINGS range. The struct
  fields are documented in `font_pcf.cyr` as `PCF_CTX_*`.

Rejects: bad magic, table_count > 64, missing required tables,
duplicate TOC types (audit F8), non-uniform metrics, unknown
format-byte bits (audit F5/F6/F7), out-of-range
width/height/glyph_count. On any failure the ctx's glyph count is 0
(since 1.3.0, audit 2026-10-30 F1), so `kashi_pcf_decode_glyph` on it
returns `KASHI_PCF_EINVAL` and `kashi_pcf_cp_to_idx` returns `0 - 1`
instead of reading with half-set fields.

```cyrius
var hdr[56];
var ctx[144];
if (kashi_pcf_parse_header(buf, len, &hdr, &ctx) != KASHI_PCF_OK) {
    return malformed;
}
var count = load64(&hdr + PCF_PH_COUNT);
# Now iterate glyphs via kashi_pcf_decode_glyph ...
```

### `kashi_pcf_decode_glyph(buf, ctx, glyph_idx, dest, dest_size)`

```cyrius
fn kashi_pcf_decode_glyph(buf, ctx, glyph_idx, dest, dest_size);
#   glyph_idx: 0..glyph_count-1
#   dest: buffer for canonical glyph bytes
#   dest_size: >= height * ceil(width/8)
#   -> KASHI_PCF_OK / KASHI_PCF_EINVAL / KASHI_PCF_EFORMAT
```

Decodes the glyph at raw index `glyph_idx` into `dest`. Applies
scan-unit byte swap (if `byte_order != bit_order`) + bit-reverse
(if `bit_order` is LSB) to produce canonical MSB-bit output with
unpadded rows of `ceil(width / 8)` bytes (kashi's standard convention: byte 0 = leftmost 8 px, bit 7
= leftmost pixel).

```cyrius
var dest[128];
var g = 0;
while (g < count) {
    if (kashi_pcf_decode_glyph(buf, &ctx, g, &dest, 128) != KASHI_PCF_OK) {
        return malformed;
    }
    # dest holds canonical glyph bytes for index g ...
    g = g + 1;
}
```

### `kashi_pcf_cp_to_idx(buf, ctx, cp)`

```cyrius
fn kashi_pcf_cp_to_idx(buf, ctx, cp);   # -> glyph_idx (>= 0) or 0 - 1
```

Resolves a codepoint to a glyph index via the BDF_ENCODINGS table.
Returns the glyph index (`0 ≤ idx < glyph_count`) or `0 - 1` if
unmapped (out of the encoded range, or the table cell holds the
sentinel `0xFFFF`).

A PCF encodes two bytes per codepoint, so any `cp` above U+FFFF returns
`0 - 1`. Before 1.2.2 such a `cp` wrapped into the BMP: `0x10041` resolved
as U+0041
([issue](../development/issues/2026-10-29-pcf-cp-to-idx-wraps-above-ffff.md)).

```cyrius
var idx = kashi_pcf_cp_to_idx(buf, &ctx, 0x41);
if (idx >= 0) {
    # Glyph for 'A' is at raw index `idx`.
    kashi_pcf_decode_glyph(buf, &ctx, idx, &dest, 128);
}
```

## Stability

- All function signatures and the struct shapes (56-byte parsed
  header, 32-byte BDF ctx, 144-byte PCF ctx, 24-byte BDF gout,
  24-byte PSF uni-token struct) are frozen.
- Field offset constants (`KASHI_PH_*`, `KASHI_UT_*`, `BDF_PH_*`,
  `BDF_CTX_*`, `BG_*`, `PCF_PH_*`, `PCF_CTX_*`) are frozen.
- New `KASHI_PH_KIND` values may be added for future format parsers
  (e.g., `KIND = 5` for a hypothetical OTB import). Existing values
  (1, 2, 3, 4) won't be renumbered.
- Strict-validation behavior (the audit-F* tightenings) is part of
  the contract: a future cut won't loosen any of the rejections.

## Heapless guarantee

Each parser file is *separately* dependency-free — none includes
the others, none includes the stdlib. This means:

- Embedding kashi's PSF or BDF or PCF parser in a non-kashi build
  is feasible by `include`-ing only the one parser file.
- `cyrius vet src/font_psf.cyr` → "no dependencies".
- `cyrius vet src/font_bdf.cyr` → "no dependencies".
- `cyrius vet src/font_pcf.cyr` → "no dependencies".

The freestanding boundary ([ADR 0001](../adr/0001-freestanding-font-data-core.md))
extends to the parsers in this sense, even though they're nominally
"library face" by their stdlib-touching consumers (lib.cyr does the
allocations).
