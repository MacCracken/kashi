# 0011 — Unicode → CP437 cell mapping, and an extension block for U+2026 / U+FFFD

**Status**: Accepted
**Date**: 2026-10-09
**Refines**: [ADR 0003](0003-codepoint-addressing-runtime-fonts.md) (what the
library face's codepoint means for a **built-in** font) and
[ADR 0004](0004-cp437-glyph-range.md) (the freestanding cell space grows past
`0xFF`, additively — `0x20..0xFF` keeps its meaning). Also corrects the
built-in sentence in [ADR 0010](0010-psf-u-variant.md)'s Consequences.

## Context

Every consumer in the stack — the agnos kernel console, and the aethersafha
compositor, dhancha, jalwa and thoth on the desktop side — draws text by
handing **bytes** to `kashi_glyph_row` / `kashi_glyph_ptr`, which index kashi's
CP437 cell table. (puka differs: its framebuffer renderer passes decoded Unicode
**scalars** as if they were cells, which is right only for ASCII.) Two problems
follow, and kashi owned neither answer:

1. **UTF-8 text renders as CP437 mojibake.** "Ä" is the bytes `C3 84`, which
   draw as `├` then `ä`. dhancha and thoth each grew a private Unicode→CP437
   fold to cope; the others did not. kashi had **no** Unicode→CP437 mapping.
2. **There is no ellipsis.** The compositor truncates window titles with
   U+2026 HORIZONTAL ELLIPSIS (its Rust original did; aethersafha's roadmap
   carries it), and CP437 has no cell for it. Nor for U+FFFD REPLACEMENT
   CHARACTER, the conventional "this could not be drawn" mark.

And one real bug: the library face's unified accessors are documented as
**codepoint**-addressed for every font (ADR 0003), but for the built-ins they
passed the codepoint straight into the CP437 table. Anything past ASCII came
back wrong — U+00E7 'ç' returned cell `0xE7`, which is 'τ'; U+2500 '─' was
out of range and blank — while a runtime PSF font with a Unicode table answered
the same codepoints correctly. The 1.0 comment papered over it ("built-ins:
`cp` is an ASCII codepoint").

Constraints that shape the answer:

- **Byte-indexed consumers must see no change.** Cells `0x20..0xFF` are what
  agnos renders and what the glyph-sheet pin (`b68ca4d9…`) records; not one
  byte of them may move, and no CP437 code position may change meaning.
- **The freestanding boundary (ADR 0001).** The kernel needs the mapping too,
  so it lives in `src/font_data.cyr`: `store8`/`load8` + arithmetic, no
  include, no heap, `cyrius vet` → "no dependencies".
- **Both faces have been frozen since 1.0.0**: no signature, constant-value or
  removal changes inside 1.x; additions only.

## Decision

**Add `kashi_cp437_from_unicode(cp)` to the freestanding core, add an extension
block of two kashi-drawn glyphs at cells `0x100..0x101`, and make the library
face map built-in codepoints through the new function.**

### 1. `kashi_cp437_from_unicode(cp)` → cell, or 0

A Unicode codepoint becomes the **cell** that draws it in the built-in fonts —
the value `kashi_glyph_ptr` / `kashi_glyph_row` / `kashi_glyph_row_byte` take —
or `0` when no built-in glyph exists. It is font-independent: it names a cell,
and every built-in font has every cell.

| Codepoint(s) | Cell |
|---|---|
| U+0020..U+007E (printable ASCII) | itself |
| the 128 codepoints of the CP437 high half | `0x80..0xFF`, per the standard CP437 mapping (U+00E7 'ç' → `0x87`, U+2500 '─' → `0xC4`, U+00A0 NBSP → `0xFF`) |
| U+2302 HOUSE '⌂' | `0x7F` |
| U+007F DELETE | `0` (see *the 0x7F rule*) |
| U+2026 HORIZONTAL ELLIPSIS | `KASHI_EXT_FIRST` (`0x100`) |
| U+FFFD REPLACEMENT CHARACTER | `KASHI_EXT_FIRST + 1` (`0x101`) |
| everything else — C0 / C1 controls, the codepoints of CP437's `0x01..0x1F` pictures (☺ ♥ …, cells kashi does not encode), letters CP437 lacks, negatives, values past U+10FFFF | `0` |

That is exactly **226** mapped codepoints (95 + 128 + 1 + 2); the test suite
proves the count by sweeping U+0000..U+10FFFF. `0` is below
`KASHI_GLYPH_FIRST`, so every accessor already answers it with the blank / null
sentinel; a consumer that wants a visible fallback substitutes
`kashi_cp437_from_unicode(0xFFFD)`.

**The high half's source** is Python's `cp437` codec, which is generated from
the Unicode Consortium's `VENDORS/MICSFT/PC/CP437.TXT`. A second, independent
oracle — dhancha's hand-typed table — agrees entry for entry. The kashi table is
generated mechanically from the codec; `tests/kashi.tcyr` carries its own copy
and checks the round trip cell → Unicode → cell for all 224 cells. **No
aliases** beyond the codec (e.g. Linux's consolemap also sends U+03B2 'β' to
`0xE1` 'ß'): the function stays one-to-one and checkable against an external
oracle; aliases can be added later without breaking anyone.

**Implementation.** An ASCII fast path (two compares), the four fixed cases,
two range rejections (`< U+00A0`, `> U+25A0` — the high half's bounds), then a
binary search over a 128-entry reverse index `{codepoint, cell}`.
Allocation-free, `load8` only, at most 8 probes. The index is one u64 slot per
entry (`var kashi_cp437_rev[128]` = 1 KiB — the one buffer in the core not
sized in bytes, and commented as such). Its source in `kashi_font_init` is
**already sorted by codepoint** — 16 lines of eight packed `0xCCCCLL` entries
(codepoint, then cell), each line naming its slots — so init writes every
entry straight into its own slot and runs no sort; the count is set once the
slots are full and never reset. Two things follow. Init stays cheap (the fill
measures ~1.0 µs; sorting a cell-ordered list at init by insertion measured
~21 µs, doubling `kashi_font_init`, which dhancha calls on every bitmap-text
draw). And a re-init rewrites each byte with the value already there, so a
reader racing a **re**-init never sees a wrong answer — the property the glyph
tables already had (the *first* init must still finish before another CPU
reads; kashi has no synchronisation). `tests/kashi.tcyr` checks the order
directly (codepoints strictly ascending) and every entry against its own
cell-ordered copy of the code page.
Before `kashi_font_init` the index is empty: ASCII, U+2302 and the extension
still resolve and the high half answers `0`, which renders blank — the same
"safe but blank" state every glyph is in before init.

### The 0x7F rule

The codec maps byte `0x7F` to U+007F DELETE: CP437 is ASCII below `0x80`, and
the codec does not model IBM's pictures for the control positions. But kashi's
VGA cell `0x7F` is **drawn** as IBM's house '⌂' (U+2302), the glyph the ROM
puts there. A codepoint API answers "which glyph draws this character". DELETE
is a control character and has none — exactly like U+0000..U+001F and
U+0080..U+009F, which already map to nothing — while '⌂' is a real character
whose glyph lives at `0x7F`. So:

- **U+2302 → `0x7F`** — the house becomes reachable by its own codepoint
  (it was unreachable before; the library face had no codepoint for it).
- **U+007F → `0`** — no glyph.

Consequences, stated rather than discovered later: the library face's
`kashi_font_row(builtin, 0x7F, row)` changes from the house to blank (the 1.0
comment listed `0x7F` in its "ASCII" range, so this is the one documented input
whose answer moves — it moves because answering a control character with a
picture was the same cell-for-codepoint bug); byte-indexed reads of cell `0x7F`
are unchanged. The **CGA 8×8 draws cell `0x7F` blank** (its hand-drawn half
always had DEL blank — audit 2026-05-27, F3), so '⌂' renders blank in that font:
the mapping names the cell, coverage is per font, and `0x7F`'s bytes are frozen.

### 2. The extension block

| Constant | Value | |
|---|---|---|
| `KASHI_EXT_FIRST` | `0x100` | `== KASHI_GLYPH_LAST + 1` |
| `KASHI_EXT_LAST` | `0x101` | |
| `KASHI_EXT_COUNT` | `2` | |

Cell order: `KASHI_EXT_FIRST + 0` = U+2026, `+ 1` = U+FFFD. Every built-in
font stores the two glyphs **directly after cell `0xFF`** (the buffers grow
224 → 226 glyphs), so the existing `(ch - KASHI_GLYPH_FIRST) * size` offset
reaches them unchanged and `kashi_glyph_encoded` needs one new bound.
`kashi_glyph_encoded` / `kashi_glyph_ptr` / `kashi_glyph_row` /
`kashi_glyph_row_byte` accept the two cells; `kashi_fset16` / `kashi_fset8`
guard `[KASHI_GLYPH_FIRST, KASHI_EXT_LAST]`; the VGA 9×16 derivation walks
through them (outside `0xC0..0xDF`, so column 9 is blank).
**`KASHI_GLYPH_FIRST` / `LAST` / `COUNT` and `kashi_font_first` /
`kashi_font_count` keep their meaning** — they describe the CP437 range, and
enumerating `first .. first + count` still yields exactly the 224 CP437 cells.

### Why the extension is reachable only by codepoint

- **No byte can name it, by construction.** All 256 byte values are CP437 code
  positions — `0x00..0x1F` too: kashi does not draw them, but they are CP437's
  (☺ is `0x01`). Putting '…' in any of them would change what some
  byte-indexed consumer draws for that byte, and kashi's "CP437" would stop
  being CP437. Cells ≥ `0x100` cannot be produced from a byte, so the kernel,
  a CP437 art file, or raw UTF-8 bytes fed to `kashi_glyph_row` can never land
  on one by accident.
- **The cell numbers are kashi's, not an encoding.** No standard says `0x100`
  is '…'. The name of the ellipsis is U+2026; `kashi_cp437_from_unicode(0x2026)`
  is how a caller gets its cell, so the block can grow (more cells after
  `0x101`) without anyone having spelled a number. That is also why there are
  **no per-glyph constants** (`KASHI_EXT_ELLIPSIS`): `KASHI_EXT_FIRST` /
  `LAST` / `COUNT` exist for bounds and enumeration (the glyph sheet, the fuzz
  harness), not to name glyphs.
- **The two numbering spaces do not leak.** The library face's codepoint
  accessors never take a cell: U+0100 'Ā' is unmapped there (blank / null) even
  though core cell `0x100` is '…'.

### 3. Library face: built-ins take a Unicode codepoint

`kashi_font_ptr` / `kashi_font_row` / `kashi_font_row_byte` route a built-in id
through `kashi_glyph_*(font_id, kashi_cp437_from_unicode(cp), …)`. An unmapped
codepoint returns `0` / null — the same answer a runtime font gives for a
codepoint its map lacks. `kashi_font_seq_glyph` is unchanged (built-ins have no
sequences). Callers that want the CP437 **cell** — byte-indexed text, CP437 art
— keep calling the core's `kashi_glyph_*`, which still take the cell.

### 4. The glyph designs

Not from any ROM: kashi-drawn, in each font's own weight. `#` = set pixel.

**VGA 8×16.** The ellipsis is three copies of the period's dot — the VGA '.'
(`0x2E`) is a 2×2 dot `0x18` on rows 10–11 — packed into one cell with 1-px
gaps: `0xDB` on rows 10–11, nothing else. U+FFFD is a pointed lozenge
(cols 0–6, rows 1–13, centred on row 7 like the ROM's diamonds) with a standard
5×7 '?' knocked out of its body (rows 4–10); col 7 stays blank, as in the ROM's
letters.

```
row  '.' 0x2E   U+2026     U+FFFD
 0   ........   ........   ........
 1   ........   ........   ...#....
 2   ........   ........   ..###...
 3   ........   ........   .#####..
 4   ........   ........   ##...##.
 5   ........   ........   #.###.#.
 6   ........   ........   #####.#.
 7   ........   ........   ####.##.
 8   ........   ........   ###.###.
 9   ........   ........   #######.
10   ...##...   ##.##.##   ###.###.
11   ...##...   ##.##.##   .#####..
12   ........   ........   ..###...
13   ........   ........   ...#....
14   ........   ........   ........
15   ........   ........   ........
```

`kashi_fset16(0x100, 0x0000000000000000, 0x0000DBDB00000000)`,
`kashi_fset16(0x101, 0x0010387CC6BAFAF6, 0xEEFEEE7C38100000)`.

**CGA 8×8.** The CGA period sits on rows 5–6, so the ellipsis is `0xDB` on rows
5–6. U+FFFD is an octagon in the CGA letter box (cols 0–6, rows 0–6; row 7
blank like the letters) with a 3×5 '?' knocked out. Eight rows cannot hold a
pointed diamond *and* a readable '?': in a 7×8 diamond the '?' dot lands in a
3-pixel tip row, where it reads as a notch.

```
row  '.' 0x2E   U+2026     U+FFFD
 0   ........   ........   ..###...
 1   ........   ........   .#...#..
 2   ........   ........   ####.##.
 3   ........   ........   ###.###.
 4   ........   ........   #######.
 5   ...##...   ##.##.##   .##.##..
 6   ...##...   ##.##.##   ..###...
 7   ........   ........   ........
```

`kashi_fset8(0x100, 0x0000000000DBDB00)`, `kashi_fset8(0x101, 0x3844F6EEFE6C3800)`.

**VGA 9×16.** Derived per ADR 0006: column 9 is blank for both (the extension
is outside `0xC0..0xDF`), so the ellipsis reads `##.##.##.` with the cell's
spacing column intact.

**The 8-wide ellipsis has no spacing column — a recorded trade-off.** The ROM
leaves column 7 blank in its letters as the gap between characters (in VGA
printable ASCII only `*` and `_` ink it, besides the box-drawing cells that
are meant to join; the CGA's hand-drawn half is looser — `*/1QR_m` ink it
too). Three 2-px dots with 1-px gaps need all eight columns, so
in VGA 8×16 and CGA 8×8 the ellipsis inks column 7, and at an 8-px advance it
touches whatever follows on its rows: `……` draws `##.##.####.##.##` (two dots
merge into a 4-px bar), and a following glyph with column-0 ink on those rows
joins it on the baseline — in VGA 8×16 (rows 10–11)
`$%&/23568@ABDEFHJKLMNOPQRSUWXZacdeghkmoqrsuwxyz`, in CGA 8×8 (rows 5–6)
`$%&/023568=ABDHJKMNOQRSUWXZacdemosuwx{`. Kept anyway:

- **It keeps the period's weight.** Every design that frees column 7 shrinks
  a dot: 1-px dots (`#..#..#.` — Terminus, whose own period is 1 px wide;
  `.#..#..#` — LatGrkCyr-8x16, which inks column 7 anyway) or a thin middle
  dot (`##.#.##.` — pancyrillic.f16). Beside VGA text, whose period is 2×2,
  those read lighter than '.'.
- **It is what the VGA lineage already draws.** Of the console fonts in kbd
  that are 8 px wide and carry U+2026 (checked 2026-10-09: four, all 8×16),
  eurlatgr — built from latarcyrheb-sun16, the Fedora / RHEL default console
  font, with the same 2×2 period on rows 10–11 — has exactly `0xDB` on rows
  10–11. The other three are the designs quoted above.
- **No consumer in the stack meets the case.** The agnos console is
  byte-indexed and cannot reach cell `0x100`. The two that draw Unicode text
  through kashi — the aethersafha compositor and dhancha — advance 9 px per
  glyph (8 px plus a 1-px gap), so column 7 never meets a neighbour there,
  and the compositor's '…' is the last cell of a truncated title anyway.

⚠ Decide before tagging, not after: `docs/api/core.md` promises that existing
extension cells keep their glyphs, so once a release carries this '…',
redrawing it is a stated exception to that promise, not a tweak.

### 5. The 1.0.0 freeze, relaxed additively

- **Added**: one function (`kashi_cp437_from_unicode`), one enum group
  (`KashiExtRange`, three constants), two cells in each of the three built-in
  fonts. No signature, constant value, font id or `KASHI_RT_FONT_BASE` changes.
- **Not changed**: every byte of cells `0x20..0xFF` in all three fonts — proven
  by FNV-1a checksums against the 1.0.10 sheet in `tests/kashi.tcyr`, and by
  the first 672 lines of the new glyph sheet hashing to the 1.0.x pin
  `b68ca4d9…`.
- **Behaviour that does change, deliberately**: (a) the core's cell accessors
  answer for cells `0x100` / `0x101`, which were out of range (sentinels);
  (b) the library face on built-in ids reads `cp` as Unicode: `0x80..0xFF`
  now resolve by the CP437 mapping instead of as raw cells (the bug), U+007F
  goes blank, and U+2302, the box-drawing / Greek / maths codepoints, U+2026 and
  U+FFFD resolve where they were blank. A caller that was passing CP437 cells
  through `kashi_font_*` switches to `kashi_glyph_*` (cells) or passes the
  Unicode codepoint.
- **Version**: a minor, 1.1.0 — new API plus a bug fix, nothing removed.

## Consequences

- **Positive** — one owned, tested Unicode→CP437 fold, available to the kernel
  and to every byte-indexed consumer; dhancha's and thoth's private folds can
  retire onto it. The compositor gets a real '…' and every renderer a real '�'.
  The library face now means the same thing by "codepoint" for built-in and
  runtime fonts.
- **Positive** — the freestanding boundary holds (`cyrius vet` → "no
  dependencies"), and byte-indexed consumers are byte-for-byte unaffected.
- **Negative** — a caller that passes Unicode scalars to the cell accessors sees
  U+0100 / U+0101 change from blank to '…' / '�'. puka's `fb__glyph_draw`
  (`puka/src/render/fb.cyr`) is one; it already drew U+0080..U+00FF as the wrong
  CP437 cells. Mapping each scalar through `kashi_cp437_from_unicode` before
  `kashi_glyph_row` fixes both, and is puka's migration when it moves its tag.
- **Negative** — a per-glyph mapping cost on Unicode text paths: ~4 ns for
  ASCII, ~26 ns for a high-half codepoint (binary search); `kashi_font_row` on a
  built-in pays it per **row** call (+3 ns ASCII), so a render loop should
  resolve once per glyph (`kashi_font_ptr`, or `kashi_cp437_from_unicode` +
  `kashi_glyph_ptr`) and read rows with `load8`. Numbers in
  `docs/benchmarks.md`.
- **Negative** — `kashi_font_init` costs **+1.2 µs (+6 %)**: 20.8 → 22.0 µs
  in a same-session A/B against 1.0.10 (medians of 9 interleaved runs, the
  `font_init` bench), ~0.2 µs for the extension glyphs and ~1.0 µs for the
  index fill. That is per *draw* for dhancha, whose `dh_draw_text_ink` calls
  `kashi_font_init()` unconditionally (crab and puka inherit it through their
  vendored `lib/dhancha.cyr`); guarding the call with `kashi_font_is_ready()`
  would make it once.
- **Negative** — every binary that links the core grows by ~6 KB: 1,928 B of
  BSS (2 glyphs × 3 fonts at the u64-unit reservation, plus the 1 KiB index
  and its count) and ~3 KB of code (the map, the index fill, the extension
  glyphs). Measured: DCE'd demo 137,936 → 144,000 B with the demo source
  unchanged. In agnos, which takes the core by `path`, the kernel image built
  by its `scripts/build.sh` on 6.6.6 keeps less headroom under the 0x390000
  BSP-boot-stack ceiling: 99,336 → 93,592 B (−5,744 B) plain, 45,600 →
  39,840 B (−5,760 B) with `TEST=1 KTEST=1`. Both still pass its image-layout
  gate.
- **Negative** — two numbering spaces to keep straight: **cells** (core
  accessors, `0x20..0x101`) and **codepoints** (`kashi_cp437_from_unicode`,
  the library face). Documented at the top of `src/font_data.cyr` and in
  `docs/api/core.md`.
- **Neutral** — `kashi_glyph_encoded(0x100)` and `(0x101)` flip from 0 to 1. A
  caller that fed raw Unicode codepoints to the core's cell accessors (already
  wrong for U+0080..U+00FF) now gets '…' / '�' for U+0100 / U+0101 instead of
  blank. No in-tree consumer does this — they pass bytes.
- **Neutral** — CP437's `0x01..0x1F` pictures (☺ ♥ ♪ ► …) remain unencoded,
  so their codepoints map to `0`. They are natural candidates for further
  extension cells, additively, when a consumer asks.

## Alternatives considered

- **Map in the library face only** (stdlib side). Rejected: the kernel and all
  six desktop consumers take the freestanding core — they are the ones drawing
  mojibake.
- **Put '…' in an existing cell** (`0xFF` NBSP, or one of `0x00..0x1F`).
  Rejected: changes what a byte draws, and stops kashi's CP437 being CP437.
- **Extension cells past U+10FFFF** (e.g. `0x110000`), so no codepoint can
  collide with a cell. Rejected: needs a second offset formula and a second
  bound in every accessor on the kernel's per-row hot path, to protect only a
  usage that is already wrong; and `docs/api/core.md` already promised the cell
  range could widen past `0xFF`.
- **U+007F → `0x7F` as well** (keep the 1.0 library-face answer at `0x7F`).
  Rejected: it draws a picture for a control character, contradicts the C0 /
  C1 rule, and breaks "`0` = no glyph". The 1.0 answer was a symptom of the
  cell-for-codepoint bug, not a design.
- **A faster lookup** — a direct table over U+00A0..U+25A0 (~9.5 KB, ×8 at
  module scope), per-page tables, or a perfect hash. Rejected for now: the
  binary search is ~26 ns against ~18 ns per glyph *row*, the table is 1 KiB,
  and the code is obvious. Revisit with a measurement if a consumer's profile
  says otherwise.
- **Sort the index at init** from a cell-ordered source — sixteen lines that
  read like the code-page chart, insertion-sorted into place by
  `kashi_font_init`. Built, measured and rejected: the sort's entry moves cost
  ~21 µs per call, **doubling** `kashi_font_init` (~21 → ~43 µs), on a function
  dhancha runs per draw; and resetting the count before refilling made a
  re-init briefly answer `0` (or match a half-shifted entry) to a racing
  reader. The generator sorts once instead, and the chart order survives as
  `tests/kashi.tcyr`'s independent oracle. Writing the 128 entries one per
  call (no packing) was also measured: the same speed, ~900 B more code.
- **An ellipsis that keeps column 7 blank** (1-px dots, or a thin middle dot).
  Not taken — see the trade-off under §4.
- **Lazy initialisation** inside `kashi_cp437_from_unicode`. Rejected: a hidden
  side effect on a hot path in kernel context; the core's model is one explicit
  `kashi_font_init` (ADR 0001).
- **Named per-glyph constants** (`KASHI_EXT_ELLIPSIS = 0x100`). Rejected: turns
  kashi's internal numbering into an API; the codepoint is the name.
- **Aliases beyond the codec** (β → `0xE1`, Ω U+2126 → `0xEA`, …). Deferred:
  one-to-one against an external oracle first; additive later.
