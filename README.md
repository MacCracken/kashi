# kashi

> **kashi** (काशि, "shining") — the AGNOS console-font subsystem.

kashi owns AGNOS's **bitmap console fonts**: the glyph byte tables a
framebuffer console paints, plus the machinery to load and query fonts.
Written in [Cyrius](https://github.com/MacCracken/cyrius). Part of the
[AGNOS](https://github.com/MacCracken/agnosticos) ecosystem.

## The two faces

kashi exposes **two surfaces** with a hard, file-level boundary, because its
two consumer classes have incompatible link environments (see
[`docs/adr/0001`](docs/adr/0001-freestanding-font-data-core.md)):

| Face | File | Links stdlib? | Consumer |
|------|------|---------------|----------|
| **Freestanding font-data core** | `src/font_data.cyr` | **No** — `store8`/`load8` + arithmetic only | agnos kernel (freestanding, `[deps] stdlib = []`) |
| **Full library** | `src/lib.cyr` (+ future modules) | Yes | userland TTY/console tooling, font-import paths |

The freestanding core is the load-bearing piece: a freestanding kernel
cannot link the Cyrius stdlib, so kashi gives it a pure, single-file
`include` with no heap, no syscalls, and no sakshi.

## Built-in fonts

Both cover printable ASCII `0x20`–`0x7F` (96 glyphs):

| id | Name | Cell | Source |
|----|------|------|--------|
| `KASHI_FONT_VGA_8X16` (0) | IBM VGA BIOS 8×16 ROM font | 8×16 | Public domain (IBM 1981); same byte table Linux's `lib/fonts/font_8x16.c` carries |
| `KASHI_FONT_CGA_8X8` (1) | Hand-drawn CGA-style 8×8 | 8×8 | Public domain (AGNOS original) |

Each glyph is stored as one byte per row, top-to-bottom; bit 7 is the
leftmost pixel.

## Freestanding font-data core API

The whole core is in `src/font_data.cyr` and uses no stdlib. A consumer
calls `kashi_font_init()` once, then queries glyphs:

```cyrius
kashi_font_init();                                  # populate both tables (once)

# One row byte of a glyph (bit 7 = leftmost pixel):
var bits = kashi_glyph_row(KASHI_FONT_VGA_8X16, ch, row);

# Base pointer to a glyph's contiguous row bytes (read `height` of them):
var p = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, ch);   # 0 (null) if out of range

# Metadata:
kashi_font_width(font_id)    # 8
kashi_font_height(font_id)   # 16 (VGA) / 8 (CGA) — also the rows-per-glyph
kashi_font_first(font_id)    # 0x20
kashi_font_count(font_id)    # 96
kashi_glyph_encoded(ch)      # 1 if ch in [0x20, 0x7F], else 0
```

All accessors are **bounds-safe**: an out-of-range char, unknown `font_id`,
or out-of-range `row` returns the safe sentinel (`0` row byte, `0`/null ptr)
— never a wild dereference.

## How agnos consumes the freestanding core

agnos (the kernel) consumes **only** `src/font_data.cyr`, booked for its
1.38.0 cycle. In agnos's `cyrius.cyml`:

```toml
[deps.kashi]
path    = "../kashi"
modules = ["src/font_data.cyr"]
```

Then in the framebuffer console, `kashi_font_init()` runs at console init
and the per-glyph render loop reads `kashi_glyph_row(...)` — replacing the
glyph table currently inlined in `kernel/arch/x86_64/fb_console.cyr`. The
extracted bytes are verified byte-for-byte against that kernel source (see
`src/test.cyr` fidelity assertions), so the swap is behavior-preserving.

## Full library face (in progress)

`src/lib.cyr` re-exports the freestanding core and **books** (skeletons) the
runtime surface — PSF1/PSF2 import (`kashi_load_psf`), runtime font
registration (`kashi_register_font`). As of 0.1.0 these return
`KASHI_ENOSYS` ("booked, not built"); the implementation is tracked in
[`docs/development/roadmap.md`](docs/development/roadmap.md) and built out
along that roadmap.

## Build & test

```sh
cyrius deps                              # resolve stdlib deps
cyrius build src/lib.cyr build/kashi     # build the library
cyrius run   src/main.cyr                # demo: render 'A' in both fonts as ASCII art
cyrius test  src/test.cyr                # unit tests (glyph fidelity, accessors, bounds)
cyrius test  tests/kashi.tcyr            # integration tests (cross-font invariants)
cyrius bench tests/kashi.bcyr            # accessor benchmarks
cyrius vet   src/font_data.cyr           # proves the freestanding core has zero deps
```

## Documentation

- [`docs/adr/`](docs/adr/) — decisions (why the freestanding split).
- [`docs/architecture/`](docs/architecture/) — non-obvious invariants (the
  u64-unit BSS quirk).
- [`docs/guides/getting-started.md`](docs/guides/getting-started.md) — build,
  test, add a font.
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — milestones
  through v1.0.
- [`docs/development/state.md`](docs/development/state.md) — live state.

## License

GPL-3.0-only.
