# kashi — Roadmap

> **Last Updated**: 2026-05-28 (**1.0.0 shipped — roadmap closed**)
>
> Milestone plan through v1.0. Live status lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against what
> dependency gates. The freestanding core (M0) is done; M1+ builds out the
> stdlib-using library face.

## v1.0 criteria — all met

- [x] **Public API frozen** — every exported symbol documented and
      tested. (0.9.0; `docs/api/` covers all 45 functions and 20
      enum groups with signatures, param tables, return conventions,
      code examples, and stability notes.)
- [x] **PSF1 + PSF2 import working and fuzzed** — width 1..32 with
      multi-byte rows (ADR 0005), Unicode→codepoint mapping (ADR
      0003), ligature lookup (ADR 0005), sidecar Unicode-table
      attach (ADR 0010).
- [x] **At least one runtime-loaded font registered and rendered
      end-to-end** — PSF, BDF, PCF file round-trips in
      `tests/kashi.tcyr`.
- [x] **Test coverage adequate for the surface area; accessor
      bounds fuzzed** — 442 assertions, 0 failed; 4 fuzz harnesses
      (~7,500 rounds) over the parsers and accessors.
- [x] **Benchmarks captured** in `docs/benchmarks.md` with
      version-over-version comparison from 0.1.0 to 1.0.0.
- [x] **Downstream consumer green** — agnos's framebuffer console
      rendering the freestanding core. *(M3, 2026-05-28 — agnos
      1.38.0 consumed `src/font_data.cyr` end-to-end; the
      freestanding boundary held without kashi-side fixes.)*
- [x] **CHANGELOG complete** from 0.1.0 onward.
- [x] **Security audit pass** — three audits in the trail
      (`docs/audit/2026-05-27`, `2026-05-28`,
      `2026-05-28-audit-0.8.0`); most recent is the
      CVE-research-driven 0.8.0 hardening pass.

## Milestones

### M0 — Freestanding glyph core (0.1.0) — ✅ shipped 2026-05-27 *(this cut)*

- `src/font_data.cyr`: no-stdlib glyph core with the **IBM VGA 8×16** and
  **hand-drawn CGA 8×8** tables (96 glyphs each, ASCII 0x20–0x7F),
  byte-for-byte identical to agnos's `fb_console.cyr` source (verified, 0
  mismatches).
- Bounds-safe accessors: `kashi_font_init`, `kashi_glyph_row`,
  `kashi_glyph_ptr`, `kashi_font_{width,height,first,count}`,
  `kashi_glyph_encoded`.
- `src/lib.cyr` library face re-exports the core and **books** the runtime
  surface (skeleton: `kashi_load_psf`, `kashi_register_font`).
- Demo, unit + integration tests (81 assertions), fuzz harness, benchmarks.
- ADR 0001 (freestanding split), architecture note 001 (u64-unit BSS).

### M1 — PSF import path (0.2.0) — ✅ shipped 0.2.0, 2026-05-27

- **PSF1** (`0x36 0x04`) and **PSF2** (`0x72 0xB5 0x4A 0x86`) import via
  `kashi_load_psf` / `kashi_load_psf_file`, plus `kashi_register_font` for
  raw tables. Parsed by a self-contained `src/font_psf.cyr`; glyph bytes
  copied into an owned runtime store (registry in `src/lib.cyr`).
- Validation-first parsing — magic, version, header size, geometry, glyph
  count, and total length all checked before any glyph read. Fuzzed in
  `tests/kashi.fcyr` (4000 random/truncated/mutated buffers, no crashes).
- Unified `kashi_font_row` / `kashi_font_ptr` dispatch built-in (ids 0,1) vs
  runtime (ids ≥ 2); `kashi_rt_font_{width,height,count}`. See
  [ADR 0002](0002-runtime-font-registry.md) +
  [the guide](../guides/loading-psf-fonts.md).
- **Dep gate**: satisfied by the already-declared `io`/`alloc`/`vec`/`string`
  — no `cyrius.cyml` change was needed.
- **Deferred** (future milestone): glyph width > 8 (multi-byte rows) and the
  PSF Unicode→glyph map (runtime fonts are addressed by glyph index for now).

