# Result codes + constants reference

> **Frozen as of 0.9.0**. Numeric values for every exported enum
> in kashi. Values won't change in a future cut; new members may
> be added.

## Result codes

### `KashiResult` (`src/font_psf.cyr`)

The canonical result enum. Used by parsers (`KASHI_OK` is success;
the others are positive error codes; load functions return
`0 - <code>` to make errors negative).

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_OK` | `0` | Success. |
| `KASHI_ENOSYS` | `101` | Booked, not built (no current user). |
| `KASHI_EINVAL` | `102` | Bad arguments (null buf, negative len, invalid id, etc.). |
| `KASHI_EFORMAT` | `103` | Malformed / unsupported / truncated input. |

The base-100 numbering leaves room for non-overlapping
parser-specific codes in `KashiBdfRc` and `KashiPcfRc` (1, 2 in
each — these are local enums that don't collide because the BDF /
PCF parsers are fully self-contained and don't import
`KashiResult`).

### `KashiBdfRc` (`src/font_bdf.cyr`)

Used internally by the BDF parser; the library face's
`kashi_load_bdf*` translates these to the public `KashiResult`
form. Don't pass these to `kashi_load_*` — they're for direct
`kashi_bdf_parse_header` / `kashi_bdf_next_glyph` users.

| Constant | Value | Equivalent KashiResult |
|---|---|---|
| `KASHI_BDF_OK` | `0` | `KASHI_OK` |
| `KASHI_BDF_EINVAL` | `1` | `KASHI_EINVAL` (102) |
| `KASHI_BDF_EFORMAT` | `2` | `KASHI_EFORMAT` (103) |

### `KashiPcfRc` (`src/font_pcf.cyr`)

Same story for the PCF parser.

| Constant | Value | Equivalent KashiResult |
|---|---|---|
| `KASHI_PCF_OK` | `0` | `KASHI_OK` |
| `KASHI_PCF_EINVAL` | `1` | `KASHI_EINVAL` (102) |
| `KASHI_PCF_EFORMAT` | `2` | `KASHI_EFORMAT` (103) |

## Font IDs

### `KashiFontId` (`src/font_data.cyr`)

Built-in font ids.

| Constant | Value | Font |
|---|---|---|
| `KASHI_FONT_VGA_8X16` | `0` | IBM VGA BIOS 8×16. |
| `KASHI_FONT_CGA_8X8` | `1` | Dual-sourced 8×8 (AGNOS hand-drawn low half + Linux PD high half). |
| `KASHI_FONT_VGA_9X16` | `2` | VGA 9×16 derived from 8×16 + col-9 rule. |

### `KashiRtBase` (`src/lib.cyr`)

Runtime-font ID base.

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_RT_FONT_BASE` | `3` | First runtime font_id. Lower values are built-ins; loaded fonts get `≥ 3`, assigned in load order. |

## Codepoint range

### `KashiGlyphRange` (`src/font_data.cyr`)

The built-in fonts' codepoint coverage.

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_GLYPH_FIRST` | `0x20` (32) | First encoded codepoint (space). |
| `KASHI_GLYPH_LAST` | `0xFF` (255) | Last encoded codepoint (last CP437 cell). |
| `KASHI_GLYPH_COUNT` | `224` | `LAST - FIRST + 1`. |

## PSF constants

### `KashiPsfConst` (`src/font_psf.cyr`)

PSF1 / PSF2 magic bytes, mode/flag bits, validation bounds.

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_PSF1_MAGIC0` | `0x36` | PSF1 magic byte 0. |
| `KASHI_PSF1_MAGIC1` | `0x04` | PSF1 magic byte 1. |
| `KASHI_PSF1_MODE512` | `1` | PSF1 mode bit 0: 512-glyph bank. |
| `KASHI_PSF1_MODEHASTAB` | `2` | PSF1 mode bit 1: has Unicode table. |
| `KASHI_PSF1_MODESEQ` | `4` | PSF1 mode bit 2: table has sequences (implies HASTAB). |
| `KASHI_PSF1_HDR` | `4` | PSF1 header byte length. |
| `KASHI_PSF2_MAGIC` | `0x864AB572` | PSF2 magic (LE u32). |
| `KASHI_PSF2_HDR_MIN` | `32` | PSF2 fixed header size. |
| `KASHI_PSF2_HAS_UNICODE` | `1` | PSF2 flags bit 0: has Unicode table. |
| `KASHI_PSF_MAX_HEIGHT` | `32` | Max glyph height (pixels). |
| `KASHI_PSF_MAX_WIDTH` | `32` | Max glyph width (pixels). |
| `KASHI_PSF_MAX_COUNT` | `65536` | Max glyph count. |
| `KASHI_PSF_KIND_PSF1` | `1` | Parsed-header KIND value for PSF1. |
| `KASHI_PSF_KIND_PSF2` | `2` | Parsed-header KIND value for PSF2. |

