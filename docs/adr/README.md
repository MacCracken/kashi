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
- [0002 — Runtime font registry and unified accessor dispatch](0002-runtime-font-registry.md) — how PSF-loaded fonts (ids ≥ 2) live in a library-face registry beside the built-ins (ids 0,1), reached through one `kashi_font_row`/`kashi_font_ptr` dispatcher; width ≤ 8 and glyph-index addressing for M1.