### M2 — Runtime registry, Unicode addressing, additional fonts — ✅ shipped

- `kashi_register_font` + the runtime registry landed in M1.
- ✅ **PSF Unicode→glyph map** (codepoint addressing for runtime fonts) —
  shipped 0.3.0, 2026-05-27. See
  [ADR 0003](0003-codepoint-addressing-runtime-fonts.md).
- ✅ **Active-font knob** (`kashi_set/active_font`) — shipped 0.4.0,
  2026-05-27. Enumeration of dense ids `[0, kashi_font_total())` is
  implicit through the existing accessors; no separate API needed.
- ✅ **Additional built-in glyph data** — shipped 0.4.0: the VGA 8×16
  built-in extended to the full CP437 range (`0x20..0xFF`, 224 glyphs)
  from Linux's PD source. See [ADR 0004](0004-cp437-glyph-range.md).
- ✅ **Wide-glyph + PSF ligatures** — shipped 0.5.0 (ADR 0005). Multi-byte
  rows (widths 1–32), `kashi_font_stride` / `*_row_byte` accessors,
  PSF2 wider widths accepted, ligature lookup helper. Backward-compatible
  for 8-wide consumers.
- ✅ **VGA 9×16 derived built-in** — shipped 0.5.1 (ADR 0006).
  `KASHI_FONT_VGA_9X16 = 2`, derived at init from the existing VGA 8×16
  via the VGA col-9 replication rule (`0xC0..0xDF`); no new byte tables.
  `KASHI_RT_FONT_BASE` bumped `2 → 3`.
- ✅ **CGA 8×8 high half** — shipped 0.5.2 (ADR 0007). 128 glyphs from
  Linux's PD `font_8x8.c` fill the previously-blank `0x80..0xFF` slots;
  CGA font is now dual-sourced (hand-drawn AGNOS ASCII low half + IBM
  PD high half).

### 0.6.x — Hardening + patch space

- ✅ **0.6.0**: P(-1) hardening pass (audit, fix `kashi_register_font`
  `glyph_count` overflow, strict UTF-8 in the Unicode decoder, stale-doc
  cleanup). See `docs/audit/2026-05-28-audit.md`.
- **0.6.1, 0.6.2, …**: reserved for issue-resolution patches surfaced
  during the agnos integration arc (M3) or by downstream consumers.
  No 0.6.x patch was actually needed — agnos integrated cleanly.

### M3 — Consumption contract hardening + agnos integration — ✅ done 2026-05-28

- agnos **1.38.0** consumed `src/font_data.cyr` end-to-end via the
  booked `[deps.kashi] modules=["src/font_data.cyr"]` contract.
- The freestanding boundary (`cyaudit vet` → "no dependencies") held
  through the integration; no kashi-side fixes were required.
- Glyph-set queries (codepoint coverage, fallback policy) — deferred;
  the existing accessors (`kashi_glyph_encoded`, `kashi_font_total`)
  cover the agnos console's needs. Revisit if a future consumer asks.

### 0.7.x — Additional bitmap import formats (post-agnos-integration)

Booked for after the agnos integration. Each format is a self-contained
library-face addition (the freestanding core stays bitmap-data only);
landing them post-integration de-risks contract churn during the agnos
hookup.

- ✅ **0.7.0 — BDF import**, 2026-05-28 (ADR 0008). Heapless text
  parser in `src/font_bdf.cyr`; library face gains `kashi_load_bdf` /
  `kashi_load_bdf_file`. Strict-BBX policy (every glyph's BBX matches
  `FONTBOUNDINGBOX`), `ENCODING -1` glyphs dropped, 4 MiB file cap.
  Fuzzed (2000 rounds).
- ✅ **0.7.1 — PCF import**, 2026-05-28 (ADR 0009). Heapless binary
  parser in `src/font_pcf.cyr`; library face gains `kashi_load_pcf` /
  `kashi_load_pcf_file`. Strict-uniform-metrics, all four
  byte-order × bit-order combos with canonicalize-on-load (scan-unit
  byte swap + bit reverse), both compressed and uncompressed metrics
  layouts. Required tables: `PCF_METRICS`, `PCF_BITMAPS`,
  `PCF_BDF_ENCODINGS`. Fuzzed (1500 rounds).
