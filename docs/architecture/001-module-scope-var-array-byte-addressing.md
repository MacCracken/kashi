# 001 — Module-scope glyph buffers are byte-addressed inside u64-unit BSS

> **Last Updated**: 2026-05-27

**Affects**: `src/font_data.cyr` (`kashi_font16`, `kashi_font8`, the
`kashi_fset16` / `kashi_fset8` packers, `kashi_glyph_ptr`).

## The invariant

The freestanding core declares its glyph storage as module-scope arrays:

```cyrius
var kashi_font16[1536];   # VGA 8x16: 96 glyphs x 16 rows
var kashi_font8[768];     # CGA 8x8 : 96 glyphs x  8 rows
```

In Cyrius, **module-global `var X[N]` allocates `N * u64` (8N bytes)**,
whereas a **function-local `var X[N]` allocates N bytes**. (Two scopes, two
unit conventions — see the genesis memory `cyrius-var-array-u64-units`.)

So `kashi_font16[1536]` actually reserves **12 KiB** of BSS, and
`kashi_font8[768]` reserves **6 KiB**. The code uses only the first 1536 /
768 **bytes** — one byte per glyph row — via `store8` / `load8`. The upper
7/8 of each buffer is reserved-but-unused padding.

## Why it's written this way (and not "fixed")

This is **deliberate and mirrors agnos exactly.** The agnos kernel's
`fb_console.cyr` declares `var fb_font[1536]` and over-reserves the same
way; matching its declaration keeps the drop-in semantics obvious to anyone
diffing the two. A consumer reasoning about memory footprint must know the
real cost is 8× the element count, not the byte count the name implies.

Do **not** "optimize" these to `[192]` / `[96]` (bytes ÷ 8) — that would
under-allocate, because the unit is u64 *only at module scope* and the
buffers are addressed per byte. The element count IS the byte count we
intend to use; the 8× is the language's allocation unit, not a bug.

## Consequence for accessors

Because storage is byte-addressed, `kashi_glyph_ptr` returns a raw byte
address (`&kashi_font16 + (ch - 0x20) * 16`) and the consumer reads
`height` consecutive bytes with `load8`. Glyph N and glyph N+1 are exactly
`height` bytes apart — a property the integration tests assert
(`test_pointer_monotone`). If the storage model ever changed to packed u64s,
every accessor and every consumer's read loop would change with it; that's
why this is an architecture note, not just a comment.
