# 001 — Module-scope glyph buffers are byte-addressed inside u64-slot arrays

> **Last Updated**: 2026-10-29 (1.2.0 — array-size expressions compile from
> cyrius 6.7.2; the core keeps enum sizes for its consumers' 6.6.x pins.
> 1.1.1 — the buffers are declared in u64
> slots, sized to exactly the bytes used; until 1.1.1 the element count was
> the byte count, which reserved 8× and left 7/8 of every buffer unused)

**Affects**: `src/font_data.cyr` (`kashi_font16`, `kashi_font8`,
`kashi_font9_16`, their sizes `_KashiFontStore`, the `kashi_fset16` /
`kashi_fset8` packers, the 9×16 derivation loop in `kashi_font_init`,
`kashi_glyph_ptr`, `kashi_glyph_row`, `kashi_glyph_row_byte`; and
`kashi_cp437_rev` / `kashi_cp437_from_unicode`); `src/test.cyr`'s
"storage" and "fset guard" groups.

## The invariant

In Cyrius, **a module-scope `var X[N]` reserves N u64 slots = 8N bytes**,
whereas a **function-local `var X[N]` reserves N bytes**. (Two scopes, two
units — see the genesis memory `cyrius-var-array-u64-units`.)

The freestanding core addresses its glyph storage **per byte** (`store8` /
`load8`) and declares each buffer as **bytes used ÷ 8** slots:

```cyrius
enum _KashiFontStore {
    _KASHI_FONT16_SLOTS = 452;    # 3,616 B = 226 glyphs x 16 rows
    _KASHI_FONT8_SLOTS = 226;     # 1,808 B = 226 glyphs x  8 rows
    _KASHI_FONT9_16_SLOTS = 904;  # 7,232 B = 226 glyphs x 16 rows x 2 bytes
}
var kashi_font16[_KASHI_FONT16_SLOTS];      # VGA 8x16
var kashi_font8[_KASHI_FONT8_SLOTS];        # CGA 8x8
var kashi_font9_16[_KASHI_FONT9_16_SLOTS];  # VGA 9x16 (derived at init)
```

226 = the 224 CP437 cells `0x20..0xFF` followed directly by the two
extension cells `0x100..0x101` (1.1.0, ADR 0011) — contiguous, so the one
`(ch - 0x20) * size` offset serves both. **N u64 slots hold exactly the
bytes used**: no slack past the last cell.

The sizes are an enum, not literals, so that `src/test.cyr`'s "storage"
group checks the same N the declarations use (`_KASHI_FONT16_SLOTS * 8 ==
226 * 16`, and its twins). Below cyrius 6.7.2 an array size takes a
literal or an enum constant, not an expression, so it cannot be written
`[3616 / 8]`. From 6.7.2 it can: on kashi's 6.7.9 pin (1.2.0)
`var a[3616 / 8];` reserves 452 slots, measured. The core still keeps the
enum and its literal sizes. agnos, dhancha, crab, puka, jalwa and
aethersafha compile `src/font_data.cyr` on their own 6.6.x pins, so the core
must not use syntax added in 6.7. The `_` prefix marks the enum internal
(`docs/api/README.md`).

### Adding a cell or a font

A cell appended after `KASHI_EXT_LAST` grows each buffer by
`rows × stride` bytes — **N by `rows × stride / 8`**: +2 slots for the
8×16, +1 for the 8×8, +4 for the 9×16. The row counts are all multiples of
8, so the division is exact. Skip it and the new cell's rows land on the
next variable — with no slack, that is the next glyph buffer. The "storage"
group fails until the sizes move (mutation-checked: an extension cell added
without growing the buffers fails four assertions).

## The size, and who pays it

cycc writes module-scope storage into the **file image** as zeros — it is
not a BSS segment that costs nothing on disk — so every byte reserved here
is a byte of every binary that links the core, the agnos kernel image
included. 1.1.1's sizing is 12,656 B for the three glyph buffers (plus the
1 KiB reverse index); 1.1.0 reserved 101,248 B for the same 12,656 B used.

