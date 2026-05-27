# Architecture notes

Non-obvious constraints, quirks, and invariants that a reader cannot derive from the code alone. Numbered chronologically — never renumber.

Not decisions (those live in [`../adr/`](../adr/)) and not guides (those live in [`../guides/`](../guides/)). An item here describes *how the world is*, not *what we chose* or *how to do something*.

## Items

- [001 — Module-scope glyph buffers are byte-addressed inside u64-unit BSS](001-module-scope-var-array-byte-addressing.md) — *Affects `src/font_data.cyr`.* `var kashi_font16[1536]` reserves 12 KiB (module-scope `var X[N]` = N×u64), not 1536 bytes; the glyph data is byte-addressed via store8/load8. Mirrors agnos's `fb_font` declaration. Don't "shrink" the arrays.
