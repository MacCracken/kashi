# `kashi_pcf_cp_to_idx` wraps a codepoint above U+FFFF into the BMP — RESOLVED (1.2.2)

**Status:** ✅ **RESOLVED in kashi 1.2.2** (2026-10-29) — see *Resolution* at the end. Found 2026-10-29 by the 1.2.1
doc sweep while checking `docs/api/parsers.md` against `src/font_pcf.cyr`. Not fixed in 1.2.1, a toolchain + docs
release that changed no source.
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

## Resolution (1.2.2)

Fixed as suggested: `if (cp > 0xFFFF) { return 0 - 1; }` sits in `kashi_pcf_cp_to_idx`'s guard block, after `cp < 0`.
- **Unit.** A new group "pcf cp_to_idx" (10 assertions) against `_mk_pcf`'s font (0x41..0x43, byte1 = 0) covers:
  - in-range hits;
  - out-of-range misses (`0x40`, `0x141`, `0xFFFF`);
  - `0x10041`, `0x110041` and `0x7FFFFFFFFFFF0041`, whose low 16 bits all map;
  - a negative.
- **Fuzz.** The PCF contract gains code 11: a cp above U+FFFF must miss.
  - The first draft probed random codepoints and did **not** catch the bug with the guard removed. The fuzz template maps
    two codepoints, so a random cp has about 2 in 65,536 odds of mapped low bits.
  - The probe now splices a random nonzero high part (up to 2^46) onto a codepoint drawn from the font's own encoded
    range.
- **Mutation.** With the guard removed, the unit suite fails 3 assertions and the fuzz harness exits 11.
- **Gates.** The glyph sheet and the demo output are byte-identical, and the sizes and the bench are unchanged.
  `dist/kashi.cyr` regenerated.