Measured on cyrius 6.6.20 (1.1.0 → 1.1.1, nothing else changed): the
library build 288,528 → 199,952 B, the `CYRIUS_DCE=1` demo 145,376 → 56,800
B; a freestanding `--no-deps` program that includes only the core, on
agnos's 6.6.6 pin, 127,312 → 38,736 B — **−88,576 B** each. That is 16 B
short of 7/8 of 101,248 (88,592) because globals are aligned (6.6.13: 8,
and on x86 16 for anything over 8 bytes), and the 1.1.1 declaration order
packs one alignment gap tighter. agnos measured the full 88,592 B of
headroom in its own image (issue
`docs/development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md`).

## Why it was 8× until 1.1.1, and why that reasoning did not hold

Through 1.1.0 this note said N was the **byte** count on purpose: it
mirrored agnos's old `var fb_font[1536]` declaration (8× over-reserved the
same way), and "shrinking to bytes ÷ 8 would under-allocate". The second
claim was wrong — at module scope `[452]` *is* 3,616 bytes, exactly the
bytes `kashi_font16` uses; the unit only differs inside a function, and
these arrays are module scope. The first stopped buying anything once agnos
took kashi's tables (1.38.0) and dropped its own declaration. agnos filed
it when the 0x400000 ceiling of its kernel image made the 88 KB visible.

## Layout is the compiler's

Nothing in kashi computes one global's address from another's: every
access is `&buffer + offset` within that buffer's own span. That matters
now that the buffers have no slack, and cyrius 6.6.13 states it outright —
the gap between two globals is the compiler's. Two tests do look at the
layout, both in a way that cannot false-fail:

- **"storage"**: the core's variables (the three used spans, the reverse
  index, its count, the ready flag) are pairwise disjoint, read from the
  compiled addresses — order-free.
- **"fset guard"**: an out-of-range pack must leave every byte the core
  owns unchanged (one FNV-1a over all of them). With cycc's
  declaration-order layout a skipped guard's write lands on a neighbour
  that checksum covers — past the end of `kashi_font16` is `kashi_font8`,
  and below it the reverse index, declared first for that reason in 1.1.1
  (until then nothing kashi owned sat below `kashi_font16`, and removing
  its lower guard went unnoticed). A different layout could make the check
  blind; it cannot make it fail.

## The reverse index: `kashi_cp437_rev[128]`

The Unicode reverse index (1.1.0) uses the same unit, counting **entries**:

```cyrius
var kashi_cp437_rev[128];  # 128 entries x one u64 slot (1 KiB): byte 0 = cp >> 8,
                           # byte 1 = cp & 0xFF, byte 2 = cell, bytes 3-7 unused
```

Each entry owns one whole 8-byte slot, addressed `&kashi_cp437_rev + i * 8`
with `load8` / `store8`. Through 1.1.0 it was "the one exception" to the
glyph buffers' N = bytes; since 1.1.1 every buffer in the core counts
slots. Do not "fix" it to `[1024]`: that is 8 KiB.

## Consequence for accessors

Because storage is byte-addressed, `kashi_glyph_ptr` returns a raw byte
address (`&kashi_font16 + (ch - 0x20) * 16`, for a CP437 or extension
cell `ch`) and the consumer reads
`height * stride` consecutive bytes with `load8` — `kashi_glyph_row` returns
the leading byte of a row and `kashi_glyph_row_byte(id, ch, row, b)` the
rest, with `kashi_font_stride(id)` giving the bytes per row (1, 1, 2).
Glyph N and glyph N+1 are exactly `height * stride` bytes apart — a property
the integration tests assert (`test_pointer_monotone`, and
`test_vga_9x16_structural` for the stride-2 buffer); the last cell's last
row byte is the last byte of the buffer ("storage"). If the storage model
ever changed to packed u64s, every accessor and every consumer's read loop
would change with it; that's why this is an architecture note, not just a
comment. 1.1.1 changed only the reservation: symbols, offsets and every
accessor's result are the same.
