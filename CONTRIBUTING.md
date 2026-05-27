# Contributing to kashi

kashi is the AGNOS console-font subsystem. Read the genesis repo's
[`CLAUDE.md`](https://github.com/MacCracken/agnosticos/blob/main/CLAUDE.md)
and this repo's [`CLAUDE.md`](CLAUDE.md) before making changes.

## Development Process

1. Install the Cyrius toolchain at the pinned version (see `cyrius.cyml`
   `[package].cyrius`).
2. Resolve deps: `cyrius deps`
3. Build the library: `cyrius build src/lib.cyr build/kashi`
4. Run tests: `cyrius test src/test.cyr` and `cyrius test tests/kashi.tcyr`
5. Run the demo: `cyrius run src/main.cyr`

## The two faces — know which one you're touching

kashi has a **hard, file-level boundary** (see
[`docs/adr/0001`](docs/adr/0001-freestanding-font-data-core.md)):

- **`src/font_data.cyr`** — the FREESTANDING core. NO stdlib, NO heap, NO
  syscalls, NO sakshi — only `store8`/`load8` intrinsics + arithmetic. The
  agnos kernel `include`s this file directly. **Never add an `include` or a
  stdlib call here.** `cyrius vet src/font_data.cyr` must report "no
  dependencies".
- **`src/lib.cyr`** (and future modules) — the stdlib-using library face.
  May `include "src/font_data.cyr"`, never the reverse.

A change that adds a built-in font goes in the freestanding core (so the
kernel can see it). A change that adds runtime loading / PSF import goes in
the library face.

## Rules

- **Bound every accessor.** For any `(font_id, ch, row)` the accessors must
  return safe sentinels (row `0`, ptr `0`) rather than wild-deref. The fuzz
  harness (`tests/kashi.fcyr`) enforces this.
- **Glyph bytes are load-bearing.** The built-in tables must stay
  byte-for-byte identical to what the agnos kernel renders. If you change a
  glyph, update the corresponding fidelity assertions in `src/test.cyr`.
- Test after every change, not after the feature is "done".
- ONE change at a time — never bundle unrelated changes.
- Library code never panics — errors flow through return codes (and, once
  the runtime surface lands, sakshi `Result`).

## Code Style

- Functions / variables: `snake_case`; enum constants: `UPPER_CASE`.
- Comments explain the *contract* (packing order, the freestanding
  boundary, the u64-unit quirk), not the obvious.
- Formatting is enforced by `cyrius fmt --check`. Run `cyrius fmt <file>
  --check` on every source file before pushing.
- Enum values must be plain non-negative integer literals (Cyrius grammar);
  no negative literals (`0 - N`, not `-N`); no arithmetic in enum bodies.

## License

All contributions are licensed under GPL-3.0-only.
