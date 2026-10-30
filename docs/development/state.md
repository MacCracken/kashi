# kashi — Current State

> **Last refresh**: 2026-10-29 (**1.2.2** — the `kashi_pcf_cp_to_idx`
> fix and the stale source comments; cyrius 6.7.18) |
> **Refresh cadence**: bumped every release.
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**1.2.2** — Patch. Fixes the bug the 1.2.1 sweep found, plus its list of
stale source comments. The toolchain is unchanged (cyrius 6.7.18).
- **Fixed: `kashi_pcf_cp_to_idx` above U+FFFF.** byte1 was masked to 8
  bits, so a codepoint above U+FFFF wrapped into the BMP (`0x10041` →
  U+0041's glyph). It now returns `0 - 1`. Only direct callers of the
  parser saw this; the PCF loader passes in-range codepoints only.
- **Tests.** A new unit group "pcf cp_to_idx" (unit 454 → **464**), and
  fuzz code 11 in the PCF contract, probed from the font's own encoded
  range. With the guard removed, the unit suite fails 3 assertions and
  fuzz exits 11.
- **Comments.** About 15 stale comments are fixed across the five
  `src/` modules: runtime ids, the PCF ctx size, `cyrius vet`, and why the
  slot counts are not expressions. `src/font_data.cyr` changed in comments
  only and builds on every consumer pin (6.6.3 – 6.6.20).
- **Gates.** The glyph sheet `5544d63f…` and the demo output are
  byte-identical. Fuzz is clean. `cyrius vet` reports "no dependencies" on
  all four freestanding files. Sizes are unchanged. The bench is flat
  (±2.6 %; the fixed function is not on a benched path).

**1.2.1** — Patch: a toolchain bump plus the full-tree doc sweep that 1.2.0
left outstanding. `src/`, `tests/` and `docs/examples/` are unchanged.
- **cyrius 6.7.9 → 6.7.18.** `lib/` was re-vendored from the 6.7.18
  snapshot and `cyrius.lock` re-locked (17 of 24 hashes moved).
  `cyrius deps --verify` reports 24 / 0 failed, and `--locked` is clean.
  kashi has no git deps.
- **Gates.** The glyph sheet is byte-identical (`5544d63f…`), and so is the
  demo's output. 454 + 145 assertions pass. Fuzz is clean. `cyrius vet`
  reports "no dependencies" on all four freestanding files. fmt and strict
  lint are clean. All four guard mutants are still caught.
- **Cost.** None. Library **199,968 B** and DCE'd demo **56,816 B**, the
  same as on 6.7.9. The bench is flat: every row is within ±0.9 %, over
  nine interleaved rounds.
- **Bundle.** Only the version stamp moved. A scratch consumer builds and
  runs the 1.2.1 bundle on 6.6.3, 6.6.6 and 6.7.18.
- **Docs.** The full-tree sweep owed at the 1.2 minor; the ledger
  (`docs/doc-health.md`) has the per-file grades.

**1.2.0** — Minor, cut to mark the move to cyrius's 6.7.x language line. It
is a toolchain-only bump: `src/`, `tests/` and `docs/examples/` are
unchanged, and no glyph, symbol, signature or accessor result changed.
- **cyrius 6.6.20 → 6.7.9.** The 6.7.0 – 6.7.9 consumer notes were checked
  against every source file, and none of their shapes is present. `lib/`
  was re-vendored from the 6.7.9 snapshot and `cyrius.lock` re-locked
  (14 of 24 hashes moved). `cyrius deps --verify` reports 24 / 0 failed,
  and `--locked` is clean.
- **Gates.** The glyph sheet is byte-identical (`5544d63f…`). 454 + 145
  assertions pass. Fuzz is clean. `cyrius vet` reports "no dependencies" on
  all four freestanding files. fmt and strict lint are clean on all 11
  files. The fset-guard mutation was re-run, and all four guards are still
  caught.
- **Cost.** Library 199,952 → **199,968 B**, DCE'd demo 56,800 →
  **56,816 B** (+16 B each). The bench is flat: every row is within ±1.6 %
  of 1.1.1 on 6.6.20, over nine interleaved rounds.
- **Bundle.** `dist/kashi.cyr` changed only in its version stamp, and
  `dist/kashi.deps` is unchanged.
- **The core stays on 6.6.x syntax.** Every consumer compiles
  `src/font_data.cyr` on its own pin: agnos 6.6.20, aethersafha 6.6.11,
  dhancha, crab and puka 6.6.6, and jalwa 6.6.3.

**1.1.1** — Patch. Toolchain bump + the one issue filed against 1.1.0. No
glyph, symbol, signature or accessor result changed.
- **Glyph storage right-sized**
  ([issue](issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md),
  filed by agnos). `kashi_font16` / `kashi_font8` / `kashi_font9_16` were
  declared `[3616]` / `[1808]` / `[7232]` — the BYTE counts — but a
  module-scope `var X[N]` is N u64 slots, so they reserved 101,248 B for
  12,656 B used, all of it zeros in the file image of every consumer. Now
  `[452]` / `[226]` / `[904]` slots (named in the internal
  `_KashiFontStore`), exactly the bytes used. Measured on 6.6.20: library
  288,528 → **199,952 B**, DCE'd demo 145,376 → **56,800 B**; a
  freestanding core-only program on agnos's 6.6.6 pin 127,312 → 38,736 B
  — **−88,576 B** each (agnos measured 88,592 B of kernel-image headroom).
- **Tests**: the "fset guard" check that read the first *padding* byte
  (there is none now) became a checksum of every byte the core owns, and
  the reverse index is declared before the glyph buffers so all four
  packer guards are observable (removing `kashi_fset16`'s lower guard went
  unnoticed before); new "storage" group (declared slots × 8 == bytes
  used; the core's variables pairwise disjoint). Unit 448 → **454**.
  12 mutants, all caught.
- **cyrius 6.6.6 → 6.6.20**. Glyph sheet byte-identical (`5544d63f…`) on
  both toolchains and after the fix; bench flat (±2 %). A bare `cyrius
  test` runs both suites (6.6.17) — CI's separate unit step dropped.
  `cyrius.lock` now tracked (6.6.9; the stdlib-only siblings track theirs; CI's
  `cyrius deps` refuses a vendored file whose hash differs). distlib
  (6.6.18): `dist/kashi.deps` 8 leaves → 4, and the bundle opens with a
  requires block; a scratch consumer builds against it on 6.6.20 and 6.6.6.
- **CI lint gate armed**: `cyrius lint` exits 0 on warnings, so the step
  never failed; now `--exit-with-count --strict-deferrals`, and `tests/`
  is linted too (all files were already clean).

**1.1.0** — Additive minor (ADR 0011). The ellipsis the compositor needs
for truncated titles, and Unicode text for every consumer:
- **`kashi_cp437_from_unicode(cp)`** in the freestanding core: Unicode
  codepoint → the cell that draws it, or 0. ASCII identity; the CP437
  high half per the standard mapping (Python's `cp437` codec = the
  Unicode Consortium's CP437.TXT; dhancha's table agrees entry for
  entry); U+2302 → `0x7F` while U+007F DELETE → 0; 226 codepoints map in
  all of Unicode. ASCII fast path + binary search over a 128-entry index
  that `kashi_font_init` fills from a source already sorted by codepoint
  (straight stores, no sort, no count reset); `load8`/`store8` only,
  `cyrius vet` still "no dependencies".
- **Extension block** `KASHI_EXT_FIRST..KASHI_EXT_LAST` (`0x100`
  U+2026 '…', `0x101` U+FFFD '�'), kashi-drawn in all three fonts,
  stored right after `0xFF`; reachable only by codepoint.
  `KASHI_GLYPH_*` / `kashi_font_count` unchanged.
- **Fixed**: the library face's `kashi_font_row` / `_ptr` / `_row_byte`
  read a built-in's codepoint as a CP437 cell (U+00E7 'ç' drew 0xE7 'τ');
  they map it now. Callers passing CP437 cells there switch to
  `kashi_glyph_*`.
- Byte-indexed consumers see nothing: every byte of `0x20..0xFF` in all
  three fonts is 1.0.10's (FNV-1a test + the new sheet's first 672 lines
  hash to the old pin `b68ca4d9…`). Glyph-sheet pin re-pinned
  deliberately: 678 lines, `5544d63f…`.
- Tests: unit 393 → **448**, integration 57 → **145** (incl. the index's
  order and its byte identity across a re-init); fuzz gains the Unicode
  contract (100k triples) and reaches the extension cells. CI now runs the
  unit suite as its own step (a bare `cyrius test` skips it on 6.6.6).
  Mutation-tested in two rounds: 26 against the first draft (all 24 that
  change behaviour caught, 2 equivalent) and 12 against the final index
  build (all 11 that change behaviour caught; the 12th, a count reset
  before the fill, is equivalent single-threaded).
- Cost: library 256,592 → **262,656 B**, DCE'd demo 137,936 → **144,072 B**
  (+6,064 B of that from the core: 1,928 B BSS + ~3 KB code);
  `kashi_font_init` **+1.2 µs (+6 %)**, 20.8 → 22.0 µs; `font_row_builtin`
  +3 ns; ~26 ns per high-half lookup; agnos image headroom −5.7 KB.

**1.0.10** — No API change. Three gaps closed and a docs sweep:
- **`dist/kashi.cyr` actually published.** The 1.0.6 bundle was
  gitignored and never attached to a release, so a `git` + `tag`
  consumer could not resolve it (checked: the cached 1.0.6–1.0.8
  clones have no `dist/`). Now tracked (+ `dist/kashi.deps`), attached
  to releases, and gated: CI regenerates and fails on drift or if the
  files are untracked; the release refuses a bundle whose version
  stamp is not the tag. Verified with a scratch consumer resolving
  `git` + `tag` + `modules = ["dist/kashi.cyr"]` against a tagged clone.
- **Fuzz harness fixed and armed.** `tests/kashi.fcyr`'s known-font
  list predated the 9×16 (0.5.1) — a latent false failure — and its
  core-accessor contract only ever saw two canned seeds. Now: id 2
  listed, contract stride-aware and split into distinct codes
  (1–8), plus a 200k-triple random driver over three input shapes
  (byte, full-width i64, near-edge). Mutation-tested: five injected
  defects (two harness, three library) each caught.
- **Glyph-sheet pin.** `docs/examples/glyph_sheet.cyr` (the freestanding
  render example; resolves the empty `docs/examples/`) dumps all 672
  built-in glyphs as hex; CI pins the output's sha256
  (`docs/examples/glyph_sheet.sha256`). This is the toolchain-bump
  recipe's byte-for-byte comparison, run on every push.
- `tests/kashi.tcyr` +8 assertions (9×16 structural group): **57**.
- Docs currency: `getting-started.md` rewritten for 1.0; `cyaudit vet`
  → `cyrius vet` across living docs; `architecture/001` sizes;
  benchmarks CSV + doc (1.0.9 / 1.0.10 rows); `cyrius.cyml` prose;
  function count 43 (was 45 / 44); ADR forward pointers; this file's
  interior.

**1.0.9** — Toolchain bump. Pins cyrius `6.6.6` (was `6.6.4`); no
source changes, public API still frozen. Re-verified on 6.6.6 per the
roadmap recipe: 393 unit + 49 integration assertions, 0 failed;
`cyrius vet src/font_data.cyr` → dependency-free; fuzz clean; bench
flat within noise. The full glyph-sheet render (3 fonts × 224 glyphs,
every row byte) is **byte-identical** between the 6.6.4 and 6.6.6
builds, and `dist/kashi.cyr` differs only in its version stamp.
Library 256,592 B; DCE'd demo 137,936 B. Ignored `lib/` re-vendored to
the 6.6.6 snapshot (6.6.5 moved the aarch64 syscall peer).

**1.0.8** — Toolchain bump. Pins cyrius `6.6.4` (was `6.6.2`); no
source changes, public API still frozen. Re-tested clean on 6.6.4
(49 integration + 1 unit, 0 failed; `cyrius vet src/font_data.cyr` →
dependency-free). Moves with agnos 1.57.4.

**1.0.4 – 1.0.7** — Toolchain bumps (6.4.62 → 6.5.27 → 6.6.2), plus
1.0.6's `[lib]` + `dist/kashi.cyr` bundle. Narrative in `CHANGELOG.md`.

**1.0.3** — Toolchain bump. Pins cyrius `6.4.62` (was `6.2.22`); no
source changes, public API still frozen. Rebuilt + re-tested clean
on 6.4.62 (393 unit + 49 integration, 0 failed; `cyrius vet` → "no
dependencies" for all four freestanding files).

**1.0.2** — Toolchain bump. Pins cyrius `6.2.22` (was `6.2.2`); no
source changes, public API still frozen. Rebuilt + re-tested clean
on 6.2.22 (393 unit + 49 integration, 0 failed; `cyrius vet` → "no
dependencies" for all four freestanding files).

**1.0.1** — Toolchain bump. Pins cyrius `6.2.2` (was `6.0.3`); no
source changes, public API still frozen. Rebuilt + re-tested clean
on 6.2.2 (393 unit + 49 integration, 0 failed; `cyrius vet` → "no
dependencies" for all four freestanding files). 6.2.2 fixes the
6.0.3 `cyrius vet` packaging bug, so the `cyaudit vet` workaround is
retired across `README.md`, `CONTRIBUTING.md`, and CI.

**1.0.0** — Stable release. The full surface (freestanding core,
runtime loading for PSF / BDF / PCF, sidecar Unicode-table attach,
codepoint-addressed accessors) is frozen — see
[`docs/api/`](../api/) for the reference and stability promise.
No new features in this cut; the work was a clean review of the
0.9.0-frozen surface:
- `README.md` rewritten for the shipped state (was stuck at 0.1.0).
- `SECURITY.md` refreshed: current buffer sizes, current parser
  list, current audit trail, supported-versions = 1.x.
- `CONTRIBUTING.md` polished: `cyaudit vet` workaround, current
  return-code convention.
- Stale comments in `src/font_data.cyr` and `src/font_psf.cyr`
  fixed: agnos-integration language, CGA-blank-high-half
  (was filled in 0.5.2), scope-as-of-0.6.0 bumped to 1.0.0.

The roadmap closes here. Post-1.0 ideas are noted as
out-of-scope; none are booked.

## Toolchain

- **Cyrius pin**: `6.7.18` (in `cyrius.cyml [package].cyrius`; 6.7.9 at
  1.2.0, 6.6.20 at 1.1.1, 6.6.6 through 1.1.0).
- **`cyrius.lock`**: tracked since 1.1.1 — the sha256 of the 24 stdlib
  files `cyrius deps` vendors from the pinned snapshot; moves only with
  the pin (roadmap procedure, step 2).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr`. NO stdlib
  (`cyrius vet` → "no dependencies"). Cells `0x20..0xFF` (full CP437,
  224 slots) + the extension block `0x100..0x101` ('…', '�'; 1.1.0).
  Three built-in fonts:
  - `KASHI_FONT_VGA_8X16` (id 0) — IBM VGA BIOS 8×16, 224 glyphs (full
    CP437; PD source = Linux's `font_8x16.c`).
  - `KASHI_FONT_CGA_8X8` (id 1) — dual-sourced 8×8 (hand-drawn AGNOS
    original for `0x20..0x7F`; Linux PD `font_8x8.c` for `0x80..0xFF`).
  - `KASHI_FONT_VGA_9X16` (id 2) — VGA 9×16 derived at init from
    `KASHI_FONT_VGA_8X16` + the VGA col-9 replication rule.
  - Accessors: `kashi_font_init`, `kashi_font_is_ready`,
    `kashi_glyph_row` (stride-aware), `kashi_glyph_row_byte`,
    `kashi_glyph_ptr`, `kashi_font_{width,height,first,count,stride}`,
    `kashi_glyph_encoded`. All bounds-safe for the full `i64` input
    domain.
  - **Unicode → cell** (1.1.0, ADR 0011): `kashi_cp437_from_unicode` —
    226 mapped codepoints, everything else 0; `KASHI_EXT_FIRST` /
    `_LAST` / `_COUNT`.
- **PSF parser** — `src/font_psf.cyr` (heapless). `kashi_psf_parse`
  validates PSF1/PSF2 headers; `kashi_psf_uni_token` decodes the
  Unicode table.
- **BDF parser** — `src/font_bdf.cyr` (heapless; 0.7.0, ADR 0008).
  `kashi_bdf_parse_header` + `kashi_bdf_next_glyph` cursor pair;
  strict-BBX, `ENCODING -1` skipped.
- **PCF parser** — `src/font_pcf.cyr` (heapless; 0.7.1, ADR 0009).
  `kashi_pcf_parse_header` walks the TOC + validates the three
  required tables; `kashi_pcf_decode_glyph` canonicalizes one glyph
  to MSB-bit / 1-byte-stride form (scan-unit byte swap + bit-reverse
  as needed); `kashi_pcf_cp_to_idx` resolves codepoints via the
  BDF_ENCODINGS table. Strict-uniform-metrics, all four
  byte-order × bit-order combos, both compressed and uncompressed
  metric layouts. `cyrius vet` → "no dependencies".
- **Library face** — `src/lib.cyr`. Runtime font registry + the
  PSF / BDF / PCF import paths + sidecar table attachment:
  - **PSF**: `kashi_load_psf` / `kashi_load_psf_file` /
    `kashi_register_font` → `font_id ≥ 3` or `0 - <KashiResult>`.
  - **BDF** (0.7.0): `kashi_load_bdf` / `kashi_load_bdf_file`. 4 MiB
    file cap.
  - **PCF** (0.7.1): `kashi_load_pcf` / `kashi_load_pcf_file`. 4 MiB
    file cap. Walks the encoded codepoint range, collects (cp,
    glyph_idx) pairs, insertion-sorts. Multiple codepoints may map
    to the same glyph index.
  - **Sidecar attach** (0.7.2): `kashi_attach_unicode_table` /
    `kashi_attach_unicode_table_file` for binary PSF1/PSF2 tables;
    `kashi_attach_unicode_text` / `kashi_attach_unicode_text_file`
    for the `psfgettable` text format. Replace semantics; runtime
    font ids only (built-ins → `KASHI_EINVAL`). 256 KiB file cap
    (`KASHI_TAB_FILE_CAP`).
  - Unified **codepoint-addressed** `kashi_font_row` / `kashi_font_ptr`
    (built-in 0,1,2 vs runtime ≥ 3). Built-ins map the Unicode codepoint
    through `kashi_cp437_from_unicode` since 1.1.0 (before, they read it
    as a CP437 cell).
  - **Multi-byte row access**: `kashi_font_stride` /
    `kashi_glyph_row_byte` in the core; `kashi_rt_font_stride` /
    `kashi_rt_glyph_row_byte` / `kashi_font_row_byte` in the library.
  - **Ligature lookup**: `kashi_font_seq_glyph` (PSF-only; BDF/PCF
    have no ligature concept).
  - Raw glyph-index `kashi_rt_glyph_row` / `kashi_rt_glyph_ptr`;
    `kashi_rt_font_{width,height,count,stride}`; `kashi_font_total`;
    `kashi_set_active_font` / `kashi_active_font`.
  - Scope: width 1–32 (multi-byte rows up to 4 bytes/row).
- **Published bundle** — `dist/kashi.cyr` + `dist/kashi.deps`
  (`cyrius distlib` over the `[lib]` modules; 4,001 lines as distlib
  reports them, 4,026 by `wc -l`). Tracked and
  released since 1.0.10; what a consumer declares as
  `modules = ["dist/kashi.cyr"]`. CI fails on drift. Since 1.1.1 (cyrius
  6.6.18's distlib) it opens with a requires block — `include
  "lib/{vec,alloc,string,io}.cyr"` — and the sidecar names those four
  leaves (was eight).
- **Demo** — `src/main.cyr`, renders 'A' in both 8-wide built-in fonts,
  then '…' and '�' through `kashi_cp437_from_unicode`.
- **Example / gate** — `docs/examples/glyph_sheet.cyr`, every built-in
  glyph as hex (672 CP437 lines, then the 6 extension lines); sha256
  pinned in `docs/examples/glyph_sheet.sha256` (`5544d63f…` since 1.1.0;
  the first 672 lines still hash to the 1.0.x pin `b68ca4d9…`) and
  checked in CI.

## What's booked (not built — future)

[`roadmap.md`](roadmap.md) holds the milestones:
- **1.3.x — Hardening, next and owed now:**
  - the P(-1) security audit (none since 0.8.0);
  - the worst-case cost of the insertion-sorted codepoint maps;
  - fuzz contracts with a round-trip oracle;
  - load-path benchmarks.
- **1.4.x — Unicode coverage:** more extension glyphs and aliases, when a
  consumer names a glyph.
- **1.5.x — Loader breadth:** BDF lenient-BBX, PCF per-glyph metrics and
  streaming loads, when a consumer's font won't load.
- **1.x.x:** a fourth built-in font (agnos) and 6.7 syntax in the core
  (once every consumer pins ≥ 6.7.2).

## Build & size

- Library build (`cyrius build src/lib.cyr build/kashi`): clean, **199,968 B**
  (sha256 `55b2766a…`; the same size since 1.2.0). 1.1.1: 199,952 B on 6.6.20. 1.1.0: 262,656 B on
  6.6.6, 288,528 B on 6.6.20.
- Demo (`CYRIUS_DCE=1 cyrius build src/main.cyr`): **56,816 B**
  (sha256 `b7ab70cb…`, unchanged by 1.2.2; the same size since 1.2.0). 1.1.1: 56,800 B on 6.6.20. 1.1.0: 144,072 B on
  6.6.6, 145,376 B on 6.6.20.
- 1.2.0's toolchain move (6.6.20 → 6.7.9, no source change) added 16 B to
  each; 1.2.1's (6.7.9 → 6.7.18) added nothing.
- 1.1.1's two steps, separately: the toolchain (6.6.6 → 6.6.20, no source
  change) grew the library +25,872 B and the demo +1,304 B — stdlib
  growth; the storage fix then took **−88,576 B** off each. The core's
  module-scope storage is now 13,696 B: the three glyph buffers'
  12,656 B, exactly the bytes used (were 101,248 B), the 1 KiB reverse
  index, two 8-byte scalars, 8 B of alignment. cycc writes it into the file
  image as zeros, so every binary that links `src/font_data.cyr` carries
  it — architecture note 001.
- Both on cyrius 6.7.18.

## Tests

- `src/test.cyr` — 464 assertions (incl. the 13 audit regression
  assertions added in 0.8.0 for findings F1, F2, F3, F5, F6, F7, F8,
  F9; F4 was cosmetic; 1.1.0 added the extension-glyph fidelity bytes,
  a Unicode spot group, and moved the out-of-range probes from `0x100`
  to `KASHI_EXT_LAST + 1`; 1.1.1 replaced the fset guard's two padding
  probes with one whole-core checksum and added the 7-assertion
  "storage" group; 1.2.2 added the 10-assertion "pcf cp_to_idx" group).
  **0 failed.** A bare `cyrius test` runs it first
  (6.6.17+); on the 6.6.6 pin, through 1.1.0, it needed
  `cyrius test src/test.cyr`, and CI ran it as its own step from 1.1.0
  (never before).
- `tests/kashi.tcyr` — 145 assertions (structural invariants for all
  three built-ins incl. the 9×16 stride-2 group added in 1.0.10; 1.1.0's
  Unicode groups: the full CP437 round trip against the suite's own
  codec copy, the reverse index's order (codepoints strictly ascending)
  and its byte identity across a re-init, a sweep of all of
  U+0000..U+10FFFF (exactly 226 map),
  the 0x7F rule, the extension glyphs' rows in all three fonts, extension
  bounds, the library face on built-ins across 224 cells × 3 fonts, and
  FNV-1a proof that `0x20..0xFF` is byte-for-byte 1.0.10; PSF + BDF + PCF
  file round-trips, PSF + sidecar tab file round-trip, missing-file
  negative cases).
- **609 assertions total, 0 failed** (two files, one bare `cyrius test`).
- `tests/kashi.fcyr` — the core-accessor bounds contract (three canned
  seeds + 200,000 random `(font, ch, row)` triples across small (9-bit
  cells, so the extension is reached), full-width i64 and near-edge
  shapes; eight distinct failure codes), the Unicode → cell contract
  (1.1.0: 100,000 `(cp, font, row)` triples, codes 9–11 — only 0 or an
  encoded cell, ASCII identity, library face == core of the mapped cell),
  the PSF parser (4000 rounds), the BDF parser (2000 rounds), the PCF
  parser (1500 rounds; since 1.2.2 a cp above U+FFFF must miss, code 11),
  and the text-tab parser (1000 rounds across
  random / template / mutated / truncated picks). No crashes; all
  accept-paths bounds-safe. Mutation-tested in 1.0.10.
- `tests/kashi.bcyr` — see `docs/benchmarks.md`; the core hot path is
  unchanged source-side since 0.4.0, and ~12 % faster toolchain-side
  since 1.0.0 (cyrius 6.0.3 → 6.6.6). 1.1.0 adds three Unicode benches
  and `font_init`, and costs `font_row_builtin` +3 ns (the mapping) and
  `kashi_font_init` +1.2 µs (the extension glyphs + the index fill).
  1.1.1: flat — every row within ±2 % across 1.1.0 on 6.6.6, 1.1.0 on
  6.6.20 and 1.1.1 on 6.6.20 (nine interleaved runs each). 1.2.0: flat,
  every row within ±1.6 % of 1.1.1 on 6.6.20 (nine interleaved rounds).
  1.2.1: flat, every row within ±0.9 % of 1.2.0 on 6.7.9. 1.2.2: flat,
  every row within ±2.6 % of 1.2.1.
- Glyph-sheet pin — `docs/examples/glyph_sheet.sha256`, checked in CI.

## Cleanliness (P(-1) gates)

- `cyrius build` — clean.
- `cyrius fmt <file> --check` — clean on all src, test and example files.
- `cyrius lint --exit-with-count --strict-deferrals` — 0 warnings, 0
  untracked deferrals on all src, test and example files. (CI enforces it
  since 1.1.1; a bare `cyrius lint` exits 0 on warnings, so before that the
  step only printed them.)
- `cyrius deps --verify` — 24 verified, 0 failed against the tracked
  `cyrius.lock`; `cyrius deps --locked` clean (6.7.18).
- `cyrius distlib` — tracked `dist/` in sync (CI gate).
- `cyrius vet src/font_data.cyr` — "no dependencies".
- `cyrius vet src/font_psf.cyr` — "no dependencies" (incl. the 0.7.2
  overlong-UTF-8 tightening — pure arithmetic, no new deps).
- `cyrius vet src/font_bdf.cyr` — "no dependencies".
- `cyrius vet src/font_pcf.cyr` — "no dependencies".
- `cyrius vet src/lib.cyr` — 4 deps (the four files it includes: core,
  psf, bdf, pcf), 0 untrusted, 0 missing. The 0.7.2 attach APIs and the text-tab parser are
  inline in lib.cyr (no new dependency module).
- **Security audit** — `docs/audit/2026-05-28-audit-0.8.0.md` (0.8.0
  P(-1) hardening). CVE-research-driven 17-point checklist walk
  against the full post-0.7.2 surface; 9 findings landed (F1..F9),
  all fixed. None exploitable; the worst (F9) was a stale-state-
  after-parse-failure pattern in the attach API matching
  CVE-2015-1803. Earlier audits: `2026-05-27-audit.md` (0.2.0 P(-1)
  on the freestanding core), `2026-05-28-audit.md` (0.6.0 P(-1)).

## Glyph fidelity

Built-in tables diffed byte-for-byte against agnos's source — **0
mismatches**. The two extension glyphs (1.1.0) are kashi-drawn, not from
any ROM; their rows are pinned in `src/test.cyr` / `tests/kashi.tcyr` and
drawn in ADR 0011. Runtime-loaded fonts (PSF / BDF / PCF) carry their own
glyphs from the source; kashi exercises round-trips on small
synthetic buffers in `src/test.cyr` and `tests/kashi.tcyr`.

## Dependencies

Direct (declared in `cyrius.cyml [deps].stdlib`): `string`, `fmt`,
`io`, `vec`, `alloc`, `syscalls`, `assert`, `bench`. The library face
actively uses `alloc`/`vec` (runtime registry), `string` (`memcpy`),
and `io` (`file_read_all` for the three `kashi_load_*_file` and the two
`kashi_attach_*_file` entry points); `fmt` / `syscalls` / `assert` /
`bench` serve the demo and the harnesses. The list is unchanged since
0.7.2. No git-deps. `cyrius.lock` is **tracked since 1.1.1** (it was
omitted by policy; cyrius 6.6.9+ writes one for a stdlib-only project and
asks that it be committed — dhancha, crab, puka, jalwa, aethersafha and
rekha all track theirs). `dist/kashi.deps` names
the four leaves the bundle needs — `vec`, `alloc`, `string`, `io` (eight,
the whole list, through 1.1.0; 6.6.18's distlib derives it from a
compile-verify).

## Consumers

All six take the **freestanding core** (`modules = ["src/font_data.cyr"]`,
vendored as `lib/kashi_font_data.cyr`); none calls a runtime loader —
surveyed 2026-09-21 across `~/Repos`. Pins re-read from each `cyrius.cyml` 2026-10-29:

| Consumer | kashi | their cyrius | notes |
|---|---|---|---|
| **agnos** (kernel) | `path` only, no tag | 6.6.20 | Integrated at 1.38.0 (M3, 0.7.0); folds `src/font_data.cyr` into the kernel since 1.57.4. `fb_console.cyr` renders via `kashi_glyph_ptr` + `load8`; `gpu.cyr` via `kashi_glyph_row`. |
| dhancha | 1.0.10 | 6.6.6 | Default face for `dh_draw_text`. |
| crab | 1.0.10 | 6.6.6 | |
| jalwa | 1.0.7 | 6.6.3 | |
| aethersafha | 1.1.0 | 6.6.11 | |
| puka | 1.0.10 | 6.6.6 | Five programs include the core. ⚠ `fb__glyph_draw` passes Unicode **scalars** to `kashi_glyph_row` — not a byte-indexed consumer (below). |

The desktop-stack four carry a manifest note to switch to
`modules = ["dist/kashi.cyr"]` the day they need runtime loading — which
resolves only from 1.0.10 on. 1.0.9 and 1.0.10 are data-identical to
1.0.6–1.0.8, so no consumer needs to move for correctness.

**1.1.0 for consumers** — nothing changes for a byte-indexed consumer
(every `0x20..0xFF` byte is 1.0.10's). ⚠ puka is not one: its framebuffer
renderer passes scalars, so on its bump U+0100 / U+0101 go from blank to
'…' / '�' — map through `kashi_cp437_from_unicode` first, which also fixes
its long-wrong U+0080..U+00FF. What it offers: a consumer drawing
UTF-8 decodes, then `kashi_cp437_from_unicode` per scalar (dhancha's
`dh_text_cp437` and thoth's `graster.cyr` fold are private copies of that
job); the compositor's truncated titles get a real '…' via
`kashi_cp437_from_unicode(0x2026)`. What every core consumer pays: ~6 KB
of binary and +1.2 µs per `kashi_font_init()` — per draw for dhancha
(and crab / puka through their vendored copy), whose `dh_draw_text_ink`
inits unconditionally; a `kashi_font_is_ready()` guard there would make
it once. ⚠ agnos takes kashi by `path`, so a sibling checkout at 1.1.0 is
what its next local build compiles, no glyph change: measured with its
`scripts/build.sh` on 6.6.6, image headroom under the 0x390000 ceiling
99,336 → 93,592 B plain, 45,600 → 39,840 B with `TEST=1 KTEST=1` (both
pass its image-layout gate). Its `KASHI_REF` clone fallback still says
1.0.10 — bump it (build.sh / test.sh / bench.sh) when 1.1.0 is tagged.
(Re-read 2026-10-13: all three now say 1.1.0.)

**1.1.1 for consumers** — nothing to change, nothing to re-render: the
same symbols, the same glyph offsets, every accessor answers the same,
and the 678-line glyph sheet hashes to the 1.1.0 pin. What moves is size:
**−88,576 B** in every binary that links the core (12,656 B of glyph
storage, was 101,248 B). The core still compiles freestanding on 6.6.6 —
agnos's pin and that of most of the stack — and passes both suites there.
⚠ The glyph buffers now have no slack: a consumer that read past the last
cell's rows through a raw `kashi_glyph_ptr` (none in the stack does — they
read `height × stride` bytes) would now read the next variable instead of
zeros.
agnos measured its kernel image against a scratch copy with the same three
sizes: headroom under its 0x400000 ceiling 325,952 → **414,544 B**
(plain build); it takes 1.1.1 by `path` on its next local build, and
through `KASHI_REF` (still 1.1.0 in build.sh / test.sh / bench.sh) once
the tag exists. (Re-read 2026-10-29: `KASHI_REF` now says 1.1.1.)

**1.2.0 for consumers** — nothing to change. `src/font_data.cyr` is
byte-identical to 1.1.1's, and `dist/kashi.cyr` differs only in its version
stamp. A consumer on any 6.6.x pin compiles the same core as before. The
6.7.9 pin is kashi's own build. ⚠ When a consumer moves its *own* pin to
6.7.6 or later, `[deps.kashi]` changes meaning. A `path` beside `git` /
`tag` (crab) is used only under `--local`; a default build builds the tag.
A `path`-only entry (agnos) is refused by name.
