# kashi — Current State

> **Last refresh**: 2026-05-27 (**0.5.1** cut — VGA 9×16 derived built-in;
> see `docs/adr/0006`) | **Refresh cadence**: bumped every release
> (ideally by the release post-hook).
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**0.5.1** — VGA 9×16 derived built-in (`KASHI_FONT_VGA_9X16 = 2`). Bytes
computed at `kashi_font_init` time from the existing VGA 8×16 + the VGA
col-9 replication rule (CP437 box-drawing range `0xC0..0xDF` repeats col 7
into col 8; else col 8 is 0). `KASHI_RT_FONT_BASE` bumped `2 → 3`
(pre-1.0). `kashi_glyph_row` is now stride-aware (backward-compatible for
the 8-wide built-ins). Tagged by the user (who handles all git operations);
the tag push drives the release workflow.

## Toolchain

- **Cyrius pin**: `6.0.3` (in `cyrius.cyml [package].cyrius`).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr`. NO stdlib
  (`cyaudit vet` → "no dependencies"). Range `0x20..0xFF` (full CP437, 224
  slots; widened in 0.4.0 per `docs/adr/0004`). Three built-in fonts:
  - `KASHI_FONT_VGA_8X16` (id 0) — IBM VGA BIOS 8×16, 224 glyphs (full
    CP437; PD source = Linux's `font_8x16.c`).
  - `KASHI_FONT_CGA_8X8` (id 1) — hand-drawn CGA 8×8, 96 ASCII glyphs
    (`0x20..0x7F`); high-half slots exist but render blank.
  - `KASHI_FONT_VGA_9X16` (id 2) — VGA 9×16 **derived at init** from
    `KASHI_FONT_VGA_8X16` + the VGA col-9 replication rule (`0xC0..0xDF`
    box-drawing range); ADR 0006.
  - Accessors: `kashi_font_init`, `kashi_font_is_ready`, `kashi_glyph_row`
    (stride-aware), `kashi_glyph_row_byte`, `kashi_glyph_ptr`,
    `kashi_font_{width,height,first,count,stride}`, `kashi_glyph_encoded`.
    All bounds-safe for the full `i64` input domain (audited 2026-05-27).
- **PSF parser** — `src/font_psf.cyr` (self-contained, heapless).
  `kashi_psf_parse` validates PSF1/PSF2 headers + reports the Unicode-table
  offset; `kashi_psf_uni_token` decodes the table (PSF1 LE-u16 / PSF2 UTF-8).
  Validation-first, fuzzed.
- **Library face** — `src/lib.cyr`. Runtime font registry + the PSF import
  path (see `docs/adr/0002`, `docs/adr/0003`, `docs/adr/0005`):
  - `kashi_load_psf` / `kashi_load_psf_file` / `kashi_register_font` →
    `font_id ≥ 2` or negative `0 - <KashiResult>`. A loaded font's Unicode
    table is built into a sorted codepoint→glyph map **and** (0.5.0) a
    flat list of multi-codepoint ligature records.
  - Unified **codepoint-addressed** `kashi_font_row` / `kashi_font_ptr`
    (built-in 0,1 vs runtime ≥ 2; runtime resolves codepoint via the map,
    or identity index if the font carried no table).
  - **Multi-byte row access** (0.5.0): `kashi_font_stride` /
    `kashi_glyph_row_byte` in the core; `kashi_rt_font_stride` /
    `kashi_rt_glyph_row_byte` / codepoint `kashi_font_row_byte` in the
    library. Single-byte accessors return the leading byte for wide fonts.
  - **Ligature lookup** (0.5.0): `kashi_font_seq_glyph(id, cps_ptr, len)`
    → glyph index or `0 - 1`. Linear scan; sequences are typically few.
  - Raw glyph-index `kashi_rt_glyph_row` / `kashi_rt_glyph_ptr`;
    `kashi_rt_font_{width,height,count,stride}`; `kashi_font_total`;
    `kashi_set_active_font` / `kashi_active_font` (default 0 = VGA).
  - Scope: width 1–32 (multi-byte rows up to 4 bytes/row).
- **Demo** — `src/main.cyr`, renders 'A' in both built-in fonts.

## What's booked (not built — future)

- 9×16 box-drawing built-in (wide-glyph infrastructure now exists; adding
  the font data is a future cut).
- CGA 8×8 high half — currently blank; would need 128 hand-rolled glyphs.
- agnos consumption contract hardening + agnos-side integration (M3 / agnos
  **1.38.0**).
- Text shaping / BiDi — out of scope (kashi exposes data only).

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`CYRIUS_DCE=1 cyrius build src/lib.cyr build/kashi`): clean.
- Binary size: ~74 KB DCE (demo + library; the freestanding core itself adds
  ~18 KB of font tables + accessors when included by a consumer).

## Tests

- `src/test.cyr` — 243 assertions (metadata + stride, encoded gate,
  exact-byte fidelity (ASCII + CP437 high-half), accessor bounds, full
  224-glyph coverage, library surface, audit, PSF parse (8-wide + width
  12/32), runtime register/load (incl. 9-wide), Unicode token decode +
  codepoint mapping, raw-index + multi-byte row access, active-font,
  **ligature lookup**). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 24 assertions (structural invariants over the full
  0x20..0xFF range, PSF file round-trip + Unicode-table file by codepoint).
- **267 assertions total, 0 failed.**
- `tests/kashi.fcyr` — fuzz over the accessor bounds contract, the PSF
  parser, the Unicode-table map build, codepoint resolution, **multi-byte
  row access (random byte_idx), and ligature lookup** (4000 rounds with
  random widths 1–32; no crash, bounds-safe).
- `tests/kashi.bcyr` — `glyph_row` ~17 ns, `glyph_ptr` ~7 ns,
  `scan_vga_8x16` ~60 µs (now over 224 glyphs); unified dispatch
  `font_row_builtin` ~19 ns, `font_row_runtime` ~42 ns,
  `font_row_runtime_cp` ~62 ns (codepoint binary
  search) (x86_64). CSV trail in [`benchmarks.md`](../benchmarks.md).

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
