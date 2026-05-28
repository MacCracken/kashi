# 0002 — Runtime font registry and unified accessor dispatch

**Status**: Accepted
**Date**: 2026-05-27

## Context

M1 (0.2.0) adds runtime font loading: import PSF1/PSF2 bitmap fonts
(`kashi_load_psf`) and register raw glyph tables (`kashi_register_font`)
into a runtime store, alongside the two built-in fonts (VGA 8×16 = id 0,
CGA 8×8 = id 1) that live in the freestanding core (`src/font_data.cyr`).

Three forces shape the design:

1. **The freestanding boundary is sacred (ADR 0001).** The runtime path uses
   the heap, `vec`, and file I/O — none of which may touch
   `src/font_data.cyr`. The core's accessors
   (`kashi_glyph_ptr`/`kashi_glyph_row`/`kashi_font_*`) know only ids 0 and 1
   and cannot be taught about a registry without breaking the boundary. They
   also cannot be *redefined* in the library face (a duplicate symbol is a
   compile error).
2. **kashi's accessor model is one byte per glyph row** (8-pixel width). The
   built-ins are 8 wide; the accessors return a single `load8` byte.
3. **PSF glyph order is not guaranteed to be ASCII/Unicode** without parsing
   the optional PSF Unicode table.

## Decision

**A runtime registry in the library face, with a unified dispatcher.**

- **ID space.** Built-in fonts keep ids 0/1. Runtime fonts get ids starting
  at `KASHI_RT_FONT_BASE = 2`, assigned in registration order. The registry
  is a module-scope `vec` (in `src/lib.cyr`) of heap records
  `{width,height,count,charsize,data}`; each font **owns a copy** of its
  glyph bytes (so the caller's input buffer can be freed/reused).

- **Pure parser, separate file.** PSF byte parsing lives in a self-contained,
  heapless `src/font_psf.cyr` (`kashi_psf_parse` → validated header struct).
  It uses only `load8`/`load32`/`load64` + arithmetic, so it lints standalone
  and fuzzes in isolation. The registry/allocation/dispatch (the stdlib-using
  part) lives in `src/lib.cyr`. Include chain:
  `lib.cyr → {font_data.cyr, font_psf.cyr}`; the leaves never include each
  other or the reverse.

- **Unified dispatcher.** New library accessors `kashi_font_row(id,ch,row)`
  and `kashi_font_ptr(id,ch)` route `id < 2` to the freestanding core and
  `id ≥ 2` to the registry, so a console has **one** hot-path entry point for
  any font. Runtime metadata is `kashi_rt_font_{width,height,count}`.
  `kashi_font_total()` spans both id spaces.

- **Width ≤ 8 only (M1).** PSF1 is always 8 wide. PSF2 with width > 8 is
  rejected with `KASHI_EFORMAT` — supporting it needs a multi-byte-row
  accessor, deferred to a later milestone.

- **Glyph-index addressing (M1).** Runtime glyphs are addressed by raw glyph
  index `[0,count)`. The PSF Unicode table's *presence* is recorded but the
  map is not built — codepoint→glyph mapping is deferred.

- **Return convention.** `kashi_load_psf`/`kashi_register_font` return a
  `font_id ≥ 2` on success or `0 - <KashiResult>` (negative) on failure, so a
  caller can branch with `if (id < 0)`. (`0 - N` form satisfies the
  no-negative-literals rule; `KashiResult` codes stay non-negative.)

## Consequences

- **Positive** — the freestanding core is untouched; `cyrius vet
  src/font_data.cyr` still reports "no dependencies". The parser is a small,
  pure, fuzzable unit. A consumer renders any font through one accessor.
- **Negative / wart** — the dispatcher's 2nd argument is **overloaded**: an
  ASCII codepoint (`0x20..0x7F`) for built-ins, but a glyph **index**
  `[0,count)` for runtime fonts. This is the price of deferring the Unicode
  map; a future milestone that adds codepoint→glyph mapping for runtime fonts
  will make the argument uniformly a codepoint.
- **Negative** — runtime glyph stores are heap allocations that are never
  individually freed (the stdlib bump allocator reclaims at reset). Fine for
  a load-once-at-boot consumer; a font-churning consumer would want an
  eviction story (out of scope for M1).
- **Neutral** — dispatch adds ~2 ns to the built-in path and the runtime path
  costs ~28 ns/row vs ~18 ns for the direct core accessor (registry lookup +
  bounds), per `docs/benchmarks.md`.

## Alternatives considered

- **Teach the core accessors about the registry.** Rejected: breaks the
  freestanding boundary (ADR 0001) — the kernel could not include the core.
- **Parallel `kashi_rt_*` accessor set, no unified dispatcher; consumer
  branches on id range.** Workable but pushes id-space dispatch onto every
  consumer; a console wants a single entry point. (The `kashi_rt_font_*`
  metadata accessors *do* exist for runtime fonts; only the hot-path
  row/ptr accessors are unified.)
- **Support wide (>8 px) glyphs now.** Rejected for M1: needs a multi-byte-row
  accessor and reworks the storage/contract; deferred.
- **Parse the Unicode table and address by codepoint in M1.** Rejected for
  M1 scope: more parsing/storage/tests; glyph-index addressing is the
  simplest correct cut and unblocks the import path now.
