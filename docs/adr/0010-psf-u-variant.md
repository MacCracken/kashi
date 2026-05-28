# 0010 — PSF u-variant: sidecar Unicode tables + strict UTF-8

**Status**: Accepted
**Date**: 2026-05-28
**Refines / extends**: [ADR 0003](0003-codepoint-addressing-runtime-fonts.md)
(codepoint→glyph map for runtime fonts), the 0.6.0 audit F2 tightening
(reject codepoints > U+10FFFF and the UTF-16 surrogate range).

## Context

The 0.7.x track closes with the "PSF u-variant" item from the
roadmap: edge cases around PSF Unicode tables that the
0.2.0/0.3.0/0.5.0 PSF work left untouched.

There are two real loose ends:

1. **Sidecar Unicode tables.** Linux's `psfgettable` / `psfaddtable`
   convention pairs a `.psf` file *without* an inline Unicode table
   with a separate file describing the codepoint→glyph mapping. Today
   kashi can load the tableless PSF but the result is unaddressable
   by codepoint (the unified accessors fall back to identity-index
   per ADR 0003) — useless for any real consumer that wants to render
   text by codepoint. There's no kashi API to attach an
   externally-supplied table to an already-loaded font.

2. **Overlong-UTF-8 encodings.** The 0.6.0 audit F2 tightening
   stopped accepting codepoints > U+10FFFF and the UTF-16 surrogate
   range, but did not reject **overlong** sequences — a codepoint
   encoded with more UTF-8 bytes than necessary (e.g., U+0041
   encoded as `0xC1 0x81` instead of `0x41`). RFC 3629 declares
   overlong sequences invalid for security reasons (canonical
   filtering bypass). The decoder accepts them today.

The roadmap also mentioned "larger table-walk caps" as a 0.7.2
item, but on closer inspection the existing walk is buffer-bounded
with no infinite-loop risk: every `kashi_psf_uni_token` call either
returns `KASHI_TOK_END` (which the outer walks treat as a stop) or
advances `pos` by at least 1 byte. There's nothing concrete to fix;
this item is dropped.

## Decision

### Four attach APIs in `src/lib.cyr`

```
kashi_attach_unicode_table(font_id, buf, len, kind)
kashi_attach_unicode_table_file(font_id, path, kind)
kashi_attach_unicode_text(font_id, buf, len)
kashi_attach_unicode_text_file(font_id, path)
```

The **binary** pair takes raw PSF1 (LE-u16 entries) or PSF2 (UTF-8
entries) Unicode-table bytes — the same byte format `font_psf.cyr`
already walks for inline tables. `kind` is one of
`KASHI_PSF_KIND_PSF1` or `KASHI_PSF_KIND_PSF2`. The existing
`_kashi_build_umap` and `_kashi_build_useqs` helpers are reused with
`unioff = 0` (the whole buffer is the table).

The **text** pair accepts the `psfgettable` output format — a small
line-oriented decoder that lives in `src/lib.cyr` (not heapless;
allocates the umap directly without a synthetic intermediate buffer):

```
0x0020   U+0020
0x0041   U+0041 U+0391    # 'A' or Greek Alpha
# blank lines / comment-only lines ignored
0xFFFD   U+FFFD U+FFFC
```

Per line: a glyph index in hex (`0x<hex>` or `0X<hex>`), then one or
more Unicode codepoints in `U+<hex>` form, then an optional `#`
comment that runs to end-of-line. Multiple codepoints on a line all
map to the same glyph (the common case for alternate encodings —
e.g., LATIN CAPITAL A and GREEK CAPITAL ALPHA both rendering as the
same glyph).

**Scope cuts** for the text parser (kept tight for 0.7.2):
- No continuation lines (`psfaddtable`'s indented-continuation
  feature is rare in practice; covered by repeating the glyph_idx).
- No sequence (ligature) lines — the binary path covers those; the
  text format doesn't have a single canonical sequence syntax
  across implementations.
