# kashi — Current State

> **Last refresh**: 2026-05-28 (**0.8.0** cut — P(-1) hardening +
> security audit) | **Refresh cadence**: bumped every release.
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

**0.8.0** — P(-1) hardening + security audit pass after the 0.7.x
import-formats track closed. Research-driven audit against the
2020–2026 font-parser CVE corpus (libXfont, FreeType, X.Org
advisories, FontForge) produced a 17-point checklist; **9 findings,
0 exploitable**, all fixed: PSF1/PSF2 unknown header bits (F1/F2),
BDF integer overflow path tightened (F3), PCF dead code removed
(F4), PCF format-byte unknown bits rejected in all three required
tables (F5/F6/F7), PCF duplicate TOC entries rejected (F8),
`kashi_attach_unicode_table` no longer strips the existing map on a
failed attach (F9 — the only medium-severity finding). Full audit
in `docs/audit/2026-05-28-audit-0.8.0.md`. Next: 0.9.0 (public API
freeze, every-symbol docs + examples, benchmark trend), then 1.0.0
(clean review + bump).

## Toolchain

- **Cyrius pin**: `6.0.3` (in `cyrius.cyml [package].cyrius`).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr`. NO stdlib
  (`cyaudit vet` → "no dependencies"). Range `0x20..0xFF` (full CP437,
  224 slots). Three built-in fonts:
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
  metric layouts. `cyaudit vet` → "no dependencies".
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
    (built-in 0,1,2 vs runtime ≥ 3).
  - **Multi-byte row access**: `kashi_font_stride` /
    `kashi_glyph_row_byte` in the core; `kashi_rt_font_stride` /
    `kashi_rt_glyph_row_byte` / `kashi_font_row_byte` in the library.
  - **Ligature lookup**: `kashi_font_seq_glyph` (PSF-only; BDF/PCF
    have no ligature concept).
  - Raw glyph-index `kashi_rt_glyph_row` / `kashi_rt_glyph_ptr`;
    `kashi_rt_font_{width,height,count,stride}`; `kashi_font_total`;
    `kashi_set_active_font` / `kashi_active_font`.
  - Scope: width 1–32 (multi-byte rows up to 4 bytes/row).
- **Demo** — `src/main.cyr`, renders 'A' in both built-in fonts.

## What's booked (not built — future)

- **0.9.0**: public API freeze + every exported symbol documented
  with an example + benchmark trend captured (the M4 v1.0
  criteria from `docs/development/roadmap.md`).
- **1.0.0**: clean review + version bump.
- Text shaping / BiDi — out of scope (kashi exposes data only).

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`CYRIUS_DCE=1 cyrius build src/lib.cyr build/kashi`): clean.
- Binary size: ~84 KB DCE (demo + library; the PCF parser adds ~10 KB).

## Tests

- `src/test.cyr` — 393 assertions (incl. the 13 audit regression
  assertions added in 0.8.0 for findings F1, F2, F3, F5, F6, F7, F8,
  F9; F4 was cosmetic). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 49 assertions (structural invariants, PSF +
  BDF + PCF file round-trips, PSF + sidecar tab file round-trip,
  missing-file negative cases).
- **442 assertions total, 0 failed.**
- `tests/kashi.fcyr` — fuzz over the accessor bounds contract, the
  PSF parser (4000 rounds), the BDF parser (2000 rounds), the PCF
  parser (1500 rounds), and the text-tab parser (1000 rounds across
  random / template / mutated / truncated picks). No crashes; all
  accept-paths bounds-safe.
- `tests/kashi.bcyr` — bench numbers unchanged in 0.8.0 (no hot-path
  code modified; the audit fixes are all on the parse-time error
  paths).

## Cleanliness (P(-1) gates)

- `cyrius build` — clean.
- `cyrius fmt <file> --check` — clean on all src + test files.
- `cyrius lint` — 0 warnings on all src files.
- `cyaudit vet src/font_data.cyr` — "no dependencies".
- `cyaudit vet src/font_psf.cyr` — "no dependencies" (incl. the 0.7.2
  overlong-UTF-8 tightening — pure arithmetic, no new deps).
- `cyaudit vet src/font_bdf.cyr` — "no dependencies".
- `cyaudit vet src/font_pcf.cyr` — "no dependencies".
- `cyaudit vet src/lib.cyr` — 4 deps (lib + core + psf + bdf + pcf),
  0 untrusted. The 0.7.2 attach APIs and the text-tab parser are
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
mismatches**. Runtime-loaded fonts (PSF / BDF / PCF) carry their own
glyphs from the source; kashi exercises round-trips on small
synthetic buffers in `src/test.cyr` and `tests/kashi.tcyr`.

## Dependencies

Direct (declared in `cyrius.cyml [deps].stdlib`): `string`, `fmt`,
`io`, `vec`, `alloc`, `syscalls`, `assert`, `bench`. The library face
actively uses `alloc`/`vec` (runtime registry), `string` (`memcpy`),
and `io` (`file_read_all` for the three `kashi_load_*_file` entry
points). No manifest change for 0.7.1.

## Consumers

- **agnos** (kernel) — **integrated at 1.38.0** (M3 done, 0.7.0).
  Consumes `src/font_data.cyr` via the booked
  `[deps.kashi] modules=["src/font_data.cyr"]` contract. No
  kashi-side fixes required; the freestanding boundary held.
