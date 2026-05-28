# kashi — Public API reference

> **Frozen as of 0.9.0** (2026-05-28). Every exported symbol below
> has a stable signature and semantic guarantee through the 1.0.0
> release and beyond. Breaking changes after 1.0.0 require a major
> version bump.

This is the full reference for kashi's public surface — 44 functions
and 20 enum groups across two faces (freestanding + library). For
narrative how-tos see [`docs/guides/`](../guides/); for the *why*
behind design choices see [`docs/adr/`](../adr/).

## Audience guide

| You are… | Read… |
|---|---|
| A freestanding-kernel author embedding kashi | [`core.md`](core.md) — the 11 accessors + 3 font IDs that ship to the kernel. **Do not read the others.** |
| A userland consumer loading bitmap fonts | [`loading.md`](loading.md), then [`accessors.md`](accessors.md). |
| A consumer attaching external Unicode mappings | [`attach.md`](attach.md) (sidecar tables). |
| Building a custom font-format parser on top of kashi's primitives | [`parsers.md`](parsers.md) — the heapless `*_parse_*` and `*_next_*` cursor APIs. |
| Looking up a result code or constant | [`codes.md`](codes.md). |

## The two faces

kashi has a **hard, file-level boundary** ([ADR 0001](../adr/0001-freestanding-font-data-core.md)):

1. **Freestanding core** — `src/font_data.cyr`. Uses only
   `load8`/`store8` intrinsics + arithmetic. The agnos kernel
   `include`s this file directly. Every symbol it exposes is
   documented in [`core.md`](core.md).
2. **Library face** — `src/lib.cyr` + the parser modules
   (`src/font_psf.cyr`, `src/font_bdf.cyr`, `src/font_pcf.cyr`).
   Stdlib-using. Everything else.

Symbols in the library face will not appear in a freestanding-core
consumer's link set. Mixing the two is fine for stdlib-using
programs (the demo binary does it) — just don't try to call
`kashi_load_psf` from a kernel.

## API doc files

| File | Surface | Symbols |
|---|---|---|
| [`core.md`](core.md) | Freestanding (kernel-safe) | 11 functions + 3 font IDs + 3 range constants |
| [`loading.md`](loading.md) | Runtime font loading | 7 load/register functions |
| [`accessors.md`](accessors.md) | Reading loaded fonts | 13 accessor + metadata functions |
| [`attach.md`](attach.md) | Sidecar Unicode tables | 4 attach functions |
| [`parsers.md`](parsers.md) | Low-level parser primitives | 7 parser functions + 3 struct shapes |
| [`codes.md`](codes.md) | Result codes + constants | 20 enum groups |

## Stability promise

- **Within 1.x**: no removal or signature change to any documented
  symbol; no semantic change to documented return values; no removal
  of any documented constant.
- **Across major versions**: additive changes only (new functions,
  new enum members) within the surface. Removals require a major
  bump.
- **Undocumented internal helpers** (anything starting with
  `_kashi_`) are subject to change at any time and should not be
  called by consumers.

## What's NOT public API

- Internal helpers prefixed with `_kashi_` (e.g.,
  `_kashi_rt_register`, `_kashi_build_umap`, the various
  `_kashi_bdf_*` / `_kashi_pcf_*` / `_kashi_tab_*` primitives).
  These exist for the parser/library implementation and may change.
- `kashi_fset8` / `kashi_fset16` in `src/font_data.cyr`: these are
  init-time helpers used only by `kashi_font_init` to pack the
  built-in glyph tables. Their name omits the `_kashi_` underscore
  convention as an oversight; treat them as internal.
- The bench / test entry points (`tests/kashi.bcyr`,
  `tests/kashi.tcyr`, `tests/kashi.fcyr`).
- Struct field offsets when not specifically listed (e.g.,
  `KashiRtRec` — the runtime registry record fields — are an
  internal layout consumers should not read directly).

## Return-value conventions

Every kashi function returns one of three shapes:

1. **`KASHI_OK = 0` on success, positive code on error** — used by the
   pure parsers (`kashi_psf_parse`, `kashi_bdf_parse_header`, etc.).
   Codes: `KASHI_EINVAL = 102` (bad args), `KASHI_EFORMAT = 103`
   (malformed input). See [`codes.md`](codes.md).
2. **`font_id ≥ KASHI_RT_FONT_BASE` on success, `0 - <code>` on
   error** — used by the library face's load and register
   functions. Test with `if (id < 0)`.
3. **Glyph data or 0/null on out-of-range** — used by the read
   accessors. Out-of-range inputs return safe sentinels (row `0`,
   ptr `0`) rather than wild-deref. Bounds-safe for the full `i64`
   input domain.

See each function's entry for which convention applies.

## Cross-references

- Format-specific guides:
  [PSF](../guides/loading-psf-fonts.md),
  [BDF](../guides/loading-bdf-fonts.md),
  [PCF](../guides/loading-pcf-fonts.md).
- Architecture notes:
  [`architecture/001-module-scope-var-array-byte-addressing.md`](../architecture/001-module-scope-var-array-byte-addressing.md).
- Design records: [`adr/`](../adr/).
- Current state: [`development/state.md`](../development/state.md).
- Audit history: [`audit/`](../audit/).