- Out-of-range glyph indices are silently skipped (matching the
  binary path's `if (glyph < count)` guard in `_kashi_build_umap`).

### Replace semantics

Attaching always **replaces** any existing umap and seqs on the
runtime record. The bump allocator doesn't free the previous arrays;
the storage is leaked-conceptually but harmless (allocations live
until process exit). Callers that want to merge can read out the
current map via `kashi_rt_*` accessors and rebuild. No "merge" API
this cut — keeping the surface tight.

The attach functions operate on runtime fonts only
(`font_id >= KASHI_RT_FONT_BASE`); the built-in font ids 0, 1, 2
return `KASHI_EINVAL`.

### File-size cap for the text variant

`kashi_attach_unicode_text_file` reads at most **256 KiB** (matching
the existing PSF cap). A text Unicode table for a 512-glyph PSF is
~10 KiB at most — 256 KiB is generous.

### Strict overlong-UTF-8 rejection

`kashi_psf_uni_token` (in `src/font_psf.cyr`) gains three minimum-
codepoint checks after computing `cp` from a multi-byte sequence:
- 2-byte sequence: `cp` must be `>= 0x80` (else overlong U+0000–7F).
- 3-byte sequence: `cp` must be `>= 0x800` (else overlong U+0000–7FF).
- 4-byte sequence: `cp` must be `>= 0x10000` (else overlong).

Overlong sequences return `KASHI_TOK_END` (same behavior as the
existing out-of-range / surrogate / bad-continuation rejections in
the F2 tightening). No memory safety implication — the codepoint
would have been a u64 map key either way — but the canonical-
encoding filter belongs in the parser by RFC 3629 spirit.

## Consequences

- **Positive** — a tableless PSF + an externally-described Unicode
  mapping (the `psfaddtable` workflow) is now first-class. Users can
  build the text mapping however they want (a `.tab` file, a config
  resource, a programmatic table); kashi parses it and the unified
  codepoint accessors immediately work.
- **Positive** — UTF-8 strictness now matches RFC 3629. A future
  audit looking for canonical-encoding-bypass tricks will find none
  in the kashi parser.
- **Positive** — `cyaudit vet src/font_psf.cyr` still reports "no
  dependencies"; the strict-overlong check is pure arithmetic.
  `cyaudit vet src/font_data.cyr` is unaffected (boundary intact).
- **Negative** — the attach API works on runtime fonts only; a
  consumer that wants codepoint mapping for a built-in font has to
  fork the model (built-in ids 0/1/2 use the freestanding core's
  ASCII codepoint scheme directly). That's the boundary cost from
  ADR 0001 and is preserved here.
- **Negative** — the text-tab parser is the second text parser in
  the codebase (BDF is the first). It's small (~80 lines including
  comments) and uses the same skipline + parse-int patterns as BDF,
  so the surface area for new bugs is contained.
- **Neutral** — the runtime registry record fields (`RT_UMAP`,
  `RT_UCOUNT`, `RT_SEQS`, `RT_SEQCOUNT`, `RT_SEQDATA`) are reused;
  no struct changes.

## Alternatives considered

- **Synthesize a PSF2 byte stream from the text and reuse
  `_kashi_build_umap`.** Workable, but requires a temp allocation
  sized to "worst-case text → bytes" and an awkward two-pass over
  the text (count for the temp size, then fill). Filling the umap
  directly skips that intermediate.
- **Support text continuation lines (the indented-line convention).**
  Rare in practice; `psfgettable` doesn't emit them by default. The
  same effect can be had by repeating the `0x<idx>` on multiple
  lines. Deferred.
- **Add a `merge` mode that combines a new table with the existing
  umap.** No concrete consumer for it yet. The "replace" semantics
  match how `psfaddtable` works (it replaces the inline table).
- **Reject overlong-UTF-8 only behind a flag.** Considered for
  backward compatibility — but the existing F2 tightening already
  rejected > U+10FFFF and surrogates unconditionally, and there are
  no real-world PSF Unicode tables in the wild that produce
  overlong-encoded codepoints. Unconditional rejection keeps the
  decoder simple.
- **Drop "larger table-walk caps" from the roadmap silently.** The
  ADR mentions it for transparency: the original roadmap item is
  re-examined and found to be a non-issue, not just dropped.
