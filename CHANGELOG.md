# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [SemVer](https://semver.org/) (pre-1.0: surface still moving).

## [Unreleased]

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

[Unreleased]: https://github.com/MacCracken/kashi/compare/0.2.0...HEAD
[0.2.0]: https://github.com/MacCracken/kashi/compare/0.1.0...0.2.0
[0.1.0]: https://github.com/MacCracken/kashi/releases/tag/0.1.0
