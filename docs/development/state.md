# kashi — Current State

> **Last refresh**: 2026-05-27 (**0.2.0** cut — M1 PSF import + P(-1)
> hardening; see `docs/adr/0002` and `docs/audit/2026-05-27-audit.md`) |
> **Refresh cadence**: bumped every release (ideally by the release
> post-hook).
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**0.2.0** — M1 PSF font-import path (PSF1/PSF2 load, runtime registry,
unified dispatch) + the post-0.1.0 P(-1) hardening pass. Tagged by the user
(who handles all git operations); the tag push drives the release workflow.

## Toolchain

- **Cyrius pin**: `6.0.3` (in `cyrius.cyml [package].cyrius`).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr` (~410 lines). NO
  stdlib (`cyrius vet` → "no dependencies"). Two built-in fonts:
  - `KASHI_FONT_VGA_8X16` (id 0) — IBM VGA BIOS 8×16, 96 glyphs.
  - `KASHI_FONT_CGA_8X8` (id 1) — hand-drawn CGA 8×8, 96 glyphs.
  - Accessors: `kashi_font_init`, `kashi_font_is_ready`, `kashi_glyph_row`,
    `kashi_glyph_ptr`, `kashi_font_{width,height,first,count}`,
    `kashi_glyph_encoded`. All bounds-safe for the full `i64` input domain
    (audited 2026-05-27).
- **PSF parser** — `src/font_psf.cyr` (self-contained, heapless).
  `kashi_psf_parse` validates PSF1/PSF2 headers (validation-first, fuzzed).
- **Library face** — `src/lib.cyr`. Runtime font registry + the M1 import
  path (see `docs/adr/0002`):
  - `kashi_load_psf` / `kashi_load_psf_file` / `kashi_register_font` →
    `font_id ≥ 2` or negative `0 - <KashiResult>`.
  - Unified `kashi_font_row` / `kashi_font_ptr` (dispatch built-in vs
    runtime), `kashi_rt_font_{width,height,count}`, `kashi_font_total`.
  - Scope: width ≤ 8; glyphs addressed by index; Unicode table validated,
    not yet mapped.
- **Demo** — `src/main.cyr`, renders 'A' in both built-in fonts.

## What's booked (not built — future)

- Wide glyphs (> 8 px / multi-byte rows) + Unicode→glyph mapping for runtime
  fonts (deferred from M1 — see `docs/adr/0002`).
- Runtime font registry niceties + additional built-in fonts (M2, 0.3.0).
- agnos consumption contract hardening + agnos-side integration (M3 / agnos
  **1.38.0**).

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`CYRIUS_DCE=1 cyrius build src/lib.cyr build/kashi`): clean.
- Binary size: ~74 KB DCE (demo + library; the freestanding core itself adds
  ~18 KB of font tables + accessors when included by a consumer).

## Tests

- `src/test.cyr` — 128 assertions (metadata, encoded gate, **exact-byte
  fidelity vs agnos source**, accessor bounds, full 96-glyph coverage,
  library surface, audit: ready flag / `fset` guard / full-`i64`-range
  safety, **PSF1+PSF2 parse (valid + malformed), runtime register/load,
  unified dispatch**). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 18 assertions (byte-bounded rows, space-vs-visible
  ink, pointer monotonicity, reinit idempotency, **PSF file round-trip**).
- **146 assertions total, 0 failed.**
- `tests/kashi.fcyr` — fuzz over the accessor bounds contract **and the PSF
  parser** (4000 random/truncated/mutated buffers; no crash, bounds-safe).
- `tests/kashi.bcyr` — `glyph_row` ~18 ns, `glyph_ptr` ~7 ns,
  `scan_vga_8x16` ~27 µs; unified dispatch `font_row_builtin` ~20 ns,
  `font_row_runtime` ~28 ns (x86_64). CSV trail in
  [`benchmarks.md`](../benchmarks.md) / `docs/benchmarks/history.csv`.

## Cleanliness (P(-1) gates)

- `cyrius build` — clean (no warnings).
- `cyrius fmt <file> --check` — clean on all src + test files.
- `cyrius lint` — 0 warnings on all src files.
- `cyaudit vet src/font_data.cyr` — "no dependencies" (freestanding boundary
  proven); `cyaudit vet src/font_psf.cyr` — "no dependencies" (pure parser);
  `cyaudit vet src/lib.cyr` — 3 deps (lib + core + psf), 0 untrusted.
  **NB**: invoke `cyaudit` directly — the released 6.0.3 `cyrius vet`
  dispatches to a `cybs` that emits an ELF instead of auditing (a cyrius
  packaging bug; CI works around it the same way).
- `cyrius audit` — not run (the `check.sh` helper isn't present in this
  toolchain install); individual gates above cover P(-1).
- **Security audit** — `docs/audit/2026-05-27-audit.md`: accessor surface
  proven memory-safe for the full `i64` domain; 3 findings fixed (fset
  guard, ready-flag reader, `0x7F` comment), 2 noted.

## Glyph fidelity

Both tables diffed byte-for-byte against agnos's source — **0 mismatches**
across all 96 glyphs in each font. 8×16 from `agnos
kernel/arch/x86_64/fb_console.cyr` (HEAD); 8×8 from the pre-8×16-replacement
commit (`75914e9^`, before "self rolled glyph to font").

## Dependencies

Direct (declared in `cyrius.cyml [deps].stdlib`):
`string`, `fmt`, `io`, `vec`, `alloc`, `syscalls`, `assert`, `bench`. The
**library face** now actively uses `alloc`/`vec` (runtime registry +
glyph-store copies), `string` (`memcpy`), and `io` (`file_read_all` for
`kashi_load_psf_file`) — no manifest change was needed for M1. The
freestanding core (`src/font_data.cyr`) and the PSF parser
(`src/font_psf.cyr`) need none.

## Consumers

- **agnos** (kernel) — booked at **1.38.0**, will consume `src/font_data.cyr`
  via `[deps.kashi] path="../kashi", modules=["src/font_data.cyr"]`. Not yet
  wired (agnos-side work, out of kashi's scope).
