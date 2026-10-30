# `kashi_pcf_cp_to_idx` wraps a codepoint above U+FFFF into the BMP — OPEN

**Status:** 🟠 **OPEN.** Found 2026-10-29 by the 1.2.1 doc sweep while checking `docs/api/parsers.md` against
`src/font_pcf.cyr`. Not fixed in 1.2.1 (a toolchain + docs release that changes no source); the API reference carries
a caveat until it is.
**Severity:** Low. It gives a wrong answer, not an out-of-bounds read. The library face's PCF loader is not affected.
**Affects:** a direct caller of the public parser function `kashi_pcf_cp_to_idx` that passes `cp > 0xFFFF`. Present
since 0.7.1 (ADR 0009).

## Summary

`src/font_pcf.cyr:672-694` resolves a codepoint through the BDF_ENCODINGS table by splitting it into two bytes:

```cyrius
var b1 = (cp >> 8) & 0xFF;
var b2 = cp & 0xFF;
```

The `& 0xFF` on `b1` drops every bit above bit 15, so `cp` and `cp & 0xFFFF` resolve to the same cell. With a font
whose encoded range includes `min_b1 = 0` (any font covering ASCII), `kashi_pcf_cp_to_idx(buf, &ctx, 0x10041)` returns
the glyph for U+0041 'A'. It should return `0 - 1`, as `docs/api/parsers.md` documents for a codepoint outside the
encoded range. BDF_ENCODINGS has byte-sized `min/max_byte1`, so no PCF can encode anything above U+FFFF.

## Why nothing caught it

- The library face's loader (`src/lib.cyr:607-622`) calls it only with `cp = (b1 << 8) | b2` for `b1` / `b2` inside
  the table's own ranges, so a runtime PCF font's codepoint map never holds a wrapped entry. `kashi_font_row` /
  `kashi_font_ptr` on a PCF font resolve through that map, not through this function.
- The fuzz harness (`tests/kashi.fcyr:583-588`) draws `cp` up to `0xFFFFF` but checks only that the result is in
  `[0 - 1, count)`. A wrapped index is in range, so it passes.

## Suggested fix

Reject the high bits before splitting, in the same guard block as `cp < 0`:

```cyrius
if (cp > 0xFFFF) { return 0 - 1; }
```

Add a unit assertion that `kashi_pcf_cp_to_idx` on a supplementary-plane codepoint returns `0 - 1` for a synthetic PCF
that maps U+0041. Tighten the fuzz contract too: for `cp > 0xFFFF` the result must be `0 - 1`. Then regenerate
`dist/kashi.cyr`. It is a patch: the documented contract already says `0 - 1`, and only the wrong answer changes.
Remove the caveat in `docs/api/parsers.md` in the same change.
