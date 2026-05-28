# Sidecar Unicode-table attach API (`src/lib.cyr`)

> **Frozen as of 0.9.0**. Attach an externally-supplied
> codepoint→glyph mapping to an already-loaded runtime font. Added
> in 0.7.2 ([ADR 0010](../adr/0010-psf-u-variant.md)); the
> CVE-2015-1803 stale-state issue was closed in 0.8.0
> ([audit F9](../audit/2026-05-28-audit-0.8.0.md#f9--kashi_attach_unicode_table-left-font-with-no-map-on-failed-attach--medium--fixed)).

The use case: you have a tableless PSF (loaded via
`kashi_load_psf`), or a font you've registered via
`kashi_register_font`, and you want codepoint addressing to work
without rebuilding from scratch.

## Replace semantics + atomic failure

All four attach functions:

- **Replace** any existing umap / seqs on the runtime font record.
  The previous arrays leak conceptually; the bump allocator
  reclaims at process exit.
- **Are atomic on failure**: a returned negative code implies "no
  observable state change" — if the new table produces no entries
  or the input is malformed, the previous map is restored
  (audit F9, 0.8.0).
- **Reject built-in font IDs** (< `KASHI_RT_FONT_BASE`) with
  `KASHI_EINVAL`. Built-ins use the freestanding core's ASCII
  codepoint scheme directly; attach is runtime-only.

Return convention: `KASHI_OK` (0) on success, `0 - <KashiResult>`
on failure (`KASHI_EINVAL` = 102 for bad args; `KASHI_EFORMAT` =
103 for malformed input).

## Binary attach (raw PSF table bytes)

### `kashi_attach_unicode_table(font_id, buf, len, kind)`

```cyrius
fn kashi_attach_unicode_table(font_id, buf, len, kind);
#   font_id: runtime font id (>= KASHI_RT_FONT_BASE)
#   buf:     ptr to raw PSF Unicode-table bytes
#   len:     length of buf in bytes
#   kind:    KASHI_PSF_KIND_PSF1 (LE-u16 entries) or KASHI_PSF_KIND_PSF2 (UTF-8)
#   -> KASHI_OK or 0 - <KashiResult>
```

Attaches a raw PSF Unicode-table byte stream — the same byte
format `kashi_psf_uni_token` already walks for inline tables. The
internal `_kashi_build_umap` and `_kashi_build_useqs` helpers are
reused with `unioff = 0` (whole buffer is the table).

```cyrius
# Load a tableless PSF (mode = 0, no inline Unicode table).
var id = kashi_load_psf(psf_buf, psf_len);
if (id < 0) { return 1; }

# Attach a PSF1-format table: 256 entries, each "cp, 0xFFFF" pairs
# of LE-u16 — i.e., one cp per glyph followed by 0xFFFF terminator.
var tab = build_my_psf1_table();
var tab_len = ...;
if (kashi_attach_unicode_table(id, tab, tab_len, KASHI_PSF_KIND_PSF1) != KASHI_OK) {
    # The font's previous map (if any) is unchanged.
    return 2;
}
# Codepoint addressing now works:
var bits = kashi_font_row(id, 0x0041, 0);
```

### `kashi_attach_unicode_table_file(font_id, path, kind)`

```cyrius
fn kashi_attach_unicode_table_file(font_id, path, kind);
#   path: file path (cstr)
#   -> KASHI_OK or 0 - <KashiResult>
```

File companion: reads at most `KASHI_TAB_FILE_CAP` (256 KiB) and
delegates to `kashi_attach_unicode_table`. Returns `0 - KASHI_EINVAL`
on a read error (missing / permission denied / oversized) and
`0 - KASHI_EFORMAT` on an empty file.

```cyrius
kashi_attach_unicode_table_file(id, "/path/to/font.tab", KASHI_PSF_KIND_PSF1);
```

## Text attach (`psfgettable` format)

### `kashi_attach_unicode_text(font_id, buf, len)`

```cyrius
fn kashi_attach_unicode_text(font_id, buf, len);
#   buf: ptr to UTF-8 / ASCII text
#   len: length of buf in bytes
#   -> KASHI_OK or 0 - <KashiResult>
```

Parses the [`psfgettable` convention](https://linux.die.net/man/8/psfgettable):
one line per glyph, `0x<hex_idx>` then one or more `U+<hex_cp>`
codepoints, optional `#` line comment.

```
# Comments start with `#`.
0x0020   U+0020
0x0041   U+0041 U+0391    # alternates: ASCII 'A' or Greek Alpha
0xFFFD   U+FFFD U+FFFC    # multiple cps may share a glyph
```

Rules:

- Multiple codepoints on a line all map to the same glyph index.
- Out-of-range glyph indices (≥ font's count) are silently
  skipped (matches the binary path's `if (glyph < count)` guard).
- The text format does **not** support multi-codepoint sequences
  (ligatures). For those, use the binary path with a PSF2 table
  containing `0xFE` separators.
- No continuation-line support (the rare `psfaddtable` indented-
  continuation form). Repeat the `0x<idx>` if you need multiple
  lines for the same glyph.

```cyrius
var tab = "0x0041   U+0041 U+0391\n0x0042   U+0042\n";
var tab_len = 35;
kashi_attach_unicode_text(id, tab, tab_len);
```

### `kashi_attach_unicode_text_file(font_id, path)`

```cyrius
fn kashi_attach_unicode_text_file(font_id, path);
#   -> KASHI_OK or 0 - <KashiResult>
```

File companion: reads at most `KASHI_TAB_FILE_CAP` (256 KiB) and
delegates to `kashi_attach_unicode_text`.

```cyrius
kashi_attach_unicode_text_file(id, "/path/to/font.tab");
```

## Picking a format

| | Binary | Text |
|---|---|---|
| **Bytes per mapping** | 2 (PSF1) or 1–4 (PSF2 UTF-8) | ~10–20 (`0xHHHH<tab>U+HHHH\n`) |
| **Ligature support** | Yes (via `0xFFFE` / `0xFE` separators) | No |
| **Source readability** | Opaque bytes | Plain text |
| **Use when** | Round-tripping inline PSF tables; ligature-bearing fonts | Hand-written mappings; tooling output |

If you have a `.tab` file that was produced by `psfgettable`, use the
text path. If you've extracted the raw Unicode-table portion from a
PSF file (or are generating mappings programmatically and want
maximum density), use the binary path.

## Stability

- Function signatures and replace semantics are frozen.
- The text format is the `psfgettable` convention; future cuts may
  add continuation-line support or sequence-line support, but
  existing text files will continue to parse.
- `KASHI_TAB_FILE_CAP` (256 KiB) will only ever raise, not lower.
