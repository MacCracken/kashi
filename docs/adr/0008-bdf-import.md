# 0008 — BDF import

**Status**: Accepted
**Date**: 2026-05-28
**Refines / extends**: [ADR 0002](0002-runtime-font-registry.md) (uses the
same runtime registry and unified accessor dispatch),
[ADR 0003](0003-codepoint-addressing-runtime-fonts.md) (uses the same
codepoint→glyph map shape)

## Context

Through 0.6.0 the only runtime-import format kashi understands is PSF
(PSF1 / PSF2). PSF is a tight binary format, perfect for kernel consoles
but rare outside the Linux world. The X11 ecosystem's bitmap fonts ship
as **BDF** (Bitmap Distribution Format — a verbose text source) and
**PCF** (its compiled binary). Many high-quality bitmap console fonts
(Terminus, Spleen, Tamzen, Cozette, Unifont) are distributed as BDF
first; PSF builds, when they exist, are downstream conversions.

0.7.0 adds **BDF import**. The freestanding boundary stays intact — BDF
is library-face only, with a self-contained heapless parser (the same
posture as `src/font_psf.cyr`).

Three forces shape the design:

1. **kashi's runtime store is fixed-stride.** Each font's glyph bytes
   live in one contiguous block of `count × charsize` bytes, with
   `charsize = height × ceil(width/8)`. Every glyph is the same shape.
2. **BDF allows per-glyph geometry.** Each glyph carries its own
   `BBX w h xoff yoff` declaration; the font-wide `FONTBOUNDINGBOX` is
   nominally a maximum, not a fixed cell.
3. **BDF is the first text-based parser kashi has written.** Everything
   so far (the freestanding tables, PSF1, PSF2) is binary — load a
   header, read fixed-width fields, walk fixed-stride glyph bytes. BDF
   needs a small text-line scanner with whitespace handling, signed
   decimal parsing, and hex-byte parsing.

## Decision

**A self-contained, heapless BDF parser in `src/font_bdf.cyr` (mirroring
`src/font_psf.cyr`); the library face (`src/lib.cyr`) owns allocation,
registration, and the codepoint→glyph map build.**

### Strict BBX policy (uniform geometry)

Every glyph's `BBX w h xoff yoff` must match the font's
`FONTBOUNDINGBOX w h xoff yoff` exactly. A mismatch → `KASHI_EFORMAT`.

