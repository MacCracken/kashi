# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [SemVer](https://semver.org/). Through 0.x the
surface was moving; **as of 1.0.0 the public API is frozen** (see
[`docs/api/`](docs/api/) for the full reference and stability promise).

## [Unreleased]

## [1.1.1] - 2026-10-13 — cyrius 6.6.20, and 88 KB back

Patch. The pin moves cyrius 6.6.6 → 6.6.20, and the one issue filed against 1.1.0 is fixed
([`docs/development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md`](docs/development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md),
filed by agnos). The glyph buffers reserved 8× the bytes they use, and that was **88,576 B of zeros in every
binary that links the core**. **No glyph, symbol, signature, offset or accessor result changed.**
- The 678-line glyph sheet still hashes to the 1.1.0 pin `5544d63f…`: on 6.6.6, on 6.6.20, and after the fix.
- The demo's output is byte-identical.
- Bench is flat. Every row is within ±2 % across 1.1.0 on 6.6.6, 1.1.0 on 6.6.20 and 1.1.1 on 6.6.20.

### Fixed

- **The three glyph buffers reserved 8× the bytes they use.**
  - **The bug.** `src/font_data.cyr` declared `var kashi_font16[3616]` / `kashi_font8[1808]` /
    `kashi_font9_16[7232]`, which are the *byte* counts. A module-scope `var X[N]` is N **u64 slots** = 8N bytes,
    so the three reserved 101,248 B for the 12,656 B any code touches.
  - **Why it cost image size.** cycc writes module-scope storage into the file image as zeros, so the slack was
    not free BSS. It was image size in every consumer: in agnos's kernel, 88,592 B of the room under its 0x400000
    ceiling.
  - **The old rationale was wrong.** Architecture note 001 had kept the 8× on purpose: it "mirrors agnos" and
    "`[452]` would under-allocate". At module scope `[452]` *is* 3,616 bytes, and agnos's mirrored declaration was
    gone since its 1.38.0.
  - **Now.** `[452]` / `[226]` / `[904]` slots, exactly the bytes used, named in the internal
    `_KashiFontStore` (`_KASHI_FONT16_SLOTS` …) so the tests check the same numbers the declarations use. An array
    size takes a literal or an enum constant on 6.6.20, not an expression, so `[3616 / 8]` was not an option.
  - **Size** (same toolchain, 1.1.0 → 1.1.1):

    | Build | Toolchain | 1.1.0 | 1.1.1 | Δ |
    |---|---|---|---|---|
    | Library (`src/lib.cyr`) | 6.6.20 | 288,528 B | **199,952 B** | −88,576 B |
    | `CYRIUS_DCE=1` demo | 6.6.20 | 145,376 B | **56,800 B** | −88,576 B |
    | Freestanding `--no-deps` program, core only | agnos's 6.6.6 | 127,312 B | **38,736 B** | −88,576 B |

    agnos measured the same change in its own image: headroom under 0x400000 went 325,952 → 414,544 B
    (+88,592 B). The 16 B difference is alignment (cyrius 6.6.13 aligns globals; see the next item).
  - **For consumers.** Nothing to change. One edge: the buffers now end flush. Reading past the *last* cell's rows
    through a raw `kashi_glyph_ptr` reaches the next variable instead of zeros. No consumer in the stack reads more
    than `height × stride` bytes.
- **A packer guard that no test could see fail.**
  - **The gap.** `src/test.cyr`'s "fset guard" group proved the high-side guards by reading the first *padding*
    byte past each buffer. With no padding that byte is the next variable, so the check is replaced. Rebuilding it
    showed a pre-existing blind spot: nothing kashi owns sat *below* `kashi_font16`. Removing `kashi_fset16`'s lower
    guard (`ch < KASHI_GLYPH_FIRST`) therefore failed nothing, before 1.1.1 and after the right-sizing alike.
  - **The fix.** The guard group now takes one FNV-1a checksum over every byte the core owns, before and after the
    out-of-range barrage: the three used spans, the reverse index, its count and the ready flag. `kashi_cp437_rev`
    is now declared *before* the glyph buffers. Under cycc's declaration-order layout, all four guards' writes then
    land on checksummed bytes. Layout is the compiler's (6.6.13); a different one could blind the check but never
    false-fail it.
  - **New "storage" group** (7 assertions):
    - declared slots × 8 == bytes used, for each buffer. This is `==`, not `<=`, so the old 8× fails too;
    - each buffer's last cell ends its span;
    - the core's six variables are pairwise disjoint, read from the compiled addresses (order-free).
  - **Mutation-tested: 12 mutants, all caught.**
    - each of the four guards removed;
    - the upper guard off by one;
    - each slot count one less;
    - `_KASHI_FONT8_SLOTS` one more;
    - `_KASHI_FONT16_SLOTS` back to 3616;
    - an extension cell added without growing the buffers (four assertions fail);
    - a declaration bypassing the enum.

    Every restore was verified by sha256. Unit suite 448 → **454**.
- **CI's lint step could never fail.** `cyrius lint` exits 0 when it reports warnings. On 6.6.20 a 140-column
  line gives "1 warnings" and rc 0. The step therefore printed findings and passed.
  - It now runs `cyrius lint --exit-with-count --strict-deferrals`, which fails on any warning or untracked
    TODO/FIXME.
  - It walks `tests/` too, as `CONTRIBUTING.md` already claimed it did.
  - Every file was already clean.

### Changed

- **Toolchain: cyrius 6.6.6 → 6.6.20.** The 6.6.7 – 6.6.20 changelog was read for consumer-visible changes. None of
  the new compiler refusals has a shape in `src/`, `tests/` or `docs/examples/`; every file compiles and lints clean
  on 6.6.20. Before the fix, on unchanged source:
  - the glyph sheet was **byte-identical** to 6.6.6's;
  - 448 + 145 assertions passed;
  - `cyrius vet` reported "no dependencies" for all four freestanding files;
  - fuzz was clean;
  - the library grew 262,656 → 288,528 B (+25,872) and the demo 144,072 → 145,376 B (+1,304), from stdlib growth.

  What reached kashi:
  - **6.6.17: a bare `cyrius test` runs `[build].test` (`src/test.cyr`) before `tests/`.** One command now runs
    both suites, 454 + 145. CI's separate `cyrius test src/test.cyr` step is dropped, as the roadmap's 6.6.17 note
    asked.
  - **6.6.9: `cyrius deps` writes a `cyrius.lock` for a stdlib-only project too** ("commit it"). It is now
    **tracked**; until 1.1.1 kashi's policy omitted it.
    - It holds the sha256 of the 24 stdlib files `cyrius deps` vendors.
    - At an unchanged pin, `cyrius deps` refuses a file that differs, with exit 1 (checked with a tampered hash).
      That gives CI an integrity check on the toolchain it fetches with `curl | sh`.
    - After a pin bump it re-locks by itself (checked).
    - dhancha, crab, puka, jalwa, aethersafha and rekha all track theirs.
    - `lib/` is now exactly what `cyrius deps` vendors, byte-identical to the 6.6.20 snapshot. The roadmap's old
      "`cyrius lib sync --full` on the dev box" advice is retired: it vendors all 113 snapshot files, and the next
      `cyrius deps` locks them all — a lock a CI checkout cannot `--verify` (89 failed).
  - **6.6.18: distlib's requires block and compile-verified sidecar.**
    - `dist/kashi.deps` drops from eight leaves to four: `vec`, `alloc`, `string`, `io`. `fmt`, `syscalls`,
      `assert` and `bench` were never the library face's.
    - `dist/kashi.cyr` now opens with the matching `include "lib/…"` lines.
    - A scratch consumer declaring `[deps.kashi] path` + `modules = ["dist/kashi.cyr"]` builds and runs (font init,
      both accessor faces, a loader's error path) on **6.6.20 and on 6.6.6**.
  - **6.6.13: globals are aligned** (16 on x86 for anything over 8 bytes), and "the gap between two globals is the
    compiler's". kashi never reached one global from another. The new storage tests are written not to depend on
    the layout, and architecture note 001 now says so.
  - **6.6.11: test grading on the "N passed, M failed" summary; lint fails on compiler refusals.** Both suites
    print the summary.
- **Bench** (cyrius 6.6.20 vs 6.6.6). Three `CYRIUS_DCE=1` binaries, nine interleaved rounds, medians:

  | Benchmark | 1.1.0 / 6.6.6 | 1.1.0 / 6.6.20 | 1.1.1 / 6.6.20 |
  |---|---|---|---|
  | `glyph_row` | 17.8 ns | 18.2 ns | 18.1 ns |
  | `glyph_ptr` | 7.6 ns | 7.5 ns | 7.5 ns |
  | `scan_vga_8x16` | 63.3 µs | 62.9 µs | 63.0 µs |
  | `font_row_builtin` | 23.9 ns | 24.0 ns | 24.0 ns |
  | `font_row_runtime` | 46.9 ns | 47.1 ns | 47.0 ns |
  | `font_row_runtime_cp` | 63.1 ns | 62.8 ns | 63.2 ns |
  | `cp437_from_unicode_ascii` | 3.8 ns | 3.8 ns | 3.8 ns |
  | `cp437_from_unicode_high` | 26.6 ns | 26.6 ns | 26.7 ns |
  | `font_row_builtin_cp` | 43.3 ns | 43.2 ns | 43.5 ns |
  | `font_init` | 22.4 µs | 22.3 µs | 22.5 µs |

  All flat within ±2 %. The fix changes only the reservation; every access is the same load or store.
  `docs/benchmarks/history.csv` gains the three sessions' rows.
- **Kernel side.** agnos compiles the core with its own pin, 6.6.6. The 1.1.1 core builds freestanding there
  (`cyrius build --no-deps`), passes both suites and renders the identical sheet. kashi did not build agnos.
- **Docs.**
  - Architecture note 001 is rewritten: the invariant is now "N u64 slots = the bytes used", it covers how a new cell
    grows N and why the 8× rationale failed, and it states the storage is in the file image (not BSS) and that layout
    is the compiler's. Its index line is updated.
  - "BSS" is corrected to the buffers' real placement across `src/font_data.cyr`, `SECURITY.md`, the
    getting-started guide and `api/core.md`.
  - Getting started: the add-a-font checklist sizes a new buffer in slots and wires it into the storage tests; the
    build/test block is one `cyrius test`; `tag = "1.1.1"`.
  - `README.md`: status line, bundle leaves, quick start, tag.
  - `CONTRIBUTING.md`: test step, strict lint, the lock.
  - Roadmap: pin procedure (the lock, no `--full`, strict lint, distlib check), the 6.6.6 → 6.6.20 precedent, the
    6.6.17 note marked adopted.
  - Also updated: `api/README.md` (the internal `_KashiFontStore`), ADR 0011's size note, `state.md`, the benchmarks
    doc, the doc-health ledger (`CLAUDE.md` is Fresh again with no edit: its bare-`cyrius test` comment is true on
    6.6.20), and `.gitignore`'s lock policy.
  - The issue file is marked resolved.

## [1.1.0] - 2026-10-09 — Unicode text, and the ellipsis

Additive minor ([ADR 0011](docs/adr/0011-unicode-to-cp437-and-extension-glyphs.md)). The
aethersafha compositor truncates window titles with U+2026 HORIZONTAL ELLIPSIS, which CP437 has no
cell for; and the consumers — the agnos console, aethersafha, dhancha, jalwa, thoth — draw text by
handing CP437 **cells** to `kashi_glyph_row` / `kashi_glyph_ptr`: raw bytes, so UTF-8 renders as
CP437 mojibake ("Ä", `C3 84`, draws `├ä`), unless the consumer carries its own fold (dhancha and thoth
each wrote one). ⚠ puka is the exception: its framebuffer renderer hands **Unicode scalars** to
`kashi_glyph_row` (see *Changed*). kashi had no Unicode → CP437 mapping at all, and the library face's codepoint
accessors had a real bug on the built-ins (below). **No byte-indexed consumer sees a glyph change** (puka is not one — see *Changed*):
every byte of cells `0x20..0xFF` in all three fonts is exactly 1.0.10's — proven by FNV-1a checksums
of the three tables against the 1.0.10 glyph sheet (`tests/kashi.tcyr`), and by the first 672 lines of
the new sheet hashing to the old pin `b68ca4d9…`. No signature, constant value or font id changed.
What every binary that links the core does pay: ~6 KB of size and **+1.2 µs (+6 %) per
`kashi_font_init()` call** — per draw for dhancha, which inits on every bitmap-text draw (Size and
Bench, below).

### Added

- **`kashi_cp437_from_unicode(cp)`** (freestanding core): the cell that draws Unicode codepoint `cp`
  in the built-in fonts, or `0` when none does. ASCII U+0020..U+007E → itself; the 128 codepoints of
  the CP437 high half → `0x80..0xFF` exactly as Python's `cp437` codec maps them (generated from the
  Unicode Consortium's `CP437.TXT`; a second, independently typed table — dhancha's — agrees entry for
  entry): U+00E7 'ç' → `0x87`, U+2500 '─' → `0xC4`, U+00A0 NBSP → `0xFF`; **U+2302 HOUSE → `0x7F`**
  (the glyph kashi's VGA `0x7F` draws) while **U+007F DELETE → `0`** (a control character, like
  C0 / C1 — the ADR records why); U+2026 → `KASHI_EXT_FIRST`, U+FFFD → `KASHI_EXT_FIRST + 1`;
  everything else → `0`. Exactly 226 codepoints in all of Unicode map. ASCII fast path, then a
  binary search (≤ 8 probes) over a 128-entry reverse index. Its source in `kashi_font_init` is
  generated already sorted by codepoint — 16 lines of eight packed `0xCCCCLL` entries (codepoint,
  then cell) — so init writes each entry straight into its own slot: no sort, and a re-init rewrites
  identical bytes (no count reset), so a reader racing a re-init never sees a wrong answer. `load8` /
  `store8` only — `cyrius vet src/font_data.cyr` still reports "no dependencies". Before
  `kashi_font_init` only ASCII, U+2302 and the extension resolve (the high half answers `0`, which
  renders blank).
