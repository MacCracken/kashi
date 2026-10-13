# The three glyph tables reserve 8× the bytes they use — 88,592 B of every consumer's image, 7/8 never written — RESOLVED (1.1.1)

**Status:** ✅ **RESOLVED in kashi 1.1.1** (2026-10-13) — see *Resolution* at the end. Filed 2026-10-13 by agnos (1.57.12, the ZFS write path's ROOM step: agnos spec
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

## Resolution (kashi 1.1.1, 2026-10-13)

Fixed as suggested, with two differences (steps 1 and 2). Verified on cyrius 6.6.20 (kashi's new pin) and on
agnos's 6.6.6.

1. **Declarations** — `var kashi_font16[_KASHI_FONT16_SLOTS]` / `kashi_font8[_KASHI_FONT8_SLOTS]` /
   `kashi_font9_16[_KASHI_FONT9_16_SLOTS]`, with `enum _KashiFontStore { 452; 226; 904 }` and comments giving the
   bytes (`3,616 B = 226 glyphs x 16 rows`, …). *Difference:* the sizes are named **slot** counts, not byte counts
   divided by 8. On 6.6.20 an array size takes a literal or an enum constant, never an expression (`[X / 8]` is a parse
   error; expressions arrive in 6.7.2), and enum values are literal-only. Naming them still matters: the test reads the
   same symbol as the declaration. The `_` prefix keeps the enum out of the public API (`docs/api/README.md`).
   **Measured:** library 288,528 → 199,952 B and DCE'd demo 145,376 → 56,800 B on 6.6.20. A freestanding
   `--no-deps` program that includes only `src/font_data.cyr` went 127,312 → 38,736 B on 6.6.6. Each shrank by
   **88,576 B**, 16 B less than 7/8 of 101,248. Alignment explains the gap: 6.6.13 aligns globals, to 16 bytes on x86
   for anything over 8. 1.1.1 also declares the reverse index before the glyph buffers (see 2), which packs one
   alignment gap tighter: 88,560 B without the reorder, 88,576 B with it. Your figure of 88,592 B in the kernel image
   is what agnos's own layout gives.
2. **The padding test** — replaced, as suggested, with a layout-independent check (`src/test.cyr`):
   - **"fset guard"** now checksums every byte the core owns (FNV-1a over the three used spans, the reverse index,
     its count and the ready flag) before and after the out-of-range barrage. It no longer reads a padding byte.
   - **New "storage" group.** Its static half asserts `_KASHI_FONT16_SLOTS * 8 == 226 * 16` and its twins, using
     `==` rather than `<=` so that the 8× over-reservation would fail too. Its measured half asserts that each last
     cell's last row ends its buffer's span, and that the core's six variables are pairwise disjoint, read from the
     compiled addresses (order-free).

   *Difference:* the guard test was reworked rather than dropped. While reworking it, a pre-existing blind spot showed
   up: nothing kashi owns sat below `kashi_font16`, so removing `kashi_fset16`'s *lower* guard went unnoticed (it
   still did after the right-sizing). The reverse index is now declared first, and with cycc's declaration-order layout
   all four guards' writes land on checksummed bytes. **Mutation-tested**, 12 mutants, all caught:
   - each of the four guards removed;
   - the upper guard off by one;
   - each slot count one less, and `_KASHI_FONT8_SLOTS` one more;
   - `_KASHI_FONT16_SLOTS` back to 3616;
   - an extension cell added without growing the buffers;
   - a declaration bypassing the enum.

   Unit suite 448 → 454.
3. **Docs** — `docs/architecture/001-module-scope-var-array-byte-addressing.md` is rewritten: N counts u64 slots,
   each buffer declares bytes ÷ 8, it explains why the "mirrors agnos / would under-allocate" reasoning did not hold,
   and it notes that the storage sits in the file image, not BSS. Also updated: the storage-model note at the top of
   `src/font_data.cyr`, `SECURITY.md`, the getting-started add-a-font checklist, `state.md` and `CHANGELOG.md`
   *1.1.1*.
4. **Consumers** — no change needed: same symbols, offsets and accessor results. The 678-line glyph sheet hashes to
   the 1.1.0 pin `5544d63f…`, and the demo's output is byte-identical. On agnos's 6.6.6 pin the 1.1.1 core compiles
   freestanding, passes kashi's 454 + 145 assertions, and renders the identical sheet. kashi did not build agnos itself
   (sibling rule). Your `KASHI_REF` bump to 1.1.1 and the console smokes are agnos-side.
