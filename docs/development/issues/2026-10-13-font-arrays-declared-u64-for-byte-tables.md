# The three glyph tables reserve 8× the bytes they use — 88,592 B of every consumer's image, 7/8 never written — OPEN

**Status:** 🟡 **OPEN** — filed 2026-10-13 by agnos (1.57.12, the ZFS write path's ROOM step: agnos spec
`docs/development/proposals/2026-10-09-zfs-write.md` § k.74 (e)). Measured against kashi **1.1.0** (`93909d0`,
`src/font_data.cyr`), built into the agnos kernel with cyrius **6.6.6**. No kashi code was changed by agnos (sibling
rule: filed here, fixed here).
**Severity:** Low for kashi's own tests and demos; **material for a freestanding consumer with a fixed image ceiling**.
agnos's kernel image must end at or below `0x400000` (the top of its region 1 — the last bound its layout allows);
these three tables are 101,248 B of that image, of which 88,592 B are never written.
**Affects:** every consumer that includes `src/font_data.cyr` — agnos since 1.38.0 (`[deps.kashi] modules =
["src/font_data.cyr"]`); the sizes grew with 0.4.0 (CP437), 0.5.1 (9×16) and 1.1.0 (the two extension cells).

## Summary

`src/font_data.cyr:124-126` declares the glyph storage with the element count equal to the number of BYTES the code
uses:

```cyrius
var kashi_font16[3616];    # 226 glyphs x 16 rows (VGA 8x16)        — bytes 0..3615
var kashi_font8[1808];     # 226 glyphs x  8 rows (CGA 8x8)         — bytes 0..1807
var kashi_font9_16[7232];  # 226 glyphs x 16 rows x 2 bytes (VGA 9x16, derived)
```

At module scope a Cyrius `var X[N]` is **N u64 slots = 8N bytes**. So the three tables reserve 28,928 + 14,464 +
57,856 = **101,248 B**, and every access stays inside the first 3,616 / 1,808 / 7,232 bytes (`store8` / `load8` at
`&kashi_fontNN + (ch - 0x20) * stride + row`: `kashi_fset16` :155, `kashi_fset8` :181, the 9×16 derivation :742-743,
`kashi_glyph_ptr` :891-897). **12,656 B used, 88,592 B (7/8) reserved and never written.**

`docs/architecture/001-module-scope-var-array-byte-addressing.md` states the over-reservation and keeps it on purpose
("mirrors agnos exactly" — agnos's old `var fb_font[1536]` — and "do not optimize these to `[452]` / `[226]` / `[904]`
… that would under-allocate"). The second half of that reasoning does not hold: **at module scope `[452]` is
452 × 8 = 3,616 bytes — exactly the bytes `kashi_font16` uses**, not an under-allocation; likewise `[226]` = 1,808 B
and `[904]` = 7,232 B. The unit only differs inside a function, and these arrays are module-scope. The agnos
declaration it mirrored is gone (agnos takes kashi's tables since 1.38.0), so the mirror no longer buys anything.
`kashi_cp437_rev[128]` (1 KiB, one u64 slot per entry) is correctly sized and is not part of this issue.

## Evidence — measured in agnos's kernel image

agnos 1.57.12 tree (2026-10-13), plain build, `sh scripts/build.sh` with the pinned kashi 1.1.0 vs a scratch copy of
the same kashi whose three declarations alone read `[452]` / `[226]` / `[904]` (agnos's gate 34 prints the headroom
to `0x400000`):

| kashi `src/font_data.cyr` | `build/agnos` | LOAD end | headroom under `0x400000` |
|---|---|---|---|
| 1.1.0 as shipped (`[3616]` / `[1808]` / `[7232]`) | 2,754,592 B | `0x3b06c0` | 325,952 B |
| the three declarations ÷ 8 (`[452]` / `[226]` / `[904]`) | 2,666,000 B | `0x39acb0` | **414,544 B** |

The difference is **88,592 B**, exactly 7/8 of 101,248 — the image file and the LOAD end both shrink by it (cycc
places these module-scope arrays in the file image). The scratch build compiled and passed agnos's layout gate; it
was not booted (the glyph bytes are written and read only through the offsets above, all inside the ÷ 8 sizes).

## Fix (suggested — kashi's call)

1. Declare the three tables in u64 slots: `var kashi_font16[452];`, `var kashi_font8[226];`,
   `var kashi_font9_16[904];` — or, to keep the byte counts visible, derive them from named constants
   (`KASHI_FONT16_BYTES = 3616` … and `[KASHI_FONT16_BYTES / 8]` if the compiler takes a constant expression there),
   with the comments saying "N u64 = 8N bytes = the bytes used".
2. `src/test.cyr` :394-399 asserts that the first padding byte past each table's used span stays zero
   (`load8(&kashi_font16 + used16) == 0`). Right-sized, there is no padding: that address is the NEXT variable. Replace
   the guard with one that needs no padding — e.g. the packers' bound checks (`kashi_fset16` / `kashi_fset8` refuse a
   cell past `KASHI_EXT_LAST`) asserted directly, and a static check that `226 * 16 <= 8 * 452` (and the 8×8 / 9×16
   twins) so a future glyph added past the extension block fails the test instead of overrunning.
3. Update `docs/architecture/001-module-scope-var-array-byte-addressing.md` (the invariant becomes "N u64 slots hold
   exactly the bytes used") and the storage-model note at `src/font_data.cyr:47-58`.
4. A consumer needs no change: the symbols, offsets and accessors are the same. agnos would take it through its usual
   kashi pin bump (`KASHI_REF` in `build.sh` / `test.sh` / `bench.sh`) and re-run its console smokes.

## Not affected

The fonts' bytes, the glyph offsets, the 9×16 derivation and every accessor's results are unchanged by the fix — only
the reservation shrinks.