- **The extension block** — `KashiExtRange`: `KASHI_EXT_FIRST = 0x100`, `KASHI_EXT_LAST = 0x101`,
  `KASHI_EXT_COUNT = 2`. Two kashi-drawn glyphs in all three built-in fonts, stored right after cell
  `0xFF` and reachable **only by codepoint** (no byte value names them): `0x100` U+2026 '…', `0x101`
  U+FFFD '�'. In each font's own weight — the VGA 8×16 '…' is the period's 2×2 dot three times
  (`0xDB` on rows 10–11, the rows '.' uses), the CGA 8×8 '…' the same on its period's rows 5–6; the
  VGA '�' a pointed lozenge with a 5×7 '?' knocked out, the CGA '�' an octagon with a 3×5 '?'; the
  9×16 derives both with column 9 blank. ASCII art in the ADR. The 8-wide '…' fills all eight
  columns — no spacing column, so at an 8-px advance it touches the next glyph on its rows; the ADR
  records the trade-off (it keeps the period's 2×2 weight, kbd's VGA-derived eurlatgr draws the same
  bytes, and both consumers that draw Unicode text advance 9 px). `kashi_glyph_encoded` / `_ptr` /
  `_row` / `_row_byte` accept the two cells. `KASHI_GLYPH_FIRST` / `LAST` / `COUNT` and
  `kashi_font_first` / `kashi_font_count` keep their meaning: they describe the CP437 range, and
  enumerate exactly the 224 CP437 cells as before.
- **Demo**: `cyrius run src/main.cyr` now also draws '…' and '�' through `kashi_cp437_from_unicode`.
- **Tests** (on the 6.6.6 pin a bare `cyrius test` runs only `tests/*.tcyr`, so the full coverage
  lives in the integration suite; CI now also runs `src/test.cyr` — see Changed): `tests/kashi.tcyr`
  57 → **145** — the CP437 round trip cell → Unicode → cell for all 224 cells against the suite's own
  copy of the codec; the reverse index's order checked directly (128 entries, codepoints strictly
  ascending — the binary search's precondition, now that init runs no sort); a sweep of
  U+0000..U+10FFFF (exactly 226 map, every answer an encoded cell); controls, near-misses and i64
  extremes → 0; the 0x7F rule; the extension glyphs' exact rows in all three fonts; extension bounds
  and contiguity; the library face on built-ins across 224 cells × 3 fonts × every row byte; the
  pre-init answers; re-init idempotency (same count, every index byte unchanged); and the FNV-1a
  byte-identity proof.
  `src/test.cyr` 393 → **448** — extension-glyph fidelity bytes, Unicode spot checks, extension
  coverage. `tests/kashi.fcyr` gains `fuzz_unicode` (100,000 `(cp, font, row)` triples; codes 9–11:
  only `0` or an encoded cell, ASCII identity, library face == core of the mapped cell) and its core
  driver now reaches the extension cells (9-bit cells; near-edge picks at `KASHI_EXT_FIRST`,
  `KASHI_EXT_LAST`, `KASHI_EXT_LAST + 1`). **Mutation-tested**, in two rounds. First, 26 mutations
  of the code as first drafted — a corrupted table entry, both directions of the 0x7F rule,
  corrupted rows in the VGA '…', VGA '�' and CGA '�', the range gate reverted and off by one, both
  `fset` guards off by one, the 9×16 walk stopped at `0xFF`, each library accessor reverted to the
  1.0.10 bug, the search skipping its last entry, a broken insertion sort, the fast path and both
  range bounds off by one, U+2026 sent to the replacement cell, one CP437 byte in VGA and in CGA, a
  wrong 9×16 column-9 bit, and `kashi_glyph_row` blind to the extension (the sheet prints
  `!MISMATCH` on all six extension lines). All 24 that change behaviour failed the suites (the fuzz
  harness, on its own, also catches the reverted gate, the fast-path one, the three library reverts
  and the blind `kashi_glyph_row`); 2 — the 9×16 replication range widened to `0xE0` or started at
  `0xB3` — are equivalent: no glyph in those cells has its column-8 pixel set. Two of the 26 (the
  corrupted table entry, the broken insertion sort) targeted the index build that was then replaced;
  the rest
  target code unchanged since, and the suites have only grown. Second, 12 against the final index
  build: two adjacent entries swapped, a line's slot number off by one (overlap plus hole), one entry
  with the wrong cell, a codepoint typo, the slot bound off by one, the cell decoded with the wrong
  mask or stored in the wrong byte, a copy-paste slip in the eight-slot helper, the count set to 127 or
  never set, the last line dropped. All 11 that change behaviour fail `tests/kashi.tcyr` (5 also fail
  the unit suite). The 12th — resetting the count to 0 before the fill, the race the pre-sorted fill
  removed — is equivalent single-threaded (the end state is identical), so that property rests on the
  source's structure and its comment, not on a test. Every restore was verified by sha256.
- **Bench**: `cp437_from_unicode_ascii`, `cp437_from_unicode_high`, `font_row_builtin_cp`, and
  `font_init` — `kashi_font_init()` itself, which is init-time work but not a one-off for every
  consumer.

### Fixed

- **The library face drew the wrong glyph for any non-ASCII codepoint on a built-in font.**
  `kashi_font_ptr` / `kashi_font_row` / `kashi_font_row_byte` are documented as codepoint-addressed for
  every font (ADR 0003), but for the built-ins they passed the codepoint straight into the CP437 table:
  U+00E7 'ç' returned cell `0xE7` 'τ', box-drawing codepoints (U+2500…) were out of range and blank —
  while a runtime PSF font with a Unicode table answered the same codepoints correctly. They now map a
  built-in's codepoint through `kashi_cp437_from_unicode`; an unmapped codepoint returns `0` / null,
  as a runtime font's does. What a caller sees differently on ids 0–2: `0x80..0xFF` resolve by the
  CP437 mapping instead of as raw cells; U+007F goes blank (was the house); U+2302, the CP437
  box-drawing / Greek / maths codepoints, U+2026 and U+FFFD resolve where they were blank. ⚠ **If you
  passed CP437 cells through `kashi_font_*` on purpose, call `kashi_glyph_*` instead** — the core's
  accessors still take the cell, unchanged. No in-tree consumer calls `kashi_font_*` on a built-in
  (all six take the core and pass bytes to `kashi_glyph_row` / `kashi_glyph_ptr`); `src/test.cyr`'s
  own 9×16 dispatch test did, passing cell `0xDB` (which is U+00DB 'Û') where it meant U+2588 '█' —
  corrected.
- ⚠ **The one SemVer-visible behaviour change on a documented input**:
  `kashi_font_row(builtin, 0x7F, row)`, `kashi_font_ptr(builtin, 0x7F)` and
  `kashi_font_row_byte(builtin, 0x7F, row, b)` — `0x7F` sat inside the `0x20..0x7F` range the 1.0.x
  docs (ADR 0003, the `src/lib.cyr` header) gave for built-ins — now return blank / null instead of
  the house glyph, because U+007F is DELETE, a control character with no glyph (ADR 0011, "the 0x7F
  rule"). **To draw the house, pass U+2302**, or read cell `0x7F` with `kashi_glyph_*`. Every other
  input that moves was outside the documented range (`0x80..0xFF` returned the wrong glyph; the rest
  were out of range). Treated as part of the fix, not a break: answering a control character with a
  picture was the same cell-for-codepoint bug. No consumer in the stack calls `kashi_font_*` on a
  built-in.

### Changed

- **The cell accessors' out-of-range bound moved from `0xFF` to `KASHI_EXT_LAST`.**
  `kashi_glyph_encoded(0x100)` / `(0x101)` flip 0 → 1 (and `_ptr` / `_row` / `_row_byte` answer for
  them) — the only two inputs whose answer changes. A caller that fed raw Unicode codepoints to the
  core's cell accessors (already wrong for U+0080..U+00FF) would now see '…' / '�' for U+0100 / U+0101
  instead of blank. ⚠ **puka does**: `fb__glyph_draw` (`puka/src/render/fb.cyr`) passes the grid's
  decoded scalar straight to `kashi_glyph_row`, so on its bump to 1.1.0 'Ā' (U+0100) draws '…' and 'ā'
  (U+0101) draws '�' where both drew blank — and its U+0080..U+00FF range has always drawn the wrong
  CP437 cells ('ç', U+00E7, draws cell 0xE7 'τ'). Migration, which fixes both: map each scalar through
  `kashi_cp437_from_unicode` and pass the cell, skipping a 0. The unit tests that pinned `0x100` as out of range now probe
  `KASHI_EXT_LAST + 1`, and the `fset` guard test now proves the high side too (the first padding
  byte past the used region stays zero). `kashi_fset16` / `kashi_fset8` guard
  `[KASHI_GLYPH_FIRST, KASHI_EXT_LAST]`; the 9×16 derivation walks through the extension.
- **Buffers**: `kashi_font16` / `kashi_font8` / `kashi_font9_16` grow `[3584]` / `[1792]` / `[7168]`
  → `[3616]` / `[1808]` / `[7232]` (226 glyphs), plus the reverse index `kashi_cp437_rev[128]` —
  one u64 slot per entry, 1 KiB, the one buffer in the core not sized in bytes (architecture note
  001 records the exception). **Size**: library 256,592 → **262,656 B**; DCE'd demo 137,936 →
  **144,072 B**, of which +6,064 B is the core (the 1.0.10 demo source against the 1.1.0 core builds
  144,000 B): 1,928 B of new BSS and ~3 KB of code (the map, the index fill, the extension glyphs).
  Every binary that links `src/font_data.cyr` pays about that. **agnos** takes the core by `path`, so
  its next local build carries it: built with agnos HEAD's `scripts/build.sh` on 6.6.6 (`KASHI_DIR`
  pointed at each tree), the kernel image's headroom under the 0x390000 BSP-boot-stack ceiling drops
  99,336 → **93,592 B** (−5,744 B) plain and 45,600 → **39,840 B** (−5,760 B) with `TEST=1 KTEST=1`;
  both still pass its image-layout gate. Its `KASHI_REF` clone fallback (`scripts/build.sh`,
  `test.sh`, `bench.sh`) still names 1.0.10, so a clean CI clone builds without 1.1.0 until that moves
  to the tag.