### `KashiPsfTok` (`src/font_psf.cyr`)

Unicode-table walk token kinds + struct offsets.

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_TOK_CP` | `0` | Single codepoint. |
| `KASHI_TOK_SEQ` | `1` | Sequence separator (0xFFFE PSF1 / 0xFE PSF2). |
| `KASHI_TOK_TERM` | `2` | Glyph terminator (0xFFFF PSF1 / 0xFF PSF2). |
| `KASHI_TOK_END` | `3` | Past end or malformed — caller should stop. |
| `KASHI_UT_TYPE` | `0` | Token struct: byte offset of type field. |
| `KASHI_UT_CP` | `8` | Token struct: byte offset of codepoint field. |
| `KASHI_UT_NEXT` | `16` | Token struct: byte offset of next-position field. |

### `KashiPsfHdr` (`src/font_psf.cyr`)

Parsed-header struct byte offsets (56 bytes total).

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_PH_KIND` | `0` | Kind (1=PSF1, 2=PSF2, 3=BDF, 4=PCF). |
| `KASHI_PH_WIDTH` | `8` | Width in pixels. |
| `KASHI_PH_HEIGHT` | `16` | Height in pixels. |
| `KASHI_PH_COUNT` | `24` | Glyph count. |
| `KASHI_PH_CHARSIZE` | `32` | Bytes per glyph. |
| `KASHI_PH_DATAOFF` | `40` | Byte offset of glyph data. |
| `KASHI_PH_UNIOFF` | `48` | Byte offset of Unicode table (0 if absent). |

## BDF constants

