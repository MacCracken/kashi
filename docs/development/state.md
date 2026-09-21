# kashi — Current State

> **Last refresh**: 2026-09-21 (**1.0.10** — published bundle, fuzz
> driver, glyph-sheet pin, docs currency) |
> **Refresh cadence**: bumped every release.
>
> CLAUDE.md is preferences/process/procedures (durable); this file is
> **state** (volatile).

## Version

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

- **Cyrius pin**: `6.6.6` (in `cyrius.cyml [package].cyrius`).

## What's implemented

- **Freestanding font-data core** — `src/font_data.cyr`. NO stdlib
  (`cyrius vet` → "no dependencies"). Range `0x20..0xFF` (full CP437,
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
- **Published bundle** — `dist/kashi.cyr` + `dist/kashi.deps`
  (`cyrius distlib` over the `[lib]` modules; 3,742 lines). Tracked and
  released since 1.0.10; what a consumer declares as
  `modules = ["dist/kashi.cyr"]`. CI fails on drift.
- **Demo** — `src/main.cyr`, renders 'A' in both 8-wide built-in fonts.
- **Example / gate** — `docs/examples/glyph_sheet.cyr`, every built-in
  glyph as hex; sha256 pinned in `docs/examples/glyph_sheet.sha256`
  (`b68ca4d9…`) and checked in CI.

## What's booked (not built — future)

Nothing actively booked post-1.0. Ideas noted in
[`roadmap.md`](roadmap.md) as out-of-scope:

- BDF lenient-BBX with per-glyph cell padding (display fonts with
  italic overhangs / descender variation). Would be additive on
  top of the current strict-BBX policy (ADR 0008).
- PCF per-glyph metric variation (analog of the above for PCF;
  ADR 0009 currently strict).
- Text shaping / BiDi / complex-script handling — kashi exposes
  data only; shaping is a separate concern that could live in a
  sibling library.

See [`roadmap.md`](roadmap.md).

## Build & size

- Library build (`cyrius build src/lib.cyr build/kashi`): clean, **256,592 B**.
- Demo (`CYRIUS_DCE=1 cyrius build src/main.cyr`): **137,936 B**.
- Both on cyrius 6.6.6; the growth since the 0.x figures (~84 KB) is the
  stdlib's, not kashi's — the source is unchanged since 1.0.0.

## Tests

- `src/test.cyr` — 393 assertions (incl. the 13 audit regression
  assertions added in 0.8.0 for findings F1, F2, F3, F5, F6, F7, F8,
  F9; F4 was cosmetic). `cyrius test` → **0 failed**.
- `tests/kashi.tcyr` — 57 assertions (structural invariants for all
  three built-ins incl. the 9×16 stride-2 group added in 1.0.10, PSF +
  BDF + PCF file round-trips, PSF + sidecar tab file round-trip,
  missing-file negative cases).
- **450 assertions total, 0 failed.**
- `tests/kashi.fcyr` — the core-accessor bounds contract (three canned
  seeds + 200,000 random `(font, ch, row)` triples across byte-sized,
  full-width i64 and near-edge shapes; eight distinct failure codes),
  the PSF parser (4000 rounds), the BDF parser (2000 rounds), the PCF
  parser (1500 rounds), and the text-tab parser (1000 rounds across
  random / template / mutated / truncated picks). No crashes; all
  accept-paths bounds-safe. Mutation-tested in 1.0.10.
- `tests/kashi.bcyr` — see `docs/benchmarks.md`; the hot path is
  unchanged source-side since 0.4.0, and ~12 % faster toolchain-side
  since 1.0.0 (cyrius 6.0.3 → 6.6.6).
- Glyph-sheet pin — `docs/examples/glyph_sheet.sha256`, checked in CI.

## Cleanliness (P(-1) gates)

- `cyrius build` — clean.
- `cyrius fmt <file> --check` — clean on all src, test and example files.
- `cyrius lint` — 0 warnings, 0 untracked deferrals on all src + example files.
- `cyrius distlib` — tracked `dist/` in sync (CI gate).
- `cyrius vet src/font_data.cyr` — "no dependencies".
- `cyrius vet src/font_psf.cyr` — "no dependencies" (incl. the 0.7.2
  overlong-UTF-8 tightening — pure arithmetic, no new deps).
- `cyrius vet src/font_bdf.cyr` — "no dependencies".
- `cyrius vet src/font_pcf.cyr` — "no dependencies".
- `cyrius vet src/lib.cyr` — 4 deps (lib + core + psf + bdf + pcf),
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
and `io` (`file_read_all` for the three `kashi_load_*_file` and the two
`kashi_attach_*_file` entry points). No git-deps; `cyrius.lock` omitted
by policy. The same eight leaves are what `dist/kashi.deps` names for a
consumer. Unchanged since 0.7.2.

## Consumers

All six take the **freestanding core** (`modules = ["src/font_data.cyr"]`,
vendored as `lib/kashi_font_data.cyr`); none calls a runtime loader —
surveyed 2026-09-21 across `~/Repos`. Pins as of that survey:

| Consumer | kashi | their cyrius | notes |
|---|---|---|---|
| **agnos** (kernel) | `path` only, no tag | 6.6.4 | Integrated at 1.38.0 (M3, 0.7.0); folds `src/font_data.cyr` into the kernel since 1.57.4. `fb_console.cyr` renders via `kashi_glyph_ptr` + `load8`; `gpu.cyr` via `kashi_glyph_row`. |
| dhancha | 1.0.8 | 6.6.4 | Default face for `dh_draw_text`. |
| crab | 1.0.8 | 6.6.4 | |
| jalwa | 1.0.7 | 6.6.3 | |
| aethersafha | 1.0.7 | 6.6.2 | |
| puka | 1.0.6 | 6.6.2 | Five programs include the core. |

The desktop-stack four carry a manifest note to switch to
`modules = ["dist/kashi.cyr"]` the day they need runtime loading — which
resolves only from 1.0.10 on. 1.0.9 and 1.0.10 are data-identical to
1.0.6–1.0.8, so no consumer needs to move for correctness.