- **Glyph-sheet pin moved — deliberately, for a format change and new glyphs, not a toolchain
  change.** `docs/examples/glyph_sheet.cyr` appends the extension block after the three fonts' CP437
  lines (cells printed as three hex digits): **672 → 678 lines**, pin `b68ca4d9…` →
  **`5544d63f16e1c4262ce76a5d1f877f56424802040039d8ac560f13213d1d7814`**. The extension comes last on
  purpose: `head -672` of the new sheet is byte-identical to the 1.0.10 sheet and still hashes to
  `b68ca4d9…`.
- **Bench** (cyrius 6.6.6; same-session A/B against the 1.0.10 binary — absolute numbers sat ~5–8 %
  above the 1.0.10 record — medians of 9 interleaved runs on a quiet host; a second, loaded-host
  session agreed on every delta): `glyph_row` 17.2 → 17.7 ns (18.1 → 18.0 in the other session:
  noise), `glyph_ptr` 7.4 → 7.4 ns, `font_row_runtime` 46.6 → 46.5 ns, `font_row_runtime_cp` 62.2 →
  62.3 ns — flat; **`font_row_builtin` 20.7 → 23.7 ns** (the new mapping, paid per row on this
  convenience accessor — resolve once per glyph with `kashi_font_ptr` to pay it once);
  **`font_init` 20.8 → 22.0 µs (+1.2 µs, +6 %)** — ~0.2 µs for the extension glyphs, ~1.0 µs for the
  index fill (a separate 7-run init-only A/B: 1.0.10 21.5 µs, 1.1.0 built without the fill 21.7 µs,
  1.1.0 22.8 µs). The first draft sorted a cell-ordered index at init by insertion: ~43 µs against
  1.0.10's ~22 µs in its own init-only A/B — double. Whatever the figure, dhancha pays it per draw,
  since `dh_draw_text_ink` calls `kashi_font_init()` unconditionally (crab and puka inherit it through
  their vendored `lib/dhancha.cyr`); guarding that call with `kashi_font_is_ready()` would make it
  once. **`scan_vga_8x16` 60.3 → 62.1 µs (+3 %)** — code placement: the scan's only source
  change is the gate's bound constant, and a 1.1.0 build with that constant put back at `0xFF`
  measured the same (63.0 against 63.2 µs in a separate run). New: `cp437_from_unicode_ascii`
  3.9 ns, `cp437_from_unicode_high` 26.3 ns, `font_row_builtin_cp` 42.6 ns.
  `docs/benchmarks/history.csv` gains the 1.1.0 rows and `1.0.10-rerun` rows (`font_init` included)
  for the baseline.
- **CI runs the unit suite.** The test job's comment said a bare `cyrius test` picks up the
  `[build].test` entry; on the 6.6.6 pin it does not (measured: one file, `tests/kashi.tcyr`), so
  `src/test.cyr` — the built-in tables' byte-fidelity pins among its 448 assertions — never ran in CI.
  The job now runs `cyrius test src/test.cyr` as its own step (it exits non-zero on a failed
  assertion — checked by injecting one). From 6.6.17 a bare run covers it (checked in a scratch copy
  pinned to 6.6.17: 448 + 145 passed, two files); the roadmap's pin note says to drop the step then.
- **Docs**: ADR 0011 (mapping, the 0x7F rule, why the extension is codepoint-only, the glyph art and
  the 8-wide ellipsis's spacing trade-off, the index's pre-sorted source and its measured init cost,
  the additive relaxation of the 1.0.0 freeze) with forward pointers from ADR 0003 / 0004 / 0010 and the
  index; `docs/api/core.md` (cells vs codepoints, `kashi_cp437_from_unicode`, `KashiExtRange`,
  stability note), `accessors.md`, `attach.md`, `codes.md` (21 enum groups), `api/README.md` (44
  public functions); README (status, Unicode section, quick start, `tag = "1.1.0"`);
  `getting-started.md` (a "Draw Unicode text" section; the add-a-font checklist's buffer sizes and
  guard); the PSF guide's "built-ins take an ASCII codepoint" notes; architecture note 001 and its
  index; `SECURITY.md`; `state.md`; the roadmap; the doc-health ledger; the `ci.yml` comments on the
  glyph-sheet and test steps; `src/lib.cyr`'s comment that built-ins take an ASCII codepoint, and the
  stale "96 glyphs / ASCII" header in `src/font_data.cyr`. Corrected wherever it appeared: the claim
  that a bare `cyrius test` runs the unit suite (README quick start, `getting-started.md`,
  `CONTRIBUTING.md`, the roadmap's pin procedure — false on 6.6.6), and `getting-started.md`'s stale
  demo description.
- `cyrius.cyml` `[lib]` comment reworded (unreleased since 1.0.10): it said the fold lists the leaf
  modules "NOT `src/lib.cyr`" while the list (correctly) ends with `src/lib.cyr`, and it feared the
  aggregator would carry its `include` lines into the bundle. Measured the opposite: `distlib`
  **strips** `include` lines rather than following them, so a `modules = ["src/lib.cyr"]` fold is
  1,177 lines with no core and no parser (distlib's self-check flags it). The comment now states the
  real rule — every leaf listed explicitly, in include order, aggregator last.
- `dist/kashi.cyr` regenerated (3,985 lines as `cyrius distlib` reports them, 4,004 by `wc -l`;
  `# Version: 1.1.0`); `dist/kashi.deps` unchanged (the same eight stdlib leaves).

### Verification

Every CI step reproduced on the pinned 6.6.6 by running each job's `run:` steps verbatim (`bash -e`)
from `.github/workflows/ci.yml`, in a scratch clone with this tree committed and `git clean -fdx`
between jobs (so each job vendors `lib/` fresh, as a runner does): 19 steps, 0 failed — `cyrius deps`;
library build; the DCE'd demo (ELF magic checked); the glyph-sheet pin (678 lines, no `MISMATCH`,
sha256 = the new pin; `head -672` = `b68ca4d9…`); both test steps (`cyrius test` 145 passed,
`cyrius test src/test.cyr` 448 passed); `cyrius fmt --check` over every file CI walks; `cyrius lint`
over `src/*.cyr` and `docs/examples/*.cyr` (0 warnings); `cyrius vet src/font_data.cyr` → "no
dependencies"; the dist-sync step (bundle tracked, regenerating it is a no-op); `CYRIUS_DCE=1 cyrius
bench tests/kashi.bcyr`; the fuzz job (`CYRIUS_DCE=1` build, 30 s budget — the run completes in well
under a second); the security scan (0 warnings); the docs job's required-file and version-consistency
checks (`VERSION` = `1.1.0`, present in this file).

## [1.0.10] - 2026-09-21

No public-API change. Three gaps found by the 2026-09-21 roadmap review, each closed and proven,
plus the documentation-currency backlog the refreshed `docs/doc-health.md` ledger listed.

### Fixed

- **`dist/kashi.cyr` was never actually published.** 1.0.6 added the `[lib]` fold and called
  `dist/kashi.cyr` "the PUBLISHED library face", but `dist/` was gitignored and no release attached
  it — so a consumer's `cyrius deps` resolving `git` + `tag` + `modules = ["dist/kashi.cyr"]` found
  **no `dist/` at all** (checked against the cached 1.0.6 / 1.0.7 / 1.0.8 clones). The bundle worked
  only through a sibling `path` checkout after a local `cyrius distlib`; four consumers carry a note
  to "switch to `dist/kashi.cyr` the day you need runtime loading" and that day would have failed.
  Now: `dist/kashi.cyr` + `dist/kashi.deps` are **tracked** (`.gitignore` re-includes, the sankoch
  pattern every other bundle-publishing library in the stack uses), **attached to releases** as
  `kashi-<tag>-lib.cyr` / `-lib.deps` with checksums, and **gated** — CI regenerates the bundle and
  fails if either file is untracked or differs; the release refuses a bundle whose `# Version:` is
  not the tag. Proven end to end with a scratch consumer resolving `git` + `tag` against a tagged
  clone: `lib/kashi.cyr` vendored, the sidecar's eight stdlib leaves pulled, linked, and answering
  through both the built-in accessor and a `kashi_register_font` round-trip. The gate was also shown
  to fail on a one-line drift. ADR 0001 carries the addendum.
- **The fuzz harness false-failed on the 9×16 built-in, and never fuzzed the core accessors.**
  `tests/kashi.fcyr`'s known-font list predated `KASHI_FONT_VGA_9X16` (0.5.1), so
  `fuzz_main(2, 'A', 5)` returned FAIL — latent only because `main()` fed the bounds contract two
  canned seeds and nothing random (the random loops covered the parsers; `cyrius fuzz` just runs
  `main()`). The README's "fuzzed over the full integer range of each argument" was aspirational
  for the accessors. Now: id 2 listed; the contract split into `fuzz_core(font, ch, row)` taking
  full-width integers with **eight distinct failure codes** (row not a byte / unencoded char /
  unknown font / null ptr for a valid glyph / out-of-range row not sentinel / leading byte ≠
  `load8(p + row * stride)` / `row_byte` ≠ the row's bytes / `byte_idx` ≥ stride or negative not
  sentinel) — stride-aware, so the 9×16's two-byte rows are checked, not just the 8-wide fonts'; a
  third canned seed for id 2; and a **200,000-triple random driver** over three shapes (byte-sized,
  full-width i64 incl. negatives, near-edge around `0x20` / `0x7F` / `0xFF` and the cell heights).
  **Mutation-tested**: removing the id (with the seed off) → code 3 from the random loop; a
  non-stride contract → caught; and three injected *library* bugs — `kashi_glyph_row` dropping its
  row upper bound → 5, `kashi_glyph_row_byte` accepting `byte_idx == stride` → caught,
  `kashi_glyph_ptr` losing the unknown-font null → 3. Runtime cost of the driver: ~40 ms.

### Added

- **`docs/examples/glyph_sheet.cyr` + `docs/examples/glyph_sheet.sha256` — the glyph-sheet pin.**
  A runnable freestanding-core example (resolves the empty `docs/examples/` open since 0.1.0) that
  dumps all **672** built-in glyphs (3 fonts × 224) as hex, every row byte via
  `kashi_glyph_row_byte` cross-checked against `kashi_glyph_row`. CI builds it, runs it, and
  compares the output's sha256 with the pin (`b68ca4d9…`, the value the 1.0.9 bump measured on
  both toolchains). This is the roadmap's "compare the sheet byte-for-byte" step run on every push:
  a glyph-table edit, a toolchain whose codegen changes what the font says, and a format change in
  the example all move it, and all three must be deliberate. Shown to catch a single flipped bit.
  The example is in the fmt / lint gates too.
- **`tests/kashi.tcyr` — `vga 9x16 structural` group** (+8 assertions, **57** total): stride 2 /
  width 9 / height 16 / count 224; pointers 32 apart; leading byte == the 8×16 row it derives from
  (via both accessors); the VGA col-9 rule holds across `0x20..0xFF` (box-drawing `0xC0..0xDF`
  copies col 8, everything else 0); `byte_idx 2` is the sentinel. The 9×16 had 22 unit assertions
  and none here.
