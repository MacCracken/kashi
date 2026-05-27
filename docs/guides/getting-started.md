# Getting started with kashi

> **Last Updated**: 2026-05-27

kashi is the AGNOS console-font subsystem. It has **two faces** — read
[`../adr/0001`](../adr/0001-freestanding-font-data-core.md) first if you
haven't.

## Build & run

```sh
cyrius deps                              # resolve stdlib deps
cyrius build src/lib.cyr build/kashi     # build the library
cyrius run   src/main.cyr                # demo: render 'A' in both fonts as ASCII art
cyrius test  src/test.cyr                # unit tests
cyrius test  tests/kashi.tcyr            # integration tests
cyrius bench tests/kashi.bcyr            # benchmarks
cyrius vet   src/font_data.cyr           # confirm the freestanding core has 0 deps
```

## Layout

- `src/font_data.cyr` — the **freestanding core**. NO stdlib. Glyph tables +
  accessors. The agnos kernel `include`s this file directly.
- `src/lib.cyr` — the **library root** (`[build].entry`). stdlib-using;
  `include`s the core and books the runtime surface. Consumers who can link
  the stdlib include this.
- `src/main.cyr` — demo binary (renders 'A' in both fonts).
- `src/test.cyr` — top-level test entry (`[build].test`). Unit assertions.
- `tests/kashi.tcyr` / `.bcyr` / `.fcyr` — integration tests, benchmarks,
  fuzz harness.

## Using the freestanding core (the agnos pattern)

A freestanding consumer adds the core as a single-file module dep:

```toml
[deps.kashi]
path    = "../kashi"
modules = ["src/font_data.cyr"]
```

Then, once at startup, populate the tables; per glyph, read row bytes:

```cyrius
kashi_font_init();                                   # once

var bits = kashi_glyph_row(KASHI_FONT_VGA_8X16, ch, row);   # one row byte
var p    = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, ch);        # base of rows (0 if oob)
```

`kashi_glyph_row` returns one byte per row; bit 7 is the leftmost pixel.
Out-of-range char / font / row returns the safe sentinel (`0` byte, `0`/null
ptr).

## Adding a font

A new **built-in** font goes in the **freestanding core** (`src/font_data.cyr`)
so the kernel can see it:

1. Add an id to the `KashiFontId` enum.
2. Add a `var kashi_fontX[...]` BSS buffer (mind the module-scope u64-unit
   rule — see [`../architecture/001-*`](../architecture/001-module-scope-var-array-byte-addressing.md)).
3. Add a packer + the literal table in `kashi_font_init()`.
4. Wire `kashi_font_{width,height,first,count}` / `kashi_glyph_ptr` for the
   new id.
5. Add fidelity assertions to `src/test.cyr` and run `cyrius test`.

Runtime font *loading* (from a `.psf` file) goes in the **library face** —
see the roadmap M1.

## When a design choice deserves an ADR

See [`../adr/template.md`](../adr/template.md). Non-trivial choices (a new
storage model, a break in the freestanding boundary, an API-surface change)
earn an ADR.
