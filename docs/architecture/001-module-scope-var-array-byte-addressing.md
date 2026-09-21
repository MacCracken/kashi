# 001 — Module-scope glyph buffers are byte-addressed inside u64-unit BSS

> **Last Updated**: 2026-09-21 (sizes refreshed for the CP437 range and the
> 9×16 built-in; the invariant is unchanged since 0.1.0)

**Affects**: `src/font_data.cyr` (`kashi_font16`, `kashi_font8`,
`kashi_font9_16`, the `kashi_fset16` / `kashi_fset8` packers, the 9×16
derivation loop in `kashi_font_init`, `kashi_glyph_ptr`,
`kashi_glyph_row`, `kashi_glyph_row_byte`).

## The invariant

The freestanding core declares its glyph storage as module-scope arrays:

```cyrius
var kashi_font16[3584];    # VGA 8x16: 224 glyphs x 16 rows
var kashi_font8[1792];     # CGA 8x8 : 224 glyphs x  8 rows
var kashi_font9_16[7168];  # VGA 9x16: 224 glyphs x 16 rows x 2 bytes (stride 2)
```

In Cyrius, **module-global `var X[N]` allocates `N * u64` (8N bytes)**,
whereas a **function-local `var X[N]` allocates N bytes**. (Two scopes, two
unit conventions — see the genesis memory `cyrius-var-array-u64-units`.)

So `kashi_font16[3584]` actually reserves **28 KiB** of BSS,
`kashi_font8[1792]` reserves **14 KiB**, and `kashi_font9_16[7168]` reserves
**56 KiB**. The code uses only the first 3584 / 1792 / 7168 **bytes** — one
byte per glyph row for the 8-wide fonts, two per row for the 9×16 — via
`store8` / `load8`. The upper 7/8 of each buffer is reserved-but-unused
padding. (0.1.0 shipped `[1536]` / `[768]` for 96 glyphs; the CP437
widening in 0.4.0 and the derived 9×16 in 0.5.1 grew the numbers, not the
rule.)

## Why it's written this way (and not "fixed")

This is **deliberate and mirrors agnos exactly.** The agnos kernel's
`fb_console.cyr` declared `var fb_font[1536]` and over-reserved the same
way (it now takes kashi's tables instead); matching its declaration keeps the drop-in semantics obvious to anyone
diffing the two. A consumer reasoning about memory footprint must know the
real cost is 8× the element count, not the byte count the name implies.

Do **not** "optimize" these to `[448]` / `[224]` / `[896]` (bytes ÷ 8) — that would
under-allocate, because the unit is u64 *only at module scope* and the
buffers are addressed per byte. The element count IS the byte count we
intend to use; the 8× is the language's allocation unit, not a bug.

## Consequence for accessors

Because storage is byte-addressed, `kashi_glyph_ptr` returns a raw byte
address (`&kashi_font16 + (ch - 0x20) * 16`) and the consumer reads
`height * stride` consecutive bytes with `load8` — `kashi_glyph_row` returns
the leading byte of a row and `kashi_glyph_row_byte(id, ch, row, b)` the
rest, with `kashi_font_stride(id)` giving the bytes per row (1, 1, 2).
Glyph N and glyph N+1 are exactly `height * stride` bytes apart — a property
the integration tests assert (`test_pointer_monotone`, and
`test_vga_9x16_structural` for the stride-2 buffer). If the storage model ever changed to packed u64s,
every accessor and every consumer's read loop would change with it; that's
why this is an architecture note, not just a comment.