- **Roadmap: a durable "Moving the cyrius pin" procedure** (seven steps, with the 6.6.6 analysis
  kept as precedent) and two new reopen triggers — the pin moving under a bump, and the bundle's
  first real consumer.

### Changed

- **Docs currency** (the `docs/doc-health.md` backlog, now cleared):
  `docs/guides/getting-started.md` rewritten for the frozen 1.0 surface (it had described the
  0.1.0 scaffold since 2026-05-27); `cyaudit vet` → `cyrius vet` across README, CONTRIBUTING,
  SECURITY, `state.md` and the BDF / PCF guides; `architecture/001` + index refreshed to the real
  buffer sizes (`3584` / `1792` / `7168`, 224 glyphs, the 2-byte-stride `kashi_font9_16`);
  `state.md` interior reconciled (sizes 256,592 B / 137,936 B, deps, the six consumers with their
  pins, test counts, the bundle and example); `docs/benchmarks.md` "Current" at 1.0.10 with 1.0.9 /
  1.0.10 rows appended to `history.csv` — and a new structural-shift note: `scan_vga_8x16`
  64 → 56 µs and `font_row_runtime_cp` 66 → 58 ns since 1.0.0 are **toolchain** (6.0.3 → 6.6.6),
  not source, the sheet pin being the proof; `cyrius.cyml` description names all three built-ins
  and the `[deps]` comment stops future-tensing `io`; the function count is **43 public** everywhere
  (README said 45, `api/README.md` 44 — 45 declared minus the two internal `fset` packers; the
  `accessors.md` row said 13 for 14 functions); ADR 0002 / 0004 carry forward pointers to 0006 /
  0007; `api/README.md` and ADR 0001's index entry name the bundle; CLAUDE.md's "booked library
  skeleton" sentence and Quick Start / Work Loop updated for `cyrius distlib` and the pin.
- CI: `docs/examples/*.cyr` added to the fmt and lint loops; new "Glyph sheet pin" step (build job)
  and "Verify dist bundle is in sync with src/" step (lint job). Release: "Verify the published
  bundle" step; `-lib.cyr` / `-lib.deps` assets.
- Bench (1.0.10, cyrius 6.6.6, source unchanged): `glyph_row` 17 ns, `glyph_ptr` 7 ns,
  `scan_vga_8x16` 55.9 µs, `font_row_builtin` 19 ns, `font_row_runtime` 43 ns,
  `font_row_runtime_cp` 58 ns — flat against 1.0.9. Library 256,592 B; DCE'd demo 137,936 B.

## [1.0.9] - 2026-09-21

### Changed

- **Toolchain `6.6.4` → `6.6.6`.** No source change; the public API stays frozen. Ran the
  roadmap's *Moving the cyrius pin to 6.6.6* recipe end to end — `cyrius deps` → build → test →
  `cyrius distlib` — then rendered the full glyph sheet (3 built-in fonts × 224 glyphs, every row
  byte via `kashi_glyph_row_byte`, cross-checked against `kashi_glyph_row`) on **both** toolchains
  and compared: **byte-identical** (sha256 `b68ca4d9…`), so 6.6.6 codegen did not change what the
  font says. `dist/kashi.cyr` regenerated; only its version stamp moved.
  - Neither 6.6.5's tightenings (struct-typed locals under a binary operator, the moved aarch64
    syscall peer) nor 6.6.6's (top-level block `var` scope, one-definition globals, the nine new
    refusals) touch this tree — zero rejections; `cyrlint` still **0 warnings / 0 untracked
    deferrals** across all seven `src/*.cyr`; `cyrius fmt --check` clean. The Windows
    `O_APPEND`/`O_TRUNC` fix is not a kashi exposure (kashi never opens a file for writing — see
    the roadmap section for the full check).
  - **393 unit + 49 integration assertions, 0 failed**; `cyrius vet src/font_data.cyr` → "no
    dependencies"; fuzz harness clean at the 30 s budget.
  - Bench flat within noise (6.6.4 → 6.6.6): `glyph_row` 16 → 16 ns, `glyph_ptr` 6 → 7 ns,
    `scan_vga_8x16` 57.8 → 56.2 µs, `font_row_builtin` 19 → 19 ns, `font_row_runtime`
    43 → 43 ns, `font_row_runtime_cp` 57 → 58 ns.
  - Binaries grow with the 6.6.6 stdlib, not with kashi: library `251,888 → 256,592 B` (+4,704),
    DCE'd demo `137,328 → 137,936 B` (+608).
  - 6.6.5 requires re-vendoring at the bump (its aarch64 peer moved `SYS_UNLINKAT` 35 → 263):
    `cyrius deps` brought the 18-file declared `[deps].stdlib` closure to 6.6.6 and `cyrius lib
    sync --full` the whole snapshot. The ten orphan modules the 1.0.8 note flagged (`agnosys`,
    `base64`, `bigint`, `csv`, `cyml`, `json`, `linalg`, `matrix`, `toml`, `u128` — shipped by no
    installed 6.6.x) are gone from the dev box; the ignored `lib/` is now byte-identical to the
    6.6.6 snapshot. Nothing the repo tracks.

## [1.0.8] - 2026-09-14

### Changed

- **Toolchain `6.6.2` → `6.6.4`**, moving with agnos 1.57.4 (which folds `src/font_data.cyr` into its
  kernel and pins `KASHI_REF` to a tag). No source change; `cyrius test` **49 + 1 passed, 0 failed**;
  `cyrius vet src/font_data.cyr` still reports the freestanding face as dependency-free. ⚠ The
  ignored `lib/` snapshot on the dev box carried ten files no installed toolchain ships (leftovers of
  older pins); `cyrius lib sync --full` refreshes the 110 that 6.6.4 has — a build-box hygiene note,
  nothing the repo tracks.

## [1.0.7] - 2026-09-11

### Changed

- **Toolchain `6.5.27` → `6.6.2`.** No source change. The `Result` value form
  introduced at 6.6.0 does not touch this repo — zero compiler rejections, zero
  fail-open sites, zero arity collisions, and neither of the two `callptr` sites
  reaches a pair-returning target. **49 assertions** pass unchanged.

## [1.0.6] - 2026-08-17 — kashi publishes a library face that can actually be consumed

### Fixed — ⛔ `src/lib.cyr` WAS UNCONSUMABLE BY ANY DOWNSTREAM PROJECT

ADR 0001 splits kashi into a freestanding core for the kernel and a stdlib-using **library face**,
and says plainly that the library face "is what stdlib-linked consumers `include`". No stdlib-linked
consumer has ever managed it. `src/lib.cyr` opens with `include "src/font_data.cyr"` — a path relative
to THIS repo's root — so the moment it is vendored into a consumer's `lib/` the include cannot
resolve:

```
error: cannot open include file: src/font_data.cyr
```

⇒ Every consumer in the desktop stack (dhancha, crab, puka, aethersafha) takes the **freestanding
kernel core** instead. Not by preference — it was the only surface that linked. The ADR's own
recommendation was impossible to follow, and nothing said so.

### Added — `[lib]` + `dist/kashi.cyr`, the published bundle every other stack lib already had

kashi was the ONLY library in the desktop stack with no `[lib]` section and no `dist/`, which is why
consumers reached into `src/` at all. `cyrius distlib` now folds the four leaf modules into
`dist/kashi.cyr` (3,742 lines) with `dist/kashi.deps` beside it.

⚠ **The fold lists the LEAF modules, not `src/lib.cyr`'s includes.** distlib strips include lines, and
`kashi_font_init` appears exactly once in the bundle — verified, because a fold that duplicated the
glyph tables would be worse than the bug it replaces.

### Unchanged — the freestanding core, deliberately

⛔ `src/font_data.cyr` remains the entry point for `[deps] stdlib = []` consumers and is NOT
deprecated. agnos physically cannot link the stdlib the other three modules need. `cyrius vet` still
reports `no dependencies` for it — ADR 0001's invariant is intact.

⚠ **And the desktop-stack consumers should KEEP taking the core, now as a measured choice rather than
the only option.** Switching dhancha to the full face was built and measured: **364,640 → 548,000
bytes, +183,360 (+50%)** for PSF/BDF/PCF import and a runtime font registry it never calls.
`CYRIUS_DCE=1` reclaims **none** of it. They call `kashi_font_init` and `kashi_glyph_row`, both in the
core.

⇒ The bundle exists for consumers that want the runtime surface. Wanting glyph bytes is not that.

## [1.0.5] - 2026-08-17 — toolchain pin to 6.5.27

### Changed — `cyrius = "6.5.5"` -> **6.5.27**

Stack-wide sweep so every repo in the desktop stack declares one toolchain. Pins had drifted across
three lines (6.5.5 / 6.5.20 / 6.5.21) while the installed wrapper was 6.5.27, so every build ran with
a drift warning and the declared graph did not describe what was actually compiled.

⚠ **THE ARTIFACT CHANGED, so this is not a cosmetic edit.** The build went `243424 -> 251840` bytes and the binary differs. The pin is not a comment: it selects the stdlib snapshot under `~/.cyrius/versions/<pin>/lib`, so moving it swaps the library code this repo compiles against.

⛔ **THIS CORRECTS A HALF-TRUTH IN THE 1.0.4 ENTRY**, which reads *"the pin was documentation, not
enforcement — `cyrius build` compiles with the INSTALLED `cycc`"*. That is true **of the compiler** and
it is not the whole mechanism, and believing it was cost a wrong prediction during this sweep: the
bump was expected to be byte-neutral and was not. `cycc` is indeed the installed binary either way —
but the **stdlib** resolves from `~/.cyrius/versions/<pin>/lib`, so the pin decides which library
sources get compiled in. Measured before any other change: the pin bump ALONE moved these bytes.

⚠ The vendored `lib/` was then re-synced to the 6.5.27 bundled set, clearing the
`./lib/ shadows version-pinned` warning. Tests re-run green after both changes.

## [1.0.4] - 2026-08-02

### Changed — cyrius pin 6.4.62 -> 6.5.5

Toolchain catch-up across the whole desktop stack, cut together so the next burn runs binaries built
by ONE compiler rather than 6 different ones.

⚠ **The pin was documentation, not enforcement.** `cyrius build` compiles with the INSTALLED `cycc`,
prints a `toolchain drift` warning, and carries on — so this project was already being built by 6.5.5
before this bump. Verify provenance with `~/.cyrius/versions/<pin>/bin/cyrius` when it matters.

⭐ What the gap actually contained, for a reader deciding whether to care:
- **6.5.1** made overload-suffix arity a hard **error** where it used to warn. Latent arity
  mismatches are now build failures instead of silently-wrong code — good, and the reason this
  sweep surfaced real defects elsewhere in the stack.
- **6.4.75** fixed `fn_table` growth past 8192 silently corrupting six fn-indexed side tables.
- **6.5.0** added file-scoped `private` / per-item `public` — the first real answer to this
  ecosystem's duplicate-`fn`-silently-shadows hazard.