The console-bitmap-font corpus (Terminus, Spleen, Tamzen, Cozette,
Unifont's 8×16/16×16 ranges) satisfies this. Display BDFs with variable
per-glyph BBX — italic overhangs, descender-using outlines — are out of
scope for 0.7.0; padding-into-cell support can be a later additive cut
without changing the on-disk contract.

This is the same uniform-geometry assumption PSF already makes; kashi's
runtime store can't represent anything else without growing per-glyph
metadata.

### Reuse the PSF parsed-header struct

`kashi_bdf_parse_header(buf, len, out)` fills the **same 56-byte
parsed-header struct** the PSF parser uses (`KASHI_PH_KIND` /
`_WIDTH` / `_HEIGHT` / `_COUNT` / `_CHARSIZE` / `_DATAOFF` / `_UNIOFF`).
BDF gets a new `KASHI_PH_KIND` value of **3**. `KASHI_PH_DATAOFF` carries
the byte offset of the first `STARTCHAR` (not glyph bytes — there's no
contiguous glyph region in a BDF). `KASHI_PH_UNIOFF` is **always 0** —
BDF carries no separate Unicode table; codepoints live inside each
`STARTCHAR..ENDCHAR` block as `ENCODING <decimal>`.

Reusing the header struct keeps `lib.cyr`'s wiring symmetric with PSF
and avoids growing a new "parsed-header" surface area.

### Per-glyph cursor (no contiguous-block memcpy)

PSF's `lib.cyr` does one `memcpy(owned, buf + dataoff, count * charsize)`
because PSF glyph bytes are already laid out exactly that way. BDF can
not — each glyph's bitmap is a hex-encoded text block. So the parser
exposes a cursor:

```
kashi_bdf_next_glyph(buf, pos, end, hdr_w, hdr_h, dest, dest_size, out)
```

At a `STARTCHAR` keyword, this:
1. Reads `ENCODING <int>` and writes the codepoint into `out + BG_CP`.
2. Validates the next `BBX` line against `hdr_w`/`hdr_h` (and offsets);
   non-match → returns `KASHI_EFORMAT`.
3. Reads the `BITMAP` line, then decodes `hdr_h` rows of
   `ceil(hdr_w/8)` hex bytes each directly into `dest` (caller-provided
   `height * stride` bytes).
4. Skips to `ENDCHAR`, writes the post-`ENDCHAR` offset into
   `out + BG_NEXT`, and returns `KASHI_OK`.

Sentinel codepoints in `out + BG_CP`:
- `0 - 1` (i.e. `(u64) -1`) → `ENCODING -1` (unencoded glyph; lib.cyr
  skips both the bitmap copy and the cp-map entry).
- `0 - 2` → cursor reached `ENDFONT` or buffer end with no more glyphs
  (caller stops iterating).

This keeps the parser pure — `lib.cyr` decides allocation strategy
(over-allocate to declared `CHARS`, fill, truncate-in-place by storing
the final count); the parser only validates and decodes.

### Skip `ENCODING -1` entirely

BDF lets a glyph be named (`STARTCHAR something`) without a codepoint
(`ENCODING -1` or a `STARTCHAR` with no `ENCODING` line). 0.7.0 drops
these completely — not loaded into the glyph store, not in the
codepoint map. This matches the "codepoint-addressable bitmap font"
model the rest of kashi already enforces and avoids a separate "this
runtime font has N glyph slots but only M are addressable by codepoint"
state. Keep-in-index-but-skip-the-map was rejected (see Alternatives).

### File-size cap = 4 MiB (`kashi_load_bdf_file`)

PSF's `kashi_load_psf_file` reads at most 256 KiB — fine because PSF is
~10 bytes of header plus packed glyph bytes. BDF is 8–16× more verbose
(decimal coordinates, hex bytes with newlines, keyword overhead,
optional `STARTPROPERTIES..ENDPROPERTIES` blocks). 4 MiB covers every
real-world console BDF (Terminus, Spleen, Tamzen, Unifont's
8×16/16×16 ranges are all well under 1 MiB; a fully-loaded display BDF
might push toward 2 MiB). It is also a hard ceiling on the transient
read buffer this routine allocates.

A new constant `KASHI_BDF_FILE_CAP = 4194304` lives in `font_bdf.cyr`
for grep-discoverability.

### `STARTPROPERTIES..ENDPROPERTIES` is skipped without parse

The properties block (font name, foundry, size descriptors, weight)
carries no bitmap data and no glyph dimensions kashi needs.
`kashi_bdf_parse_header` recognises the `STARTPROPERTIES` line and
walks forward to `ENDPROPERTIES` without parsing the contents. A
malformed properties block (no `ENDPROPERTIES` before `CHARS`) →
`KASHI_EFORMAT`.

### Result code reuse — no new `KashiResult` values

Validation failures → `KASHI_EFORMAT`. Argument validation (null buf,
negative length, too-large width/height/count) → `KASHI_EINVAL`. Same
two codes PSF already uses. No new error codes.

## Consequences

- **Positive** — kashi now imports the de-facto X11 source format. Most
  bitmap console fonts in the wild ship as BDF; previously they needed
  an external `bdftopcf`/`bdf2psf` conversion before kashi could load
  them.
- **Positive** — `cyaudit vet src/font_data.cyr` still reports "no
  dependencies"; `cyaudit vet src/font_bdf.cyr` should likewise report
  "no dependencies". Freestanding boundary intact.
- **Positive** — the parsed-header struct shape stays uniform across
  PSF1, PSF2, and now BDF. `lib.cyr` reads the same fields by the same
  offsets regardless of format kind.
- **Negative** — strict BBX rules out display BDFs with per-glyph
  geometry variation (italic overhangs, descenders). Documented up
  front; lifting this is a later additive cut.
- **Negative** — BDF parsing is necessarily more code than PSF (text
  vs. binary, line scanning, whitespace handling). The parser is
  ~3–4× the lines of `font_psf.cyr`. Heaplessness + load8-only-style
  keeps the surface fuzzable and `cyaudit vet`-able.
- **Neutral** — the runtime record layout (`RT_*` offsets in `lib.cyr`)
  does not change — BDF fonts use exactly the same record fields PSF
  fonts already use. Sequence/ligature fields (`RT_SEQS` etc.) stay
  zero for BDF (BDF has no ligature concept).
- **Neutral** — a new `KASHI_PH_KIND = 3` (BDF) appears in the kind
  enumeration; consumers branching on kind values must learn it. No
  existing PSF consumer reads `KASHI_PH_KIND` (it's an internal
  parsed-header field), so this is a no-op in practice.

## Alternatives considered

- **Lenient BBX with padding into FONTBOUNDINGBOX.** Rejected for 0.7.0:
  doubles the parser's coordinate-arithmetic surface (signed BBX
  xoff/yoff vs. FONTBOUNDINGBOX origin), needs a per-row shift+mask, and
  the console-bitmap corpus doesn't need it. Reserved as a possible
  future additive cut.
- **Keep `ENCODING -1` glyphs in the index but omit from the cp-map.**
  Workable, but adds a "named but unaddressable" state consumers must
  reason about, and the runtime registry has no `name` field today.
  Skipping outright matches the codepoint-addressable model kashi
  already enforces.
- **Pre-extract `STARTPROPERTIES` into a properties map.** Out of scope
  for 0.7.0; kashi is a glyph-data provider, not a font-metadata layer.
  A consumer that wants `FONT_ASCENT` can parse the source separately.
- **A streaming `kashi_load_bdf_stream(read_fn, ctx)` interface.** More
  flexible than the in-memory buffer + file-path pair, but no concrete
  consumer for it yet; current `kashi_load_psf*` doesn't stream either.
  Defer until a use case appears.
- **Reuse PSF's parsed-header struct fields verbatim, claim `KIND=2` for
  BDF.** Rejected — conflating kinds loses the diagnostic that says
  "this came in as a BDF" if a future bug forensic needs it. Adding
  a new `KIND=3` is one literal in an enum.
- **Bigger / smaller file cap (1 MiB or 16 MiB).** Rejected (each):
  1 MiB would reject a fully-loaded Unifont BMP dump; 16 MiB makes the
  transient allocation needlessly fat. 4 MiB lands cleanly between.