### `KashiBdfConst` (`src/font_bdf.cyr`)

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_BDF_KIND` | `3` | Parsed-header KIND value for BDF. |
| `KASHI_BDF_MAX_WIDTH` | `32` | Max glyph width. |
| `KASHI_BDF_MAX_HEIGHT` | `32` | Max glyph height. |
| `KASHI_BDF_MAX_COUNT` | `65536` | Max glyph count. |
| `KASHI_BDF_FILE_CAP` | `4194304` | `kashi_load_bdf_file` cap (4 MiB). |
| `KASHI_BDF_INT_MAX_DIGITS` | `10` | Per-field digit cap (oversize → KASHI_BDF_EFORMAT, audit F3). |

### `KashiBdfHdr` (`src/font_bdf.cyr`)

Same shape and values as `KashiPsfHdr` — re-declared so
`font_bdf.cyr` stays self-contained.

| Constant | Value |
|---|---|
| `BDF_PH_KIND` | `0` |
| `BDF_PH_WIDTH` | `8` |
| `BDF_PH_HEIGHT` | `16` |
| `BDF_PH_COUNT` | `24` |
| `BDF_PH_CHARSIZE` | `32` |
| `BDF_PH_DATAOFF` | `40` |
| `BDF_PH_UNIOFF` | `48` |

### `KashiBdfCtx` (`src/font_bdf.cyr`)

BDF context struct (32 bytes). Holds the parsed `FONTBOUNDINGBOX`
for per-glyph BBX validation.

| Constant | Value | Meaning |
|---|---|---|
| `BDF_CTX_W` | `0` | FONTBOUNDINGBOX width. |
| `BDF_CTX_H` | `8` | FONTBOUNDINGBOX height. |
| `BDF_CTX_X` | `16` | FONTBOUNDINGBOX xoff (i64, signed). |
| `BDF_CTX_Y` | `24` | FONTBOUNDINGBOX yoff (i64, signed). |

### `KashiBdfGOut` (`src/font_bdf.cyr`)

Glyph-out struct from `kashi_bdf_next_glyph` (24 bytes).

| Constant | Value | Meaning |
|---|---|---|
| `BG_KIND` | `0` | Result kind. |
| `BG_CP` | `8` | Codepoint (valid when KIND = OK). |
| `BG_NEXT` | `16` | Byte offset to resume scanning. |
| `BG_KIND_OK` | `0` | Bitmap written to dest; CP holds the codepoint. |
| `BG_KIND_SKIP` | `1` | `ENCODING -1` or out-of-range cp; caller advances NEXT. |
| `BG_KIND_END` | `2` | Reached ENDFONT or buffer end. |

## PCF constants

### `KashiPcfConst` (`src/font_pcf.cyr`)

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_PCF_KIND` | `4` | Parsed-header KIND value for PCF. |
| `KASHI_PCF_MAX_WIDTH` | `32` | Max glyph width. |
| `KASHI_PCF_MAX_HEIGHT` | `32` | Max glyph height. |
| `KASHI_PCF_MAX_COUNT` | `65536` | Max glyph count. |
| `KASHI_PCF_MAX_TABLES` | `64` | Max TOC entries. |
| `KASHI_PCF_FILE_CAP` | `4194304` | `kashi_load_pcf_file` cap (4 MiB). |
| `KASHI_PCF_MAGIC0` | `0x01` | PCF magic byte 0. |
| `KASHI_PCF_MAGIC1` | `0x66` | PCF magic byte 1 (`'f'`). |
| `KASHI_PCF_MAGIC2` | `0x63` | PCF magic byte 2 (`'c'`). |
| `KASHI_PCF_MAGIC3` | `0x70` | PCF magic byte 3 (`'p'`). |
| `KASHI_PCF_HDR` | `8` | Header byte length (magic + table_count). |
| `KASHI_PCF_TOC_STRIDE` | `16` | Bytes per TOC entry. |

### `KashiPcfType` (`src/font_pcf.cyr`)

PCF table type bit-flags.

| Constant | Value | Status |
|---|---|---|
| `PCF_TYPE_PROPERTIES` | `0x01` | Ignored. |
| `PCF_TYPE_ACCELERATORS` | `0x02` | Ignored. |
| `PCF_TYPE_METRICS` | `0x04` | **Required**. |
| `PCF_TYPE_BITMAPS` | `0x08` | **Required**. |
| `PCF_TYPE_INK_METRICS` | `0x10` | Ignored. |
| `PCF_TYPE_BDF_ENCODINGS` | `0x20` | **Required**. |
| `PCF_TYPE_SWIDTHS` | `0x40` | Ignored. |
| `PCF_TYPE_GLYPH_NAMES` | `0x80` | Ignored. |
| `PCF_TYPE_BDF_ACCELERATORS` | `0x100` | Ignored. |

### `KashiPcfFmt` (`src/font_pcf.cyr`)

Format-word bit masks.

| Constant | Value | Meaning |
|---|---|---|
| `PCF_FMT_GLYPH_PAD_MASK` | `0x03` | Bits 0–1: glyph row padding (0=1B, 1=2B, 2=4B, 3=8B). |
| `PCF_FMT_BYTE_ORDER` | `0x04` | Bit 2: byte order (0=LSB, 1=MSB). |
| `PCF_FMT_BIT_ORDER` | `0x08` | Bit 3: bit order (0=LSB, 1=MSB). |
| `PCF_FMT_SCAN_UNIT_MASK` | `0x30` | Bits 4–5: scan unit (0=1B, 1=2B, 2=4B, 3=8B). |
| `PCF_FMT_COMPRESSED` | `0x100` | Bit 8: METRICS table compressed (5 bytes/glyph). |
| `PCF_FMT_VALID_METRICS` | `0x13F` | All valid bits for METRICS (0–5 + 8). |
| `PCF_FMT_VALID_OTHER` | `0x3F` | All valid bits for BITMAPS / ENCODINGS (0–5). |