- **6.4.82** completed the agnos GPU syscall wrapper band to `#82`-`#95`, so `sys_gpu_shader_op`
  (#92) and `sys_gpu_modeset_op` (#93) no longer need a raw `syscall()` behind an `#ifdef`.

### Verification

Host + `--agnos` builds green; 1 suite passes.


## [1.0.3] — 2026-07-13

**Toolchain bump.** Pins cyrius `6.4.62` (was `6.2.22`) in
`cyrius.cyml [package].cyrius`. No source changes; the public API
remains frozen per the 1.0.0 stability promise. Rebuilt and
re-tested clean on 6.4.62: 393 unit + 49 integration assertions, 0
failed; `cyrius vet` reports "no dependencies" for all four
freestanding files.

### Changed

- **`cyrius.cyml`**: `cyrius` pin `6.2.22` → `6.4.62`.

## [1.0.2] — 2026-06-18

**Toolchain bump.** Pins cyrius `6.2.22` (was `6.2.2`) in
`cyrius.cyml [package].cyrius`. No source changes; the public API
remains frozen per the 1.0.0 stability promise. Rebuilt and
re-tested clean on 6.2.22: 393 unit + 49 integration assertions, 0
failed; `cyrius vet` reports "no dependencies" for all four
freestanding files.

### Changed

- **`cyrius.cyml`**: `cyrius` pin `6.2.2` → `6.2.22`.

## [1.0.1] — 2026-06-13

**Toolchain bump.** Pins cyrius `6.2.2` (was `6.0.3`) in
`cyrius.cyml [package].cyrius`. No source changes; the public API
remains frozen per the 1.0.0 stability promise. Rebuilt and
re-tested clean on 6.2.2: 393 unit + 49 integration assertions, 0
failed; `cyrius vet` reports "no dependencies" for all four
freestanding files.

### Changed

- **`cyrius.cyml`**: `cyrius` pin `6.0.3` → `6.2.2`.

### Fixed

- **`cyrius vet` workaround retired.** 6.2.2 fixes the 6.0.3
  packaging bug where `cyrius vet` dispatched to a `cybs` that
  emitted an ELF instead of auditing; `cyrius vet src/font_data.cyr`
  now reports "no dependencies" directly. The `cyaudit vet`
  workaround language in `README.md`, `CONTRIBUTING.md`, and the
  CI vet steps is updated accordingly.

## [1.0.0] — 2026-05-28

**Stable release.** kashi's full surface — freestanding core, runtime
loading (PSF / BDF / PCF), sidecar Unicode-table attach, codepoint-
addressed accessors — is frozen. No new features in this cut; this is
the seal after the 0.9.0 API freeze. The full release narrative for
how kashi got here is in the entries below, from 0.1.0 onward.

### What 1.0 commits to

- **API stability** through the 1.x line: no signature changes, no
  removals, no semantic changes to documented behavior. New
  functions and new enum members are additive within 1.x. See
  [`docs/api/README.md`](docs/api/README.md) for the full
  stability promise.
- **Freestanding boundary intact**: `cyaudit vet` reports "no
  dependencies" for each of `src/font_data.cyr`,
  `src/font_psf.cyr`, `src/font_bdf.cyr`, and `src/font_pcf.cyr`.
  The agnos kernel consumes only `src/font_data.cyr`, never the
  library face.
- **Security posture**: three audits in the trail
  (`docs/audit/2026-05-27`, `2026-05-28`, `2026-05-28-audit-0.8.0`);
  the most recent is research-driven against the 2020–2026
  font-parser CVE corpus and produced 9 findings, all fixed.
- **Validation-first parsers**: PSF1/PSF2, BDF, PCF, plus the
  sidecar text-tab parser. All heapless, all dependency-free, all
  fuzzed (~7,500 rounds across the four input surfaces).
- **Test coverage**: 442 assertions across unit + integration
  suites, 0 failed. Fuzz suite: 4000 PSF + 2000 BDF + 1500 PCF +
  1000 text-tab rounds, no crashes.

### Changed (clean-review fixes)

- **`README.md`** rewritten from the 0.1.0 placeholder text into a
  current statement: two faces, three built-in fonts, three runtime
  formats, sidecar attach, hot-path render pattern, link to
  `docs/api/`. Removed stale "future, M1+" and "booked at agnos
  1.38.0" language.
- **`SECURITY.md`** refreshed: buffer sizes updated to the 0.4.0
  CP437 sizes (`kashi_font16[3584]`, etc.); the "M1+ parsers are
  future" language replaced with the current list of audited
  parser modules; the audit trail now lists the three completed
  audits explicitly; supported-versions changed from "pre-1.0" to
  "1.x".
- **`CONTRIBUTING.md`**: replaced `cyrius vet` with `cyaudit vet`
  (the 6.0.3 packaging-bug workaround); included the per-parser
  module dependency-freedom statement; replaced the "once the
  runtime surface lands, sakshi Result" placeholder with the
  actual return-code convention.
- **`src/font_data.cyr`** comments: "agnos, booked at its 1.38.0"
  → "agnos kernel, integrated at agnos 1.38.0"; "CGA 8x8 high
  half is intentionally blank" updated to reflect the 0.5.2 CGA
  high-half fill (ADR 0007); the fset-guard comment generalised
  from "M2 fonts" to "any future built-in font".
- **`src/font_psf.cyr`** scope comment bumped from "as of 0.6.0"
  to "as of 1.0.0".

### Tests + benchmarks

- Test counts unchanged from 0.9.0 (442 assertions, 0 failed; full
  fuzz suite passes). 1.0.0 is a docs / comments cleanup; no code
  paths exercised differently.
- Benchmark refresh: a fresh `1.0.0` row in
  `docs/benchmarks/history.csv`, captured on the same host as the
  0.x line. Numbers match 0.8.0 within noise (no hot-path changes).

### Roadmap

The roadmap closes here. Post-1.0 ideas (BDF lenient-BBX with cell
padding, PCF per-glyph metric variation, text shaping / BiDi as a
separate library) are noted in
[`docs/development/roadmap.md`](docs/development/roadmap.md) as
out-of-scope; none are booked.

## [0.9.0] — 2026-05-28

Public API freeze + every-symbol documentation + benchmark trend
capture. The penultimate cut before **1.0.0** (which is clean
review + bump). **No code changes** — pure docs + benchmark refresh.

### Added

- **`docs/api/` — full public API reference**, frozen as of this cut:
  - `docs/api/README.md` — index + audience guide (kernel devs →
    `core.md`; userland → `loading.md` + `accessors.md`; advanced →
    `parsers.md`; reference → `codes.md`).
  - `docs/api/core.md` — freestanding core
    (`src/font_data.cyr`): 11 functions + 3 font IDs + 3 range
    constants. The agnos-kernel-facing surface.
  - `docs/api/loading.md` — 7 runtime load / register functions
    (`kashi_load_psf{,_file}`, `kashi_load_bdf{,_file}`,
    `kashi_load_pcf{,_file}`, `kashi_register_font`).
  - `docs/api/accessors.md` — 13 codepoint-addressed + raw-index
    accessor and metadata functions, plus the active-font knob
    and ligature lookup.
  - `docs/api/parsers.md` — 7 low-level parser primitives
    (`kashi_psf_parse`, `kashi_psf_uni_token`,
    `kashi_bdf_parse_header`, `kashi_bdf_next_glyph`,
    `kashi_pcf_parse_header`, `kashi_pcf_decode_glyph`,
    `kashi_pcf_cp_to_idx`) + the parsed-header / context / glyph-out
    struct shapes documented field by field.
  - `docs/api/attach.md` — 4 sidecar Unicode-table attach functions
    with the F9 atomic-on-failure guarantee documented.
  - `docs/api/codes.md` — every named constant: result codes
    (`KashiResult`, `KashiBdfRc`, `KashiPcfRc`), font IDs,
    codepoint range, PSF / BDF / PCF format constants, struct
    field offsets, runtime registry layout. ~150 named values.
- **Benchmark trend** — `docs/benchmarks.md` now carries a
  version-over-version table covering 0.1.0 → 0.8.0 (one row per
  released version where the hot path was measured), with
  structural-shift call-outs (0.2.0 → 0.3.0 codepoint-resolve cost,
  0.3.0 → 0.4.0 scan_vga_8x16 workload widening, 0.4.0 → 0.5.0
  stride-aware accessor). Underlying CSV
  (`docs/benchmarks/history.csv`) gains a fresh 0.8.0 row from a
  bench run on the audit-completed surface.

### Changed

- **API surface frozen**: every documented symbol's signature and
  return semantics are stable through 1.0.0 and the 1.x line.
  Internal helpers (anything prefixed with `_kashi_`) remain
  subject to change. `kashi_fset8` / `kashi_fset16` are documented
  as init-time internal (their name omits the underscore prefix
  by historical oversight — listed under "What's NOT public API"
  in `docs/api/README.md`).

### Tests

- Test counts unchanged from 0.8.0 (442 assertions, 0 failed; full
  fuzz suite passes). 0.9.0 is docs-only.

### Roadmap note

- **1.0.0**: clean review + version bump after this cut. No new
  features; final pass on docs and any drift caught by re-reading
  the frozen surface.

## [0.8.0] — 2026-05-28

P(-1) hardening + security audit pass after the 0.7.x import-formats
track closed. Research-driven audit against the 2020–2026 font-parser
CVE corpus (libXfont, FreeType, X.Org advisories, FontForge) produced
a 17-point checklist; 9 findings landed, all fixed. **No exploitable
issues found**; all findings are defense-in-depth tightenings or
robustness fixes. See `docs/audit/2026-05-28-audit-0.8.0.md` for the
full audit report.

### Security

- **F1 (PSF1 unknown mode bits)** — `_kashi_psf1_parse` now rejects
  mode bytes with any of bits 3–7 set. PSF1 defines only bits 0–2
  (MODE512 / MODEHASTAB / MODESEQ); high bits are reserved and must
  be zero. Defense-in-depth (no exploit; rejected inputs were
  previously accepted-but-unused).
- **F2 (PSF2 unknown flags bits)** — `_kashi_psf2_parse` now rejects
  flag words with any of bits 1–31 set. Only bit 0 (HAS_UNICODE) is
  defined.
- **F3 (BDF integer overflow path)** — `_kashi_bdf_int` no longer
  silently truncates at `KASHI_BDF_INT_MAX_DIGITS` (= 10). Hitting
  the cap now returns failure rather than success with a truncated
  value. The pre-fix behavior left leftover digits in the parse
  stream and corrupted the next field's parse; downstream
  `KASHI_BDF_MAX_COUNT` checks caught the corruption so no exploit
  was possible, but the silent-truncate pattern is the same shape
  as CVE-2015-1804 (libXfont BDF metric truncation).
- **F4 (PCF dead code cleanup)** — removed a redundant
  `ref_left = ...loadi8(...) + 128 - 128` assignment in
  `_kashi_pcf_parse_metrics` that was immediately overwritten by the
  canonical `load8 - 0x80` form. Removed the now-orphan
  `_kashi_pcf_loadi8` helper. Cosmetic; no behavior change.
- **F5 / F6 / F7 (PCF format-byte unknown bits)** — each of
  `_kashi_pcf_parse_metrics`, `_kashi_pcf_parse_bitmaps`, and
  `_kashi_pcf_parse_encodings` now rejects format words with bits
  outside their valid set. METRICS allows bits 0–5 + bit 8
  (`PCF_FMT_VALID_METRICS = 0x13F`); BITMAPS and ENCODINGS allow only
  bits 0–5 (`PCF_FMT_VALID_OTHER = 0x3F`). Defense-in-depth per
  CVE-2008-0006 pattern.
- **F8 (PCF duplicate TOC entries)** — `kashi_pcf_parse_header` runs
  a new `_kashi_pcf_has_dup_tables` pre-pass over the TOC before
  any per-type lookup; duplicate `type` entries (which were
  previously silently first-wins) now return `KASHI_PCF_EFORMAT`.
  Same family as CVE-2008-0006 (libXfont PCF TOC handling).
- **F9 (failed `kashi_attach_unicode_table` stripped existing map)** —
  the pre-fix flow cleared the runtime record's umap/seqs *before*
  attempting the new build; if the build produced no entries, the
  function returned `KASHI_EFORMAT` but the prior map had already
  been wiped. Now the previous umap/seqs values are snapshotted
  pre-build and restored on failure (CVE-2015-1803
  stale-state-after-parse-error pattern). The contract is now
  strict: an error return implies "no observable state change."
  This is the only **medium-severity** finding — the others are
  low.

### Tests

- Suite now **442 assertions, 0 failed** (393 unit + 49 integration;
  was 429 at 0.7.2). New: 13 audit regression assertions, one block
  per finding (F1, F2, F3, F5, F6, F7, F8, F9 — F4 is cosmetic with
  no behavior change to test). Each regression patches the exact
  input pattern from the audit and confirms rejection (or, for F9,
  state restoration on the error path).
- The full fuzz suite passes unchanged (4000 PSF + 2000 BDF + 1500
  PCF + 1000 text-tab rounds + seeds, no crashes).

### Audit

- `docs/audit/2026-05-28-audit-0.8.0.md` — full audit report. Walks
  the 17-point CVE-derived checklist against each of the four
  parser files, records the 9 findings with severity and fix
  description, and the 8 checklist items that came up clean (with
  notes on each).
- The 0.7.2 roadmap item "larger table-walk caps" was already
  dropped as a non-issue in ADR 0010; re-confirmed here.

### Roadmap note

- **0.9.0 booked**: public API freeze + every exported symbol
  documented with an example + benchmark trend captured (the M4
  v1.0 criteria from `docs/development/roadmap.md`).
- **1.0.0 booked**: clean review + version bump after 0.9.0.

## [0.7.2] — 2026-05-28

PSF u-variant: sidecar Unicode-table attachment + strict overlong-UTF-8
rejection. Closes the 0.7.x track (BDF → PCF → u-variant); the next
substantial move is M4 / 1.0.0 freeze. See [ADR
0010](docs/adr/0010-psf-u-variant.md).

### Added

- **Sidecar Unicode-table attachment** (`src/lib.cyr`, ADR 0010):
  - `kashi_attach_unicode_table(font_id, buf, len, kind)` — attach a
    raw PSF1 (LE-u16) or PSF2 (UTF-8) Unicode-table byte stream to
    an already-loaded runtime font. Reuses `_kashi_build_umap` +
    `_kashi_build_useqs` with `unioff=0`.
  - `kashi_attach_unicode_table_file(font_id, path, kind)` — file
    companion, 256 KiB cap (`KASHI_TAB_FILE_CAP`).
  - `kashi_attach_unicode_text(font_id, buf, len)` — text-format
    parser for the `psfgettable` convention: `0x<idx> U+<cp> [U+<cp>
    ...] # comment` per line, blank/comment lines skipped, out-of-
    range glyph indices silently dropped.
  - `kashi_attach_unicode_text_file(font_id, path)` — file companion.
  - **Replace semantics**: any existing umap / seqs on the runtime
    record are overwritten (the previous arrays leak-conceptually;
    bump allocator reclaims at process exit). Built-in font ids
    (< `KASHI_RT_FONT_BASE`) return `KASHI_EINVAL` — attach is
    runtime-only.
- **Guide update**: [loading-psf-fonts.md](docs/guides/loading-psf-fonts.md)
  gains a "Sidecar Unicode tables" section covering both formats,
  the `psfgettable` text syntax, and replace semantics.

### Security

- **Strict overlong-UTF-8 rejection** (audit F3, RFC 3629): the
  PSF2 Unicode decoder in `kashi_psf_uni_token` now rejects
  sequences that encode a codepoint with more bytes than necessary
  — e.g., `0xC1 0x81` ("encoding" U+0041 in 2 bytes) is rejected.
  Three minimum-codepoint checks added: 2-byte must produce `cp >=
  0x80`, 3-byte `>= 0x800`, 4-byte `>= 0x10000`. Same posture as
  the 0.6.0 audit F2 tightening (over-U+10FFFF and surrogate
  rejections). No memory safety implication (the codepoint is just
  a u64 map key) — but the canonical-encoding filter belongs in
  the parser by RFC 3629 spirit.

### Tests

- Suite now **429 assertions, 0 failed** (380 unit + 49 integration;
  was 392 at 0.7.1). New: overlong-UTF-8 rejection across the three
  multi-byte lengths (incl. just-above-boundary acceptance for
  U+0080 / U+0800 / U+10000), binary attach round-trip (tableless
  PSF + PSF1 table → codepoint addressing works; pre-attach
  identity-index fallback verified), text attach round-trip
  (multiple codepoints per glyph, comment lines ignored, out-of-
  range indices skipped), attach error cases (built-in id rejected,
  null buf, bad kind, malformed text, empty text), file round-trip
  for the sidecar text path (`/tmp` PSF + `/tmp` .tab via
  `kashi_attach_unicode_text_file`).
- `tests/kashi.fcyr` gains `fuzz_tab` — 1000 rounds across four pick
  paths (random / valid template / mutated template / truncated
  template). Loads a stub PSF once, attaches arbitrary text on each
  round, verifies the parser never crashes and the font remains
  bounds-safe under codepoint probes.

### Notes

- **0.7.x track closed** — BDF (0.7.0), PCF (0.7.1), u-variant
  (0.7.2). The next major move is M4 / 1.0.0 freeze: API audit,
  every public symbol documented with an example, benchmark trend,
  final security audit.
- **Dropped roadmap item**: "larger table-walk caps" was on the
  0.7.2 list but on re-examination found to be a non-issue — the
  table walk is buffer-bounded with no infinite-loop risk (each
  `kashi_psf_uni_token` call advances `pos` or returns `END`).
  Documented in ADR 0010.

## [0.7.1] — 2026-05-28

PCF (Portable Compiled Format) import — X11's compiled binary
bitmap font, the third format on the 0.7.x import-formats track.
New self-contained, heapless parser in `src/font_pcf.cyr`
(`cyaudit vet` → "no dependencies"); library gains `kashi_load_pcf`
/ `kashi_load_pcf_file`. Same posture as the PSF and BDF parsers.
See [ADR 0009](docs/adr/0009-pcf-import.md).

### Added

- **PCF import** (`src/font_pcf.cyr`, ADR 0009):
  - `kashi_pcf_parse_header(buf, len, hdr, ctx)` — validates the 8-byte
    magic + table_count header, walks the table-of-contents, locates
    `PCF_METRICS` / `PCF_BITMAPS` / `PCF_BDF_ENCODINGS` (all required),
    validates METRICS uniformity, computes width/height/glyph_count.
    Fills the 56-byte parsed-header struct (KIND=4 for PCF; PSF1=1,
    PSF2=2, BDF=3) and a 128-byte PCF context with table offsets,
    format flags, and the encoding range.
  - `kashi_pcf_decode_glyph(buf, ctx, glyph_idx, dest, dest_size)` —
    decodes one glyph into a caller buffer. Applies scan-unit byte
    swap (if byte_order != bit_order) + bit-reverse (if bit_order is
    LSB) to canonicalize bitmaps to kashi's MSB-bit / 1-byte-stride
    convention regardless of the source PCF's layout.
  - `kashi_pcf_cp_to_idx(buf, ctx, cp)` — resolves a codepoint to a
    glyph index via the BDF_ENCODINGS table (2D index by `(byte1,
    byte2)`; returns `0 - 1` if unmapped or sentinel `0xFFFF`).
  - `kashi_load_pcf(buf, len)` and `kashi_load_pcf_file(path)` in
    `src/lib.cyr` — drive the parser, decode every glyph in source
    order, walk the encoding range to build the codepoint → glyph
    cp-map, register via the existing runtime registry.
    `kashi_load_pcf_file` reads at most `KASHI_PCF_FILE_CAP = 4 MiB`.
- **Documentation**: [ADR 0009](docs/adr/0009-pcf-import.md) (design
  rationale, alternatives), [guide for loading PCF
  fonts](docs/guides/loading-pcf-fonts.md) (subset accepted, format
  variants, PSF-vs-BDF-vs-PCF picker).

### Scope (0.7.1)

- **Strict uniform metrics** (analog of ADR 0008 strict BBX): every
  glyph's `(left_sb, right_sb, char_width, ascent, descent)` must
  match the first glyph's. Per-glyph kerning rejected with
  `KASHI_EFORMAT`.
- **All four byte-order × bit-order combos** with canonicalize-on-load
  to MSB-bit form. Real PCFs vary by platform (Linux/x86 →
  LSB-byte/MSB-bit; Sparc/Solaris → MSB-byte/MSB-bit; PowerPC →
  MSB-byte/LSB-bit).
- **Both compressed and uncompressed metrics** layouts; dispatch on
  `format & 0x100`.
- **Required tables**: METRICS, BITMAPS, BDF_ENCODINGS. Other table
  types walked past without parsing.
- **Width 1–32, height 1–32, glyph_count ≤ 65,536, TOC ≤ 64 entries**.
- Reuses the 56-byte parsed-header struct, the runtime registry, and
  the codepoint→glyph insertion-sort. No new public result codes.
- **Constraint** for safety: `glyph_pad >= scan_unit` (avoids
  cross-row byte swap pathology). Real PCFs satisfy this.

### Tests

- Suite now **392 assertions, 0 failed** (349 unit + 43 integration;
  was 346 at 0.7.0). New: PCF header parse (kind/dimensions/charsize),
  PCF load round-trip by codepoint (canonical bytes recovered),
  LSB-bit-order variant (bit-reverse path), uncompressed-metrics
  variant, parse-error cases (bad magic, truncated TOC, missing
  required table, non-uniform metrics), PCF file round-trip
  (`/tmp` write + `kashi_load_pcf_file` + glyph verification +
  missing-file negative).
- `tests/kashi.fcyr` extended with `fuzz_pcf` — 1500 rounds across
  four pick paths (random / valid template / mutated template /
  truncated template). PCF is the biggest untrusted-input surface
  kashi has so far; the fuzz harness probes parse_header,
  decode_glyph with random glyph indices, and cp_to_idx with random
  codepoints. No crashes; all accept-paths bounds-safe.

### Notes

- **0.7.2 booked** for PSF u-variant sidecar tables (the remaining
  item on the 0.7.x import-formats track).
- Compressed-metrics support uses the standard +0x80 bias convention
  (byte value - 128 = signed value), matching the X11 spec.

## [0.7.0] — 2026-05-28

BDF (Bitmap Distribution Format) import — the next runtime-loadable
format alongside PSF1/PSF2. New self-contained, heapless text parser
in `src/font_bdf.cyr` (`cyaudit vet` → "no dependencies"); the library
face gains `kashi_load_bdf` / `kashi_load_bdf_file`. The agnos
integration is in (M3 done from kashi's side) — 0.7.x is the
post-integration cut booked for additional bitmap import formats. See
[ADR 0008](docs/adr/0008-bdf-import.md).

### Added

- **BDF import** (`src/font_bdf.cyr`, ADR 0008):
  - `kashi_bdf_parse_header(buf, len, hdr, ctx)` — validation-first
    text parser. Recognises `STARTFONT`, `FONTBOUNDINGBOX`, `CHARS`,
    walks past `STARTPROPERTIES..ENDPROPERTIES`, locates the first
    `STARTCHAR`. Fills the 56-byte parsed-header struct (KIND=3 for
    BDF; the PSF parser uses KIND=1 / KIND=2) plus a 32-byte
    BDF-context struct holding the parsed `FONTBOUNDINGBOX` for
    per-glyph validation.
  - `kashi_bdf_next_glyph(buf, pos, end, ctx, dest, dest_size, gout)`
    — cursor that finds the next `STARTCHAR`, reads `ENCODING` + `BBX`
    (strict-match validation against `ctx`) + `BITMAP` hex rows,
    decodes into the caller's `dest`, advances past `ENDCHAR`. Returns
    OK / SKIP (ENCODING -1) / END kinds.
  - `kashi_load_bdf(buf, len)` and `kashi_load_bdf_file(path)` —
    library entry points. Drive the parser glyph-by-glyph, copy each
    OK glyph into an owned store, build the codepoint→glyph map (same
    shape PSF uses), register via the existing runtime registry.
    `kashi_load_bdf_file` reads at most `KASHI_BDF_FILE_CAP = 4 MiB`
    (vs. PSF's 256 KiB — BDF is ~10× more verbose).
- **Documentation**: [ADR 0008](docs/adr/0008-bdf-import.md) (design
  rationale + alternatives considered), [guide for loading BDF
  fonts](docs/guides/loading-bdf-fonts.md) (subset accepted, strict
  BBX, file-size cap, PSF-vs-BDF picker).

### Scope (0.7.0)

- **Strict BBX**: every glyph's `BBX w h xoff yoff` must match the
  font's `FONTBOUNDINGBOX w h xoff yoff` exactly (covers the console
  bitmap corpus). Per-glyph geometry variation — italic overhangs,
  descender outlines — is out of scope and rejected with
  `KASHI_EFORMAT`.
- **`ENCODING -1`** glyphs (named but unencoded) are dropped entirely
  — not in the glyph store, not in the codepoint map.
- **Width 1–32, height 1–32**, matching the PSF parser's existing
  caps. Glyph count ≤ 65,536 (declared via `CHARS`).
- Reuses the existing 56-byte parsed-header struct shape, the runtime
  registry, the codepoint→glyph map insertion-sort, and the unified
  `kashi_font_row` / `kashi_font_ptr` accessors. No new public
  result codes; no API changes outside the new BDF functions.

### Tests

- Suite now **346 assertions, 0 failed** (312 unit + 34 integration;
  was 305 at 0.6.0). New: BDF header parse (valid + 5 malformed
  cases), BDF load round-trip (codepoint addressing, `ENCODING -1`
  skip verification, mismatched-BBX rejection, CHARS-too-low
  rejection), BDF file round-trip (write a BDF to /tmp,
  `kashi_load_bdf_file`, verify glyph bytes by codepoint).
- `tests/kashi.fcyr` extended with `fuzz_bdf` — 2000 rounds across
  four pick paths (random / valid template / mutated template /
  truncated template). The text-based parser is a larger
  untrusted-input surface than PSF; the fuzz exposed and the cut
  fixed a leading-whitespace infinite-loop in the glyph-block scanner
  (the `p == ls` catcher missed leading-WS lines; replaced with an
  explicit `advanced` flag in both `kashi_bdf_parse_header` and
  `kashi_bdf_next_glyph`).

### Notes

- **agnos integration done** (M3 closed from kashi's side). The
  `[deps.kashi] modules=["src/font_data.cyr"]` contract held; agnos
  consumes the freestanding core unchanged.
- **0.7.x reserved** for the remaining import-format work: 0.7.1 PCF
  (X11 compiled binary) and 0.7.2 PSF u-variant sidecar tables (per
  `docs/development/roadmap.md`).

## [0.6.0] — 2026-05-28

P(-1) hardening pass before opening the 0.6.x patch arc + the M3
agnos-integration runway. Re-audited the substantial post-0.1.0 surface
(PSF parser added 0.2.0; library face grown through 0.5.1; freestanding
core extensions across 0.4.0–0.5.2). One real safety fix, one defensive
tightening, one doc cleanup pass. No behavior change for valid inputs;
no API additions or removals. Freestanding boundary intact —
`cyaudit vet` on both freestanding files → "no dependencies".

### Security

- **`kashi_register_font` `glyph_count` overflow guard** (audit F1): a
  caller passing a pathologically large `glyph_count` could have caused
  `glyph_count * charsize` to overflow `i64`, leading to a too-small
  `alloc` followed by a `memcpy` that walks off the source buffer. Now
  capped at `KASHI_PSF_MAX_COUNT` (65536), matching the PSF parser's bound.
  See `docs/audit/2026-05-28-audit.md`.

### Changed

- **Strict UTF-8 in `kashi_psf_uni_token`** (audit F2): the PSF2 Unicode
  decoder now rejects codepoints above `U+10FFFF` and the UTF-16
  surrogate range `U+D800..U+DFFF` (invalid per UTF-8 spec). No memory
  safety implication (the codepoint is just a map key), but keeps garbage
  out of the codepoint→glyph map.

### Fixed

- **Stale comments** (audit F3): several version-tagged scope notes,
  struct sizes, and return-value docs across `font_psf.cyr` and `lib.cyr`
  had not kept pace with the 0.2.0–0.5.2 evolution. All updated.

### Tests

- Suite now **305 assertions, 0 failed** (281 unit + 24 integration; was
  295 at 0.5.2): F1 `glyph_count` overflow rejected at multiple scales;
  F2 explicit out-of-range / surrogate codepoint sequences (`U+110000`,
  `U+D800`, `U+DFFF`) rejected, just-below-surrogate (`U+D7FF`) and
  exactly-at-max (`U+10FFFF`) accepted.

### Audit

- `docs/audit/2026-05-28-audit.md` — full P(-1) re-walk of the post-0.1.0
  surface (PSF parser, runtime registry, map / sequence build, wide-glyph
  accessors, VGA 9×16 derivation, CGA dual-source high half). Re-verified
  full-`i64`-range bounds-safety for the widened core accessors;
  documents the F4 perf note (double `_kashi_rt_get` on the runtime row
  hot path — no action; consumers should hoist via `kashi_font_ptr`).

### Roadmap note

- **0.6.x reserved for issue-resolution patches** during the M3 agnos
  integration arc.
- **0.7.x** booked for additional bitmap import formats (BDF → PCF →
  PSF u-variant sidecar tables), to land after the initial agnos
  integration. See `docs/development/roadmap.md`.

## [0.5.2] — 2026-05-28

CGA 8×8 high half (`0x80..0xFF`) populated from Linux's public-domain
`lib/fonts/font_8x8.c` — the previously-blank CP437 high half now renders
box-drawing, block shading, accented letters, and the rest of CP437.
Freestanding core stays dependency-free (`cyaudit vet` →
"no dependencies"); kashi's CGA font is now **dual-sourced** (hand-drawn
AGNOS ASCII low half + IBM PD high half) — documented in
[ADR 0007](docs/adr/0007-cga-high-half-from-linux-pd.md).

### Added

- **CGA 8×8 high half (CP437 `0x80..0xFF`)** — 128 glyphs sourced
  byte-for-byte from Linux's `lib/fonts/font_8x8.c`. Mechanically
  generated; fidelity spot-asserts lock the transcription (`0xDB` full
  block, `0xCD` double horizontal, `0xB0` dither, `0xDF` upper half,
  `0xFF` blank).

### Changed

- The "CGA high half intentionally blank" caveat from 0.4.0 is **gone** —
  consumers picking the CGA font now get full CP437 coverage. No API
  change; no behavior change for ASCII consumers (the low half is byte-
  for-byte unchanged).

### Tests

- Suite now **295 assertions, 0 failed** (271 unit + 24 integration; was
  289 at 0.5.1). Removed three "CGA high-half blank" assertions; added
  CGA high-half fidelity asserts.

## [0.5.1] — 2026-05-27

VGA 9×16 box-drawing built-in (derived from the existing VGA 8×16 + the
VGA hardware's col-9 replication rule). The freestanding core gains its
third built-in font without any new byte tables — bytes are computed at
init time from the existing data + the rule. `cyaudit vet` stays
"no dependencies".

### Added

- **`KASHI_FONT_VGA_9X16 = 2`** (ADR 0006) — a 9-pixel-wide built-in font
  in the freestanding core. The 9th column is the well-known VGA
  text-mode rule: for CP437 box-drawing codepoints (`0xC0..0xDF`) col 8
  repeats bit 0 of the underlying 8-wide row, so line characters connect
  cell-to-cell at a 9-pixel cell width; for all other codepoints col 8 is
  blank. Stride 2, height 16, count 224. `kashi_font_init` derives the
  bytes from the existing `font16` once at boot.
- Fidelity tests for the derivation rule (in/out of `0xC0..0xDF`, full
  block extension, double-line characters, off-range boundaries).

### Changed

- **`KASHI_RT_FONT_BASE` bumped `2 → 3`** (pre-1.0 breaking). The new
  built-in occupies id 2, so the runtime registry's id space shifts up by
  one. Consumers that read the symbolic constant are unaffected; anyone
  hard-coding `2` for "first runtime id" needs to update.
- `kashi_font_total()` returns `3 + vec_len(registry)` (was `2 + …`).
- `kashi_glyph_row` is now stride-aware: `load8(base + row * stride)`.
  Unchanged behavior for stride-1 fonts (`row*1 == row`); for the new
  9×16 it returns the LEADING byte of the row, matching the ADR 0005
  documented semantics for wide fonts.

### Tests

- Suite now **289 assertions, 0 failed** (265 unit + 24 integration; was
  267 at 0.5.0). New: VGA 9×16 metadata (width/height/first/count/stride),
  byte 0 matches the existing 8×16 source, col-9 replication for
  box-drawing range, blank col 8 outside range, library dispatcher routes
  the new built-in to the core.

## [0.5.0] — 2026-05-27

Wide-glyph rows + PSF ligature lookup. Additive accessors only; no
breaking change for 8-wide consumers. Freestanding core stays
dependency-free (`cyaudit vet` → "no dependencies").

### Added

- **Multi-byte row access** (ADR 0005): `kashi_font_stride(id)` +
  `kashi_glyph_row_byte(id, ch, row, byte_idx)` in the freestanding core;
  `kashi_rt_font_stride(id)` + `kashi_rt_glyph_row_byte(id, idx, row,
  byte_idx)` + codepoint-addressed `kashi_font_row_byte(id, cp, row,
  byte_idx)` in the library. `byte_idx 0` is the leftmost 8 px; for
  stride-1 fonts only `0` is meaningful (other `byte_idx` returns `0`).
- **Wider PSF2 fonts accepted** — `KASHI_PSF_MAX_WIDTH` rises from `8` to
  `32` (up to 4 bytes/row). `kashi_register_font` accepts width 1–32 and
  copies `count * height * stride` bytes into an owned store.
- **PSF ligature lookup** (ADR 0005): the Unicode-table walk now also
  harvests multi-codepoint sequences (previously parsed-and-skipped).
  Public `kashi_font_seq_glyph(id, cps_ptr, len)` returns the matching
  glyph index, or `0 - 1` if there's no match / no sequence table / bad
  args. kashi still doesn't do shaping — this exposes the *data* a
  shaping layer needs.

### Changed

- PSF parser's strict `charsize == height` rule becomes
  `charsize == height * ceil(width/8)` (correct for any stride; still
  catches mangled fonts).
- Runtime font record grows 56 → 88 B (new `RT_STRIDE`, `RT_SEQS`,
  `RT_SEQCOUNT`, `RT_SEQDATA`).
- Existing single-byte accessors (`kashi_glyph_row`, `kashi_rt_glyph_row`,
  `kashi_font_row`) now return the **leading byte** for wide fonts —
  identical behavior for stride-1, documented for stride > 1.

### Tests

- Suite now **267 assertions, 0 failed** (243 unit + 24 integration; was
  212 at 0.4.0): stride/row_byte for built-ins, 9-wide register +
  multi-byte read, PSF2 width 12 / 32 parse and load, ligature lookup
  (matched / unmatched / wrong length / built-in / null). Fuzz now
  exercises random width 1–32, random `byte_idx`, and seq_glyph probes —
  4000 rounds, bounds-safe.

## [0.4.0] — 2026-05-27

Rest of M2 — registry niceties + extending the freestanding-core glyph
range to full CP437. The freestanding boundary holds (`cyaudit vet` →
"no dependencies"); the agnos consumption contract is widened before
agnos integrates (booked at agnos 1.38.0).

### Added

- **Full CP437 in the VGA 8×16 built-in** — 128 new glyphs for `0x80–0xFF`
  (line/box drawing, block shading, accented letters, Greek/math symbols)
  sourced byte-for-byte from Linux's public-domain `lib/fonts/font_8x16.c`,
  the same PD IBM VGA BIOS table that already supplied the ASCII half.
  Spot-fidelity asserts lock the transcription. See
  [ADR 0004](docs/adr/0004-cp437-glyph-range.md).
- **Active-font knob** (registry nicety) — `kashi_set_active_font(id)` /
  `kashi_active_font()` let a console stash its current selection
  centrally; defaults to `0` (VGA 8×16). Purely additive — the accessors
  don't consult it; the consumer reads it and passes the id along.

### Changed

- **Freestanding addressing range widened** (pre-1.0 breaking):
  `KASHI_GLYPH_LAST` `0x7F → 0xFF`, `KASHI_GLYPH_COUNT` `96 → 224`,
  `KASHI_GLYPH_FIRST` unchanged. Consumers that use the accessors
  (`kashi_font_count`, `kashi_glyph_encoded`) are unaffected; consumers
  that hard-coded `0x7F`/`96` will be off. CGA 8×8's high half is
  intentionally **blank** in 0.4.0 (slots exist, every row reads `0`) —
  extending the hand-drawn font is deferred.
- BSS for the built-ins grows ≈ 2.3× (`var kashi_font16[3584]`,
  `var kashi_font8[1792]`); architecture note 001's `var X[N] = N bytes
  used` shape preserved.
- `tests/kashi.bcyr` `scan_vga_8x16` now sweeps the full 224-glyph range
  (was 96), so the captured bench rises ≈ 2.3× (~26 µs → ~60 µs) — this
  is the new workload, not a regression.

### Tests

- Suite now **212 assertions, 0 failed** (188 unit + 24 integration; was
  186 at 0.3.0): high-half VGA fidelity (`0xDB` block, `0xB0` dither,
  `0xC9` corner, `0xCD` h-line, `0xE9` theta, `0xFF` blank), CGA
  high-half stays blank, active-font set/get round-trip + error cases.

## [0.3.0] — 2026-05-27

M2 (part) — Unicode→glyph mapping so runtime PSF fonts are addressed by
**codepoint** like the built-ins. Library-face + the pure parser only; the
freestanding core stays dependency-free.

### Added

- **Codepoint addressing for runtime fonts** — `kashi_load_psf` now parses
  the PSF1/PSF2 Unicode table into a per-font codepoint→glyph map (sorted,
  binary-searched). `kashi_font_row(id, cp, row)` / `kashi_font_ptr(id, cp)`
  resolve `cp` through that map; an unmapped codepoint renders blank. See
  [ADR 0003](docs/adr/0003-codepoint-addressing-runtime-fonts.md).
- `kashi_rt_glyph_ptr(id, idx)` / `kashi_rt_glyph_row(id, idx, row)` — raw
  glyph-**index** access into a runtime font (tooling / enumeration /
  table-less fonts).
- Pure, bounds-safe table decoder `kashi_psf_uni_token` (PSF1 LE-u16 / PSF2
  UTF-8) in `src/font_psf.cyr`; the Unicode-table walk + map build is fuzzed
  in `tests/kashi.fcyr`. Suite now **186 assertions, 0 failed**.

### Changed

- **`kashi_font_row` / `kashi_font_ptr` runtime semantics: index → codepoint**
  (pre-1.0). For a runtime font that carries a Unicode table, the second
  argument is now a codepoint, not a glyph index. A font *without* a table
  keeps index addressing (identity fallback). Consumers that want explicit
  index addressing should use the new `kashi_rt_glyph_*` accessors.

## [0.2.0] — 2026-05-27

The **M1 PSF font-import path** plus a P(-1) hardening pass. No glyph byte
changed — fidelity vs agnos source is preserved, and the freestanding core
stays dependency-free (`cyrius vet` → "no dependencies").

### Added

- **PSF1/PSF2 font import (M1)** — load runtime bitmap fonts beside the two
  built-ins:
  - `kashi_load_psf(buf, len)` / `kashi_load_psf_file(path)` — parse and
    register a PSF1 (`0x36 0x04`) or PSF2 (`0x72 0xB5 0x4A 0x86`) font;
    validation-first (every header field + length bound checked before any
    glyph read). Returns `font_id ≥ 2` or a negative `0 - <code>`.
  - `kashi_register_font(width, height, glyph_data, glyph_count)` — register
    a raw 8-wide glyph table (bytes copied into an owned store).
  - Unified accessors `kashi_font_row(id, ch, row)` / `kashi_font_ptr(id, ch)`
    dispatch built-in ids (0,1) to the freestanding core and runtime ids
    (≥ 2) to the registry; `kashi_rt_font_{width,height,count}` for runtime
    metadata; `kashi_font_total()` now spans both.
  - New self-contained, heapless parser `src/font_psf.cyr` (fuzzed in
    `tests/kashi.fcyr` — 4000 random/truncated/mutated buffers, no crashes).
  - Scope: width ≤ 8 (one byte/row; wider PSF2 → `KASHI_EFORMAT`); runtime
    glyphs addressed by index, Unicode table validated but not mapped. See
    [ADR 0002](docs/adr/0002-runtime-font-registry.md) and
    [the guide](docs/guides/loading-psf-fonts.md).
- `kashi_font_is_ready()` — reads back the (previously write-only)
  init-complete flag, so a freestanding consumer can assert
  `kashi_font_init()` ran before the first render. Freestanding-safe
  (`cyrius vet` still reports zero dependencies).
- **Benchmark baseline** — `docs/benchmarks.md` + `docs/benchmarks/history.csv`
  (CSV, appended per release for version-over-version tracking). 0.1.0
  accessors: `glyph_row` 17 ns, `glyph_ptr` 7 ns, `scan_vga_8x16` 27 µs
  (x86_64, AMD Ryzen 7 5800H). Unified dispatch adds ~2 ns (built-in path)
  / ~10 ns (runtime path).
- **Security & hardening audit** — `docs/audit/2026-05-27-audit.md` (P(-1)
  pass). Test suite now **146 assertions, 0 failed** (was 81 at 0.1.0):
  full-`i64`-range accessor safety, the `fset` guard, the ready flag, PSF
  parse (valid + malformed), runtime register/load/dispatch, and a PSF
  file round-trip.

### Changed

- **Defensive bound in the table packers** — `kashi_fset16`/`kashi_fset8`
  now ignore an out-of-range codepoint instead of writing at a
  negative/overrun offset, guarding against a typo in a future built-in
  font table (M2) corrupting BSS. No behavior change for in-range data.

### Security

- Audited the freestanding accessor surface: proven memory-safe for the full
  signed-64-bit input domain — no out-of-buffer read or wild dereference for
  any `(font_id, ch, row)`. See `docs/audit/2026-05-27-audit.md`.

### Fixed

- Corrected the VGA 8×16 `0x7F` comment: it is the CP437 triangle glyph
  (matching IBM VGA), not "blank" as the comment claimed (the CGA 8×8 `0x7F`
  is the blank one). Documentation-only; the bytes already matched agnos.

## [0.1.0] — 2026-05-27

Initial baseline. kashi (काशि, "shining") is the AGNOS console-font
subsystem, split out of the agnos kernel's framebuffer console.

### Added

- **Freestanding font-data core** (`src/font_data.cyr`) — pure Cyrius using
  only `store8`/`load8` intrinsics + arithmetic; no stdlib, no heap, no
  syscalls, no sakshi. A freestanding kernel can `include` it directly
  (`cyrius vet` reports zero dependencies). Carries:
  - **IBM VGA BIOS 8×16 ROM font** (`KASHI_FONT_VGA_8X16`, id 0) — public
    domain; the same byte table Linux's `lib/fonts/font_8x16.c` carries.
  - **Hand-drawn CGA-style 8×8 font** (`KASHI_FONT_CGA_8X8`, id 1) — public
    domain AGNOS original; the kernel's original console font.
  - Both cover printable ASCII `0x20`–`0x7F` (96 glyphs).
  - Accessors: `kashi_font_init`, `kashi_glyph_row`, `kashi_glyph_ptr`,
    `kashi_font_width` / `_height` / `_first` / `_count`,
    `kashi_glyph_encoded`. All bounds-safe — out-of-range inputs return safe
    sentinels (row `0`, ptr `0`/null) rather than wild-dereferencing.
- **Library face** (`src/lib.cyr`) — stdlib-using root that re-exports the
  freestanding core and **books** (skeletons) the runtime surface:
  `kashi_load_psf`, `kashi_register_font`, `kashi_font_total`, and the
  `KASHI_OK` / `KASHI_ENOSYS` / `KASHI_EINVAL` / `KASHI_EFORMAT` result
  codes. Booked entries return `KASHI_ENOSYS` until implemented (M1+).
- **Demo binary** (`src/main.cyr`) — renders `'A'` in both fonts as ASCII
  art to eyeball glyph data.
- **Tests** — 81 assertions across `src/test.cyr` (71: metadata, encoded
  gate, exact-byte fidelity vs the agnos source, accessor bounds, full
  96-glyph coverage, library skeleton) and `tests/kashi.tcyr` (10:
  byte-bounded rows, space-vs-visible ink, pointer monotonicity, reinit
  idempotency). Fuzz harness (`tests/kashi.fcyr`) exercises the accessor
  bounds contract over the full integer range of each argument. Benchmarks
  (`tests/kashi.bcyr`): `glyph_row` ~18 ns, `glyph_ptr` ~7 ns,
  `scan_vga_8x16` (96 glyphs × 16 rows) ~27 µs (x86_64 Linux, indicative).
- **Glyph fidelity verified** — both tables were diffed byte-for-byte
  against agnos's source (8×16 from `fb_console.cyr` HEAD; 8×8 from the
  pre-replacement commit): **0 mismatches** across all 96 glyphs in each
  font.
- **Docs** — README (two-faces architecture + agnos consumption contract),
  CLAUDE.md, CONTRIBUTING.md, SECURITY.md, CODE_OF_CONDUCT.md;
  ADR 0001 (freestanding-core split), architecture note 001 (u64-unit BSS
  byte-addressing), roadmap (M0–v1.0), state.md, doc-health.md.

### Notes

- **Consumed by agnos at its 1.38.0** (booked) via `[deps.kashi] path="../kashi",
  modules=["src/font_data.cyr"]` — out of scope for this repo; kashi only
  makes itself consumable.
- The full library face (PSF import, runtime loading, additional fonts) is
  built out along the roadmap — see `docs/development/roadmap.md`.

[Unreleased]: https://github.com/MacCracken/kashi/compare/1.0.2...HEAD
[1.0.2]: https://github.com/MacCracken/kashi/compare/1.0.1...1.0.2
[1.0.1]: https://github.com/MacCracken/kashi/compare/1.0.0...1.0.1
[1.0.0]: https://github.com/MacCracken/kashi/compare/0.9.0...1.0.0
[0.9.0]: https://github.com/MacCracken/kashi/compare/0.8.0...0.9.0
[0.8.0]: https://github.com/MacCracken/kashi/compare/0.7.2...0.8.0
[0.7.2]: https://github.com/MacCracken/kashi/compare/0.7.1...0.7.2
[0.7.1]: https://github.com/MacCracken/kashi/compare/0.7.0...0.7.1
[0.7.0]: https://github.com/MacCracken/kashi/compare/0.6.0...0.7.0
[0.6.0]: https://github.com/MacCracken/kashi/compare/0.5.2...0.6.0
[0.5.2]: https://github.com/MacCracken/kashi/compare/0.5.1...0.5.2
[0.5.1]: https://github.com/MacCracken/kashi/compare/0.5.0...0.5.1
[0.5.0]: https://github.com/MacCracken/kashi/compare/0.4.0...0.5.0
[0.4.0]: https://github.com/MacCracken/kashi/compare/0.3.0...0.4.0
[0.3.0]: https://github.com/MacCracken/kashi/compare/0.2.0...0.3.0
[0.2.0]: https://github.com/MacCracken/kashi/compare/0.1.0...0.2.0
[0.1.0]: https://github.com/MacCracken/kashi/releases/tag/0.1.0
