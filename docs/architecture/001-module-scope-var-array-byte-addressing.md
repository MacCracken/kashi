# 001 — Module-scope glyph buffers are byte-addressed inside u64-unit BSS

> **Last Updated**: 2026-10-09 (1.1.0 — sizes for the two extension cells,
> and the one deliberate exception: the Unicode reverse index; the invariant
> is unchanged since 0.1.0)

**Affects**: `src/font_data.cyr` (`kashi_font16`, `kashi_font8`,
`kashi_font9_16`, the `kashi_fset16` / `kashi_fset8` packers, the 9×16
derivation loop in `kashi_font_init`, `kashi_glyph_ptr`,
`kashi_glyph_row`, `kashi_glyph_row_byte`; and, as the stated exception,
`kashi_cp437_rev` / `kashi_cp437_from_unicode`).

## The invariant

The freestanding core declares its glyph storage as module-scope arrays:

```cyrius
var kashi_font16[3616];    # VGA 8x16: 226 glyphs x 16 rows
var kashi_font8[1808];     # CGA 8x8 : 226 glyphs x  8 rows
var kashi_font9_16[7232];  # VGA 9x16: 226 glyphs x 16 rows x 2 bytes (stride 2)
```

226 = the 224 CP437 cells `0x20..0xFF` followed directly by the two
extension cells `0x100..0x101` (1.1.0, ADR 0011) — contiguous, so the one
`(ch - 0x20) * size` offset serves both.

In Cyrius, **module-global `var X[N]` allocates `N * u64` (8N bytes)**,
whereas a **function-local `var X[N]` allocates N bytes**. (Two scopes, two
unit conventions — see the genesis memory `cyrius-var-array-u64-units`.)

So `kashi_font16[3616]` actually reserves **28.25 KiB** of BSS,
`kashi_font8[1808]` reserves **14.1 KiB**, and `kashi_font9_16[7232]`
reserves **56.5 KiB**. The code uses only the first 3616 / 1808 / 7232
**bytes** — one byte per glyph row for the 8-wide fonts, two per row for the
9×16 — via `store8` / `load8`. The upper 7/8 of each buffer is
reserved-but-unused padding (and `src/test.cyr`'s fset-guard group asserts
the first padding byte stays zero). (0.1.0 shipped `[1536]` / `[768]` for 96
glyphs; the CP437 widening in 0.4.0, the derived 9×16 in 0.5.1 and the
extension cells in 1.1.0 grew the numbers, not the rule.)

Cyrius compiles module-scope arrays into the binary image, so the
reservation is not free: every consumer's binary carries it.

## Why it's written this way (and not "fixed")

This is **deliberate and mirrors agnos exactly.** The agnos kernel's
`fb_console.cyr` declared `var fb_font[1536]` and over-reserved the same
way (it now takes kashi's tables instead); matching its declaration keeps the drop-in semantics obvious to anyone
diffing the two. A consumer reasoning about memory footprint must know the
real cost is 8× the element count, not the byte count the name implies.

Do **not** "optimize" these to `[452]` / `[226]` / `[904]` (bytes ÷ 8) — that would
under-allocate, because the unit is u64 *only at module scope* and the
buffers are addressed per byte. The element count IS the byte count we
intend to use; the 8× is the language's allocation unit, not a bug.

## The one exception: `kashi_cp437_rev[128]`

The Unicode reverse index (1.1.0) is **not** sized in bytes:

```cyrius
var kashi_cp437_rev[128];  # 128 entries x one u64 slot (1 KiB): byte 0 = cp >> 8,
                           # byte 1 = cp & 0xFF, byte 2 = cell, bytes 3-7 unused
```

Here N counts **entries**, each owning one whole 8-byte slot — the
language's own unit, fully used, addressed `&kashi_cp437_rev + i * 8` with
`load8` / `store8`. It has no agnos declaration to mirror, and "N = bytes"
would reserve 8 KiB in every consumer for a 1 KiB table. Its declaration
carries the warning; do not "fix" it to `[1024]`.

## Consequence for accessors

Because storage is byte-addressed, `kashi_glyph_ptr` returns a raw byte
address (`&kashi_font16 + (ch - 0x20) * 16`, for a CP437 or extension
cell `ch`) and the consumer reads
`height * stride` consecutive bytes with `load8` — `kashi_glyph_row` returns
the leading byte of a row and `kashi_glyph_row_byte(id, ch, row, b)` the
rest, with `kashi_font_stride(id)` giving the bytes per row (1, 1, 2).
Glyph N and glyph N+1 are exactly `height * stride` bytes apart — a property
the integration tests assert (`test_pointer_monotone`, and
`test_vga_9x16_structural` for the stride-2 buffer). If the storage model ever changed to packed u64s,
every accessor and every consumer's read loop would change with it; that's
why this is an architecture note, not just a comment.
