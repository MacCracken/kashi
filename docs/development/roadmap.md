# kashi — Roadmap

> **Last Updated**: 2026-05-27
>
> Milestone plan through v1.0. Live status lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against what
> dependency gates. The freestanding core (M0) is done; M1+ builds out the
> stdlib-using library face.

## v1.0 criteria

- [ ] Public API frozen — every exported symbol (freestanding core +
      library face) documented and tested.
- [x] PSF1 + PSF2 import working and fuzzed. *(M1, 2026-05-27 — width ≤ 8;
      wider glyphs still to come. Unicode→codepoint mapping landed in M2.)*
- [x] At least one runtime-loaded font registered and rendered end-to-end.
      *(M2 — PSF file round-trip, addressed by codepoint; tests/kashi.tcyr.)*
- [ ] Test coverage adequate for the surface area; accessor bounds fuzzed.
- [ ] Benchmarks captured in `docs/benchmarks.md` with version-over-version
      comparison.
- [ ] **Downstream consumer green** — agnos's framebuffer console rendering
      the freestanding core (booked at agnos **1.38.0**).
- [ ] CHANGELOG complete from 0.1.0 onward.
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`).

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

### M2 — Runtime registry niceties + Unicode + additional fonts (0.3.0)

- `kashi_register_font` + the runtime registry landed in M1.
- ✅ **PSF Unicode→glyph map** (codepoint addressing for runtime fonts) —
  landed 2026-05-27, unreleased. See
  [ADR 0003](0003-codepoint-addressing-runtime-fonts.md).
- Still open: registry niceties (enumerate available fonts, select an
  "active" font); glyph width > 8 (multi-byte rows); PSF Unicode
  sequence/ligature mappings.
- Additional built-in fonts (candidates: a denser 8×16, a wider 9×16 with
  proper box-drawing, a small 6×8) — added to the **freestanding core** so
  the kernel can use them.

### M3 — Consumption contract hardening + agnos integration (0.4.0)

- Lock the `kashi`↔agnos contract: confirm `[deps.kashi] modules=
  ["src/font_data.cyr"]` drops into agnos's `fb_console.cyr` cleanly and the
  rendered output is identical (golden-image / per-glyph diff in CI).
- agnos consumes the freestanding core at its **1.38.0** (booked; agnos-side
  work, not kashi's).
- Glyph-set queries useful to consumers (`iam`, BBS/MUD art): codepoint
  coverage, fallback policy for unencoded chars.

### M4 — v1.0 freeze (1.0.0)

- P(-1) closeout: full audit, frozen API, benchmark trend, security pass.
- Every public symbol documented with an example; `docs/api/` if the surface
  warrants it.

## Out of scope (for v1.0)

- **Outline / vector fonts** — kashi is bitmap-only. TrueType/OpenType is a
  different subsystem entirely.
- **Text shaping / BiDi / complex scripts** — kashi hands back glyph
  bitmaps; layout/shaping is the consumer's (or a future shaping lib's) job.
- **Anti-aliasing / subpixel rendering** — monochrome 1-bit-per-pixel bitmap
  fonts only.
- **Rendering policy** (color, scaling, scrolling) — owned by the consumer
  (agnos `fb_console.cyr` already does this), not by kashi.
