# Architecture Decision Records

Decisions about kashi — what we chose, the context, and the consequences we accept. Use these when a future reader would reasonably ask *"why did we do it this way?"*

## Conventions

- **Filename**: `NNNN-kebab-case-title.md`, zero-padded to four digits. Never renumber.
- **One decision per ADR.** If a decision supersedes a prior one, add a new ADR and set the old one's status to `Superseded by NNNN`.
- **Status lifecycle**: `Proposed` → `Accepted` → (optionally) `Superseded` or `Deprecated`.
- Use [`template.md`](template.md) as the starting point.

## ADR vs. architecture note vs. guide

| Kind | Lives in | Answers |
|---|---|---|
| ADR | `docs/adr/` | *Why did we choose X over Y?* |
| Architecture note | `docs/architecture/` | *What non-obvious constraint is true about the code?* |
| Guide | `docs/guides/` | *How do I do X?* |

## Index

- [0001 — Freestanding font-data core, separate from the stdlib library face](0001-freestanding-font-data-core.md) — why kashi ships a no-stdlib `src/font_data.cyr` the agnos kernel can `include` directly, distinct from the stdlib-using `src/lib.cyr` library face.
- [0002 — Runtime font registry and unified accessor dispatch](0002-runtime-font-registry.md) — how PSF-loaded fonts (ids ≥ 2) live in a library-face registry beside the built-ins (ids 0,1), reached through one `kashi_font_row`/`kashi_font_ptr` dispatcher; width ≤ 8 and glyph-index addressing for M1. *(Addressing refined by 0003.)*
- [0003 — Codepoint addressing for runtime fonts via the PSF Unicode table](0003-codepoint-addressing-runtime-fonts.md) — parse the PSF Unicode table into a codepoint→glyph map so `kashi_font_*` is codepoint-addressed for runtime fonts too (resolving the 0002 wart); raw-index access moves to `kashi_rt_glyph_*`.
- [0004 — Extend the freestanding range to full CP437 (VGA 8×16)](0004-cp437-glyph-range.md) — widen `KASHI_GLYPH_LAST` to `0xFF` (96 → 224 glyphs) so the kernel console gets line/box drawing, shading, and accented letters; VGA bytes from Linux's PD source, CGA high-half intentionally blank.
- [0005 — Wide-glyph rows + PSF ligature lookup](0005-wide-glyph-and-ligatures.md) — additive `kashi_font_stride` / `*_row_byte` for multi-byte rows (widths 1–32 px); PSF2 width >8 now accepted; PSF Unicode sequences harvested into a per-font list, looked up via `kashi_font_seq_glyph`. Backward-compatible for 8-wide consumers.
- [0006 — VGA 9×16 derived built-in (col-9 replication rule)](0006-vga-9x16-derived-builtin.md) — `KASHI_FONT_VGA_9X16 = 2` derived at init from the existing VGA 8×16 + the VGA hardware's box-drawing col-9 rule. No new byte table to ship. `KASHI_RT_FONT_BASE` bumps `2 → 3` (pre-1.0 breaking for hard-coded users).
- [0007 — CGA 8×8 high half from Linux's public-domain `font_8x8.c`](0007-cga-high-half-from-linux-pd.md) — fill the previously-blank CGA `0x80..0xFF` slots from Linux's PD CGA ROM bytes; the CGA font becomes dual-sourced (hand-drawn AGNOS ASCII low half + IBM PD high half), documented and accepted.
- [0008 — BDF import](0008-bdf-import.md) — heapless text-based parser in `src/font_bdf.cyr` for X11 BDF source files; strict-BBX (per-glyph BBX must match `FONTBOUNDINGBOX`), 4 MiB file cap, `ENCODING -1` glyphs skipped. Library face only; reuses the PSF parsed-header struct (`KIND=3` for BDF) and the existing runtime registry + cp-map machinery.