- ✅ **0.7.2 — PSF u-variant**, 2026-05-28 (ADR 0010). Sidecar
  Unicode-table attachment APIs in `src/lib.cyr`
  (`kashi_attach_unicode_table*` for binary PSF1/PSF2,
  `kashi_attach_unicode_text*` for `psfgettable` text format) +
  strict overlong-UTF-8 rejection in the PSF2 decoder (audit F3 per
  RFC 3629). The "larger table-walk caps" sub-item was dropped as a
  non-issue on re-examination — the existing walk is buffer-bounded.
  Closes the 0.7.x track.

### 0.8.0 — P(-1) hardening + security audit — ✅ shipped 2026-05-28

- Research-driven audit against the 2020–2026 font-parser CVE corpus
  (libXfont, FreeType, X.Org advisories, FontForge). 17-point
  checklist walk against `src/font_psf.cyr`, `src/font_bdf.cyr`,
  `src/font_pcf.cyr`, and `src/lib.cyr`. 9 findings (F1..F9), 0
  exploitable, all fixed. 13 regression assertions added. Full audit
  in `docs/audit/2026-05-28-audit-0.8.0.md`.

### 0.9.0 — Public API freeze + docs + benchmarks — ✅ shipped 2026-05-28

- Full `docs/api/` written: README + 5 surface files (core, loading,
  accessors, parsers, attach) + codes/constants reference. Every
  exported symbol has a documented signature, parameter table,
  return convention, code example, and stability note.
- Benchmark trend table in `docs/benchmarks.md` covering 0.1.0
  through 0.8.0 with structural-shift call-outs; fresh 0.8.0 row
  added to `docs/benchmarks/history.csv`.
- API surface frozen — no signature or semantic changes through
  the 1.x line.

### M4 — v1.0 freeze (1.0.0) — ✅ shipped 2026-05-28

- Clean review of the 0.9.0 frozen surface: README rewrite from
  0.1.0 placeholder state, SECURITY.md refresh, CONTRIBUTING.md
  polish, stale-comment fixes in `src/font_data.cyr` (agnos
  integration done, CGA high half filled in 0.5.2) and
  `src/font_psf.cyr` (scope comment bumped to 1.0.0).
- API surface verified: all 45 public functions documented; all 20
  enum groups documented; signatures match between code and docs.
- Final P(-1) gate: build clean, fmt clean, lint 0 warnings,
  `cyaudit vet` confirms all four parsers "no dependencies",
  442 / 49 tests pass, fuzz green.
- Version bump to 1.0.0.

## Out of scope (post-1.0, no plans to take on)

- **Outline / vector fonts** — kashi is bitmap-only. TrueType / OpenType
  is a different subsystem entirely.
- **Text shaping / BiDi / complex scripts** — kashi hands back glyph
  bitmaps; layout / shaping belongs in a separate library (one that could
  consume kashi as its glyph-data provider).
- **Anti-aliasing / subpixel rendering** — monochrome 1-bit-per-pixel
  only.
- **Rendering policy** (color, scaling, scrolling) — owned by the
  consumer (agnos `fb_console.cyr` already does this).

## Post-1.0 ideas (out of scope unless reopened)

These come up as natural extensions but aren't on any commitment list:

- **BDF lenient-BBX with cell padding** — accept per-glyph BBX variation
  by padding into the FONTBOUNDINGBOX. Currently strict (ADR 0008). Would
  bring display BDFs with italic overhangs / descender variation into
  scope. Additive on top of the strict policy.
- **PCF per-glyph metric variation** — analog of the above for PCF.
  Currently strict-uniform metrics (ADR 0009).
- **Streaming load APIs** (`kashi_load_*_stream`) — for consumers that
  can't fit the whole file in a single buffer. No concrete need yet.
- **More built-in fonts** — adding a font ID 3 (e.g., a higher-density
  cell or a different character set) is an additive change kashi-side;
  would extend `KASHI_RT_FONT_BASE` from 3 to 4 (semver-compatible
  because the constant is referenced symbolically). No concrete request
  yet.