### `KashiPcfHdr` (`src/font_pcf.cyr`)

Parsed-header struct offsets — same values as `KashiPsfHdr`.

| Constant | Value |
|---|---|
| `PCF_PH_KIND` | `0` |
| `PCF_PH_WIDTH` | `8` |
| `PCF_PH_HEIGHT` | `16` |
| `PCF_PH_COUNT` | `24` |
| `PCF_PH_CHARSIZE` | `32` |
| `PCF_PH_DATAOFF` | `40` |
| `PCF_PH_UNIOFF` | `48` |

### `KashiPcfCtx` (`src/font_pcf.cyr`)

PCF context struct (144 bytes; caller allocates `var ctx[144]`).

| Constant | Value | Meaning |
|---|---|---|
| `PCF_CTX_GLYPH_COUNT` | `0` | Glyph count. |
| `PCF_CTX_WIDTH` | `8` | Width in pixels. |
| `PCF_CTX_HEIGHT` | `16` | Height in pixels. |
| `PCF_CTX_PCF_ROW_BYTES` | `24` | Bytes per source row (depends on glyph_pad). |
| `PCF_CTX_OUR_STRIDE` | `32` | Bytes per canonical row = `ceil(width/8)`. |
| `PCF_CTX_BMP_OFFS` | `40` | Offset of `glyph_offsets[]` within buf. |
| `PCF_CTX_BMP_DATA` | `48` | Offset of bitmap pixel data within buf. |
| `PCF_CTX_BMP_END` | `56` | One-past-end of bitmap pixel data. |
| `PCF_CTX_BMP_BE` | `64` | 1 if BITMAPS table is big-endian. |
| `PCF_CTX_BMP_BIT_LSB` | `72` | 1 if BITMAPS bit order is LSB. |
| `PCF_CTX_BMP_SCAN` | `80` | scan_unit (1, 2, 4, or 8). |
| `PCF_CTX_ENC_DATA` | `88` | Offset of `glyph_indices[]` within buf. |
| `PCF_CTX_ENC_BE` | `96` | 1 if BDF_ENCODINGS table is big-endian. |
| `PCF_CTX_MIN_B1` | `104` | min byte 1 of encoded range. |
| `PCF_CTX_MAX_B1` | `112` | max byte 1. |
| `PCF_CTX_MIN_B2` | `120` | min byte 2. |
| `PCF_CTX_MAX_B2` | `128` | max byte 2. |

## Runtime registry

### `KashiRtRec` (`src/lib.cyr`)

The runtime-font record byte offsets (88 bytes per font). **Internal
layout** — these constants are exported for tooling but consumers
should never write to the record directly; use the documented
accessors instead. Documented here for completeness.

| Constant | Value | Meaning |
|---|---|---|
| `RT_WIDTH` | `0` | |
| `RT_HEIGHT` | `8` | |
| `RT_COUNT` | `16` | |
| `RT_CHARSIZE` | `24` | |
| `RT_DATA` | `32` | Ptr to owned glyph bytes. |
| `RT_UMAP` | `40` | Ptr to sorted (cp, glyph) pairs, 0 if none. |
| `RT_UCOUNT` | `48` | Number of cp→glyph pairs. |
| `RT_STRIDE` | `56` | Bytes per row. |
| `RT_SEQS` | `64` | Ptr to sequence records, 0 if none. |
| `RT_SEQCOUNT` | `72` | Sequence count. |
| `RT_SEQDATA` | `80` | Ptr to flat codepoint backing store. |

### `KashiTabCap` (`src/lib.cyr`)

| Constant | Value | Meaning |
|---|---|---|
| `KASHI_TAB_FILE_CAP` | `262144` | `kashi_attach_unicode_*_file` read cap (256 KiB). |
