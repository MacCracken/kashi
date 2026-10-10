# Architecture notes

Non-obvious constraints, quirks, and invariants that a reader cannot derive from the code alone. Numbered chronologically — never renumber.

Not decisions (those live in [`../adr/`](../adr/)) and not guides (those live in [`../guides/`](../guides/)). An item here describes *how the world is*, not *what we chose* or *how to do something*.

## Items

- [001 — Module-scope glyph buffers are byte-addressed inside u64-unit BSS](001-module-scope-var-array-byte-addressing.md) — *Affects `src/font_data.cyr`.* `var kashi_font16[3616]` reserves 28.25 KiB (module-scope `var X[N]` = N×u64), not 3616 bytes; the glyph data (224 CP437 + 2 extension glyphs) is byte-addressed via store8/load8. Same for `kashi_font8[1808]` and the 2-byte-stride `kashi_font9_16[7232]`. Mirrors agnos's `fb_font` declaration. Don't "shrink" the arrays. The stated exception (1.1.0): the Unicode reverse index `kashi_cp437_rev[128]` counts entries, one u64 slot each.
