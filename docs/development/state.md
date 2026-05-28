# kashi — Current State

> **Last refresh**: 2026-05-28 (**0.7.0** cut — BDF import + agnos
> integration closure) | **Refresh cadence**: bumped every release
> (ideally by the release post-hook).
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**0.7.0** — BDF (Bitmap Distribution Format) import. New heapless
text parser in `src/font_bdf.cyr` (`cyaudit vet` → "no dependencies");
library gains `kashi_load_bdf` / `kashi_load_bdf_file`. agnos has now
consumed the freestanding core end-to-end (M3 done from kashi's side);
0.7.x is the booked post-integration runway for additional bitmap
import formats. Tagged by the user (who handles all git operations);
the tag push drives the release workflow.

## Toolchain

- **Cyrius pin**: `6.0.3` (in `cyrius.cyml [package].cyrius`).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr`. NO stdlib
  (`cyaudit vet` → "no dependencies"). Range `0x20..0xFF` (full CP437,
  224 slots; widened in 0.4.0 per `docs/adr/0004`). Three built-in fonts:
  - `KASHI_FONT_VGA_8X16` (id 0) — IBM VGA BIOS 8×16, 224 glyphs (full
    CP437; PD source = Linux's `font_8x16.c`).
  - `KASHI_FONT_CGA_8X8` (id 1) — dual-sourced 8×8 (hand-drawn AGNOS
    original for `0x20..0x7F`; Linux PD `font_8x8.c` for `0x80..0xFF`),
    224 glyphs total. ADR 0007.
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
- **BDF parser** — `src/font_bdf.cyr` (self-contained, heapless;
  0.7.0, ADR 0008). `kashi_bdf_parse_header` validates `STARTFONT`,
  `FONTBOUNDINGBOX`, `CHARS`, skips `STARTPROPERTIES..ENDPROPERTIES`,
  locates the first `STARTCHAR`. `kashi_bdf_next_glyph` cursor reads
  `ENCODING` + `BBX` (strict-match against `FONTBOUNDINGBOX`) +
  `BITMAP` hex rows. Same posture as the PSF parser: load8 +
  arithmetic only, no stdlib, fuzzed. `cyaudit vet` →
  "no dependencies".
- **Library face** — `src/lib.cyr`. Runtime font registry + the PSF / BDF
  import paths (see `docs/adr/0002`, `docs/adr/0003`, `docs/adr/0005`,
  `docs/adr/0008`):
  - **PSF**: `kashi_load_psf` / `kashi_load_psf_file` /
    `kashi_register_font` → `font_id ≥ 3` or negative `0 - <KashiResult>`.
    A loaded font's Unicode table is built into a sorted codepoint→glyph
    map **and** (0.5.0) a flat list of multi-codepoint ligature records.
  - **BDF** (0.7.0): `kashi_load_bdf` / `kashi_load_bdf_file` →
    `font_id ≥ 3` or negative. Drives the BDF parser glyph-by-glyph,
    over-allocates to the declared `CHARS`, drops `ENCODING -1`
    glyphs, builds the codepoint→glyph map (same shape PSF uses).
    File cap = 4 MiB (`KASHI_BDF_FILE_CAP`; PSF's was 256 KiB).
  - Unified **codepoint-addressed** `kashi_font_row` / `kashi_font_ptr`
    (built-in 0,1,2 vs runtime ≥ 3; runtime resolves codepoint via the
    map, or identity index if the font carried no table).
  - **Multi-byte row access** (0.5.0): `kashi_font_stride` /
    `kashi_glyph_row_byte` in the core; `kashi_rt_font_stride` /
    `kashi_rt_glyph_row_byte` / codepoint `kashi_font_row_byte` in the
    library. Single-byte accessors return the leading byte for wide fonts.
  - **Ligature lookup** (0.5.0): `kashi_font_seq_glyph(id, cps_ptr, len)`
    → glyph index or `0 - 1`. Linear scan; sequences are typically few.
    PSF-only; BDF has no ligature concept.
  - Raw glyph-index `kashi_rt_glyph_row` / `kashi_rt_glyph_ptr`;
    `kashi_rt_font_{width,height,count,stride}`; `kashi_font_total`;
    `kashi_set_active_font` / `kashi_active_font` (default 0 = VGA).
  - Scope: width 1–32 (multi-byte rows up to 4 bytes/row).
- **Demo** — `src/main.cyr`, renders 'A' in both built-in fonts.

## What's booked (not built — future)

- agnos consumption — **done** (kashi side). agnos 1.38.0 consumed
  `src/font_data.cyr` end-to-end via the booked
  `[deps.kashi] modules=["src/font_data.cyr"]` contract.
- 0.7.1: PCF import (X11 compiled binary).
- 0.7.2: PSF u-variant sidecar tables.
- Text shaping / BiDi — out of scope (kashi exposes data only).

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`CYRIUS_DCE=1 cyrius build src/lib.cyr build/kashi`): clean.
- Binary size: ~74 KB DCE (demo + library; the freestanding core itself adds
  ~18 KB of font tables + accessors when included by a consumer).

## Tests

- `src/test.cyr` — 312 assertions (metadata + stride, encoded gate,
  exact-byte fidelity (ASCII + CP437 high-half), accessor bounds, full
  224-glyph coverage, library surface, audit, PSF parse (8-wide + width
  12/32), runtime register/load (incl. 9-wide), Unicode token decode +
  codepoint mapping, raw-index + multi-byte row access, active-font,
  ligature lookup, VGA 9×16 derivation, **BDF parse (valid + 5
  malformed) + load round-trip + skip-unencoded + mismatched-BBX +
  CHARS-too-low**). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 34 assertions (structural invariants over the
  full 0x20..0xFF range, PSF file round-trip + Unicode-table file by
  codepoint, **BDF file round-trip + missing-file negative**).
- **346 assertions total, 0 failed.**
- `tests/kashi.fcyr` — fuzz over the accessor bounds contract, the PSF
  parser, the Unicode-table map build, codepoint resolution, multi-byte
  row access (random byte_idx), ligature lookup, **and the BDF parser**
  (2000 rounds across four pick paths: random / valid template /
  mutated-template / truncated-template; 4000 rounds for PSF; no
  crash, bounds-safe). Exposed and the cut fixed a leading-WS
  infinite-loop in the BDF glyph-block scanner.
- `tests/kashi.bcyr` — `glyph_row` ~17 ns, `glyph_ptr` ~7 ns,
  `scan_vga_8x16` ~60 µs (over 224 glyphs); unified dispatch
  `font_row_builtin` ~19 ns, `font_row_runtime` ~42 ns,
  `font_row_runtime_cp` ~62 ns (x86_64). CSV trail in
  [`benchmarks.md`](../benchmarks.md). BDF load not yet benched (one-shot
  parse-and-register cost; will add in 0.7.x if hot.)

## Cleanliness (P(-1) gates)

- `cyrius build` — clean (no warnings beyond the toolchain pin-drift
  note that survives the 6.0.3 → 6.0.7 transient).
- `cyrius fmt <file> --check` — clean on all src + test files.
- `cyrius lint` — 0 warnings on all src files.
- `cyaudit vet src/font_data.cyr` — "no dependencies" (freestanding
  boundary proven); `cyaudit vet src/font_psf.cyr` — "no dependencies"
  (pure PSF parser); `cyaudit vet src/font_bdf.cyr` — "no dependencies"
  (pure BDF parser; new in 0.7.0); `cyaudit vet src/lib.cyr` — 4 deps
  (lib + core + psf + bdf), 0 untrusted.
  **NB**: invoke `cyaudit` directly — the released 6.0.3 `cyrius vet`
  dispatches to a `cybs` that emits an ELF instead of auditing.
- **Security audit** — `docs/audit/2026-05-28-audit.md` (0.6.0 re-walk);
  BDF parser audit folded into 0.7.0 via the fuzz exposure path (no
  separate audit file this cut). New surface fuzzed; the discovered
  leading-WS infinite loop was fixed pre-release.

## Glyph fidelity

Both built-in tables diffed byte-for-byte against agnos's source — **0
mismatches** across all 96 glyphs in each font. 8×16 from `agnos
kernel/arch/x86_64/fb_console.cyr` (HEAD); 8×8 from the
pre-8×16-replacement commit (`75914e9^`, before "self rolled glyph to
font"). High-half VGA from Linux PD `lib/fonts/font_8x16.c`; high-half
CGA from Linux PD `lib/fonts/font_8x8.c`. BDF-loaded fonts carry their
own glyphs from the source; kashi exercises a round-trip on a small
synthetic BDF in `src/test.cyr` + `tests/kashi.tcyr`.

## Dependencies

Direct (declared in `cyrius.cyml [deps].stdlib`):
`string`, `fmt`, `io`, `vec`, `alloc`, `syscalls`, `assert`, `bench`. The
**library face** actively uses `alloc`/`vec` (runtime registry +
glyph-store copies), `string` (`memcpy`), and `io` (`file_read_all` for
`kashi_load_psf_file` and `kashi_load_bdf_file`) — no manifest change
was needed for 0.7.0. The freestanding core (`src/font_data.cyr`) and
the pure parsers (`src/font_psf.cyr`, `src/font_bdf.cyr`) need none.

## Consumers

- **agnos** (kernel) — **integrated at 1.38.0** (M3 done from kashi's
  side). Consumes `src/font_data.cyr` via
  `[deps.kashi] path="../kashi", modules=["src/font_data.cyr"]`. The
  freestanding boundary held through the integration; no kashi-side
  fixes were required.
