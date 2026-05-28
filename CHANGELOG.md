# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [SemVer](https://semver.org/) (pre-1.0: surface still moving).

## [Unreleased]

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

[Unreleased]: https://github.com/MacCracken/kashi/compare/0.5.1...HEAD
[0.5.1]: https://github.com/MacCracken/kashi/compare/0.5.0...0.5.1
[0.5.0]: https://github.com/MacCracken/kashi/compare/0.4.0...0.5.0
[0.4.0]: https://github.com/MacCracken/kashi/compare/0.3.0...0.4.0
[0.3.0]: https://github.com/MacCracken/kashi/compare/0.2.0...0.3.0
[0.2.0]: https://github.com/MacCracken/kashi/compare/0.1.0...0.2.0
[0.1.0]: https://github.com/MacCracken/kashi/releases/tag/0.1.0
