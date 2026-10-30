# kashi — Public API reference

> **Frozen as of 0.9.0** (2026-05-28). Every exported symbol below
> has a stable signature and semantic guarantee through the 1.0.0
> release and beyond. Breaking changes after 1.0.0 require a major
> version bump. 1.1.0 (2026-10-09) added one function and one enum
> group and changed no signature
> ([ADR 0011](../adr/0011-unicode-to-cp437-and-extension-glyphs.md)).
> Two results changed with it: `kashi_glyph_encoded` now accepts
> `0x100..0x101`, and the library face maps a built-in's codepoint (see
> [`accessors.md`](accessors.md)).
> 1.1.1 (2026-10-13) changed no public symbol: it right-sized the glyph
> buffers' storage (an internal enum, below). 1.2.0 and 1.2.1
> (2026-10-29) were toolchain-only and changed nothing here. 1.2.2 fixed
> one result: `kashi_pcf_cp_to_idx` returns `0 - 1` for a codepoint above
> U+FFFF, as documented (it used to wrap into the BMP).

This is the full reference for kashi's public surface — 44 public
functions and 21 enum groups across two faces (freestanding + library).
(`src/` declares 46 `fn kashi_*`; the other two are the init-time
`kashi_fset8` / `kashi_fset16` packers, internal — see below.) For
narrative how-tos see [`docs/guides/`](../guides/); for the *why*
behind design choices see [`docs/adr/`](../adr/).

## Audience guide

| You are… | Read… |
|---|---|
| A freestanding-kernel author embedding kashi | [`core.md`](core.md) — the 11 accessors, the Unicode → cell map and the 3 font IDs that ship to the kernel. **Do not read the others.** |
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
   Stdlib-using. Everything else. Consumed as the `cyrius distlib`
   bundle `dist/kashi.cyr` (`modules = ["dist/kashi.cyr"]`, vendored as
   `lib/kashi.cyr`; ADR 0001 amendment) — `src/lib.cyr` itself only
   resolves inside this repo.

Symbols in the library face will not appear in a freestanding-core
consumer's link set. Mixing the two is fine for stdlib-using
programs (the demo binary does it) — just don't try to call
`kashi_load_psf` from a kernel.

## API doc files

| File | Surface | Symbols |
|---|---|---|
| [`core.md`](core.md) | Freestanding (kernel-safe) | 12 functions + 3 font IDs + 3 CP437 range + 3 extension constants |
| [`loading.md`](loading.md) | Runtime font loading | 7 load/register functions |
| [`accessors.md`](accessors.md) | Reading loaded fonts | 14 accessor + metadata functions (13 entries — the active-font get/set pair shares one) |
| [`attach.md`](attach.md) | Sidecar Unicode tables | 4 attach functions |
| [`parsers.md`](parsers.md) | Low-level parser primitives | 7 parser functions + 3 struct shapes |
| [`codes.md`](codes.md) | Result codes + constants | 21 enum groups |

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
  convention as an oversight; treat them as internal. (The 1.1.0
  index builders `_kashi_cp437_ent` / `_kashi_cp437_rev8` and the
  `kashi_cp437_rev` / `kashi_cp437_n` globals are internal too.)
- `_KashiFontStore` (`_KASHI_FONT16_SLOTS` / `_KASHI_FONT8_SLOTS` /
  `_KASHI_FONT9_16_SLOTS`, 1.1.1) in `src/font_data.cyr`: the glyph
  buffers' declared sizes in u64 slots, named so the tests check the same
  numbers the declarations use. They change whenever a cell is added; for
  a glyph's size use `kashi_font_height` × `kashi_font_stride`. Not
  counted among the 21 enum groups.
- The extension **cell numbers** themselves (`0x100`, `0x101`): get a
  cell from `kashi_cp437_from_unicode`, or enumerate
  `KASHI_EXT_FIRST..KASHI_EXT_LAST`; the block may grow.
- The bench / test entry points (`tests/kashi.bcyr`,
  `tests/kashi.tcyr`, `tests/kashi.fcyr`).
- Struct field offsets when not specifically listed (e.g.,
  `KashiRtRec` — the runtime registry record fields — are an
  internal layout consumers should not read directly).

## Return-value conventions

Most kashi functions return one of three shapes:

1. **`OK = 0` on success, positive code on error** — used by the pure
   parsers. Each parser has its own codes: `kashi_psf_parse` returns
   `KASHI_EINVAL = 102` (bad args) / `KASHI_EFORMAT = 103` (malformed
   input), and the BDF / PCF parsers return `KASHI_BDF_*` /
   `KASHI_PCF_*` (`EINVAL = 1`, `EFORMAT = 2`). See [`codes.md`](codes.md).
2. **`font_id ≥ KASHI_RT_FONT_BASE` on success, `0 - <code>` on
   error** — used by the library face's load and register
   functions. Test with `if (id < 0)`.
3. **Glyph data or 0/null on out-of-range** — used by the read
   accessors. Out-of-range inputs return safe sentinels (row `0`,
   ptr `0`) rather than wild-deref. Bounds-safe for the full `i64`
   input domain.

The exceptions:
- The four `kashi_attach_unicode_*` functions and
  `kashi_set_active_font` return `KASHI_OK` or `0 - <code>`.
- `kashi_pcf_cp_to_idx` and `kashi_font_seq_glyph` return an index, or
  `0 - 1` on a miss.
- `kashi_psf_uni_token` writes its result through `out` and always
  returns 0.

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
