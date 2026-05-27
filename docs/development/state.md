# kashi — Current State

> **Last refresh**: 2026-05-27 (0.1.0 baseline) | **Refresh cadence**:
> bumped every release (ideally by the release post-hook).
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**0.1.0** — initial baseline. Split out of the agnos kernel's framebuffer
console. No git tag yet (user handles all git operations).

## Toolchain

- **Cyrius pin**: `6.0.3` (in `cyrius.cyml [package].cyrius`).

## What's implemented (0.1.0)

- **Freestanding font-data core** — `src/font_data.cyr` (~392 lines). NO
  stdlib (`cyrius vet` → "no dependencies"). Two built-in fonts:
  - `KASHI_FONT_VGA_8X16` (id 0) — IBM VGA BIOS 8×16, 96 glyphs.
  - `KASHI_FONT_CGA_8X8` (id 1) — hand-drawn CGA 8×8, 96 glyphs.
  - Accessors: `kashi_font_init`, `kashi_glyph_row`, `kashi_glyph_ptr`,
    `kashi_font_{width,height,first,count}`, `kashi_glyph_encoded`.
- **Library face** — `src/lib.cyr` (~83 lines). Re-exports the core; books
  `kashi_load_psf`, `kashi_register_font`, `kashi_font_total` + result codes
  (return `KASHI_ENOSYS` until implemented).
- **Demo** — `src/main.cyr` (~54 lines), renders 'A' in both fonts.

## What's booked (not built — parallel agent / future)

- PSF1/PSF2 import (M1, 0.2.0) — needs stdlib `io`/`fs` + heap.
- Runtime font registry + additional fonts (M2, 0.3.0).
- agnos consumption contract hardening + agnos-side integration (M3 / agnos
  **1.38.0**).

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`CYRIUS_DCE=1 cyrius build src/lib.cyr build/kashi`): clean.
- Binary size: ~74 KB DCE (demo + library; the freestanding core itself adds
  ~18 KB of font tables + accessors when included by a consumer).

## Tests

- `src/test.cyr` — 71 assertions (metadata, encoded gate, **exact-byte
  fidelity vs agnos source**, accessor bounds, full 96-glyph coverage,
  library skeleton). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 10 assertions (byte-bounded rows, space-vs-visible
  ink, pointer monotonicity, reinit idempotency). **0 failed**.
- **81 assertions total, 0 failed.**
- `tests/kashi.fcyr` — fuzz harness over the accessor bounds contract.
- `tests/kashi.bcyr` — `glyph_row` ~18 ns, `glyph_ptr` ~7 ns,
  `scan_vga_8x16` ~27 µs (x86_64 Linux, indicative — no CSV trail yet).

## Cleanliness (P(-1) gates)

- `cyrius build` — clean (no warnings).
- `cyrius fmt <file> --check` — clean on all src + test files.
- `cyrius lint` — 0 warnings on all src files.
- `cyrius vet src/font_data.cyr` — "no dependencies" (freestanding boundary
  proven); `cyrius vet src/lib.cyr` — 2 deps (core + self), 0 untrusted.
- `cyrius audit` — not run (the `check.sh` helper isn't present in this
  toolchain install); individual gates above cover P(-1).

## Glyph fidelity

Both tables diffed byte-for-byte against agnos's source — **0 mismatches**
across all 96 glyphs in each font. 8×16 from `agnos
kernel/arch/x86_64/fb_console.cyr` (HEAD); 8×8 from the pre-8×16-replacement
commit (`75914e9^`, before "self rolled glyph to font").

## Dependencies

Direct (declared in `cyrius.cyml [deps].stdlib`):
`string`, `fmt`, `io`, `vec`, `alloc`, `syscalls`, `assert`, `bench` — for
the **library face + test/bench harnesses only**. The freestanding core
needs none.

## Consumers

- **agnos** (kernel) — booked at **1.38.0**, will consume `src/font_data.cyr`
  via `[deps.kashi] path="../kashi", modules=["src/font_data.cyr"]`. Not yet
  wired (agnos-side work, out of kashi's scope).
