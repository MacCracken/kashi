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
- [ ] PSF1 + PSF2 import working and fuzzed.
- [ ] At least one runtime-loaded font registered and rendered end-to-end.
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

### M1 — PSF import path (0.2.0)

- **PSF1** import (`kashi_load_psf` for 0x36 0x04 magic): 8×N glyphs,
  256/512 glyph banks. Parse header, validate `charsize`, copy glyph
  bitmaps into a runtime font slot.
- **PSF2** import (0x72 0xB5 0x4A 0x86 magic): arbitrary `width`×`height`,
  Unicode table (`flags & 0x01`). Parse `headersize`/`length`/`charsize`,
  validate, load glyphs.
- Validation-first parsing — magic, header lengths, glyph-count bounds
  checked before any indexed read (fuzz the parser).
- **Dep gate**: needs stdlib `io` (read a `.psf`) + `fs` and heap for the
  loaded font store — added to `cyrius.cyml [deps]` when this lands.
- Reference: [PSF format note in genesis memory `reference_psf_font_format`].

### M2 — Runtime font registry + additional fonts (0.3.0)

- `kashi_register_font` implemented: register a runtime (loaded) font, get a
  `font_id`; query it through the same accessors as the built-ins.
- Font registry: enumerate available fonts, select active font.
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
