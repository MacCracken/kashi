# Architecture notes

Non-obvious constraints, quirks, and invariants that a reader cannot derive from the code alone. Numbered chronologically — never renumber.

Not decisions (those live in [`../adr/`](../adr/)) and not guides (those live in [`../guides/`](../guides/)). An item here describes *how the world is*, not *what we chose* or *how to do something*.

## Items

- [001 — Module-scope glyph buffers are byte-addressed inside u64-slot arrays](001-module-scope-var-array-byte-addressing.md) — *Affects `src/font_data.cyr`.* Module-scope `var X[N]` = N u64 slots = 8N bytes, so each glyph buffer declares bytes-used ÷ 8: `kashi_font16[452]` = 3,616 B (226 glyphs: 224 CP437 + 2 extension), `kashi_font8[226]`, the 2-byte-stride `kashi_font9_16[904]` — sizes named in the internal `_KashiFontStore` so the tests check the same N. Byte-addressed via store8/load8. Until 1.1.1 N was the byte count and reserved 8× (88,576 B of dead weight in every binary — cycc writes module-scope storage into the file image). A new cell grows each N by rows × stride / 8. `kashi_cp437_rev[128]` counts entries, one u64 slot each.
