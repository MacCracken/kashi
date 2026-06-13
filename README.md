# kashi

> **kashi** (काशि, "shining") — the AGNOS console-font subsystem.

kashi owns AGNOS's **bitmap console fonts**: the glyph byte tables a
framebuffer console paints, plus the machinery to load and query
fonts at runtime. Written in
[Cyrius](https://github.com/MacCracken/cyrius). Part of the
[AGNOS](https://github.com/MacCracken/agnosticos) ecosystem.

**Status**: 1.0.0 — public API frozen. The full surface is
documented in [`docs/api/`](docs/api/); stability promise is in
[`docs/api/README.md`](docs/api/README.md).

## The two faces

kashi exposes **two surfaces** with a hard, file-level boundary
([ADR 0001](docs/adr/0001-freestanding-font-data-core.md)):

| Face | File | Links stdlib? | Consumer |
|------|------|---------------|----------|
| **Freestanding font-data core** | `src/font_data.cyr` | **No** — `store8`/`load8` + arithmetic only | agnos kernel (freestanding, `[deps] stdlib = []`) |
| **Full library** | `src/lib.cyr` + parser modules | Yes | userland TTY/console tooling, font-import paths |

The freestanding core is the load-bearing piece: a freestanding kernel
cannot link the Cyrius stdlib, so kashi gives it a pure, single-file
`include` with no heap, no syscalls, and no sakshi. The agnos kernel
consumes only that file, at agnos 1.38.0 (integrated 2026-05).

Each parser module (`src/font_psf.cyr`, `src/font_bdf.cyr`,
`src/font_pcf.cyr`) is *also* dependency-free — `cyaudit vet` reports
"no dependencies" for each. The boundary covers them too.

## Built-in fonts

All cover the codepoint range `0x20..0xFF` (full CP437, 224 glyphs):

| id | Constant | Cell | Source |
|----|----------|------|--------|
| 0 | `KASHI_FONT_VGA_8X16` | 8×16 | IBM VGA BIOS ROM (PD); same byte table as Linux's `font_8x16.c` |
| 1 | `KASHI_FONT_CGA_8X8` | 8×8 | Hand-drawn AGNOS-original ASCII low half + Linux PD `font_8x8.c` high half |
| 2 | `KASHI_FONT_VGA_9X16` | 9×16 | Derived at init from VGA 8×16 + the VGA col-9 replication rule (ADR 0006) |

Each glyph row is one byte for the 8-wide fonts (two bytes for 9×16);
bit 7 of each byte is the leftmost pixel.

Runtime-loaded fonts get ids `≥ KASHI_RT_FONT_BASE` (= 3 as of 1.0.0),
assigned in load order.

## Loading runtime fonts

| Format | API | File cap |
|--------|-----|----------|
| **PSF1 / PSF2** (Linux console) | `kashi_load_psf(buf, len)` / `kashi_load_psf_file(path)` | 256 KiB |
| **BDF** (X11 source) | `kashi_load_bdf(buf, len)` / `kashi_load_bdf_file(path)` | 4 MiB |
| **PCF** (X11 compiled) | `kashi_load_pcf(buf, len)` / `kashi_load_pcf_file(path)` | 4 MiB |
| **Raw byte tables** | `kashi_register_font(w, h, bytes, count)` | — |
| **Sidecar Unicode tables** (psfaddtable convention) | `kashi_attach_unicode_table*` (binary) / `kashi_attach_unicode_text*` (text) | 256 KiB |

All loaders return a `font_id ≥ KASHI_RT_FONT_BASE` on success or
`0 - <KashiResult>` on failure (negative). See
[`docs/api/loading.md`](docs/api/loading.md).

## Quick start (freestanding core)

```cyrius
include "src/font_data.cyr"

kashi_font_init();   # populate the three built-in tables (once at boot)

# Hot-path render: resolve once, read rows directly.
var p = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, 0x41);   # 'A'
if (p != 0) {
    var row = 0;
    while (row < 16) {
        var bits = load8(p + row);   # bit 7 = leftmost pixel
        # blit `bits` at (col, row + base_y) ...
        row = row + 1;
    }
}
```

All accessors are **bounds-safe** for the full `i64` input domain
— out-of-range inputs return safe sentinels (`0` row byte, `0`/null
ptr), never a wild dereference. Fuzzed (`tests/kashi.fcyr`) over the
full integer range of each argument.

## Quick start (userland — load a PSF font)

```cyrius
include "src/lib.cyr"

var id = kashi_load_psf_file("/usr/share/kbd/consolefonts/default8x16.psf");
if (id < 0) { return 1; }   # 0 - KASHI_EFORMAT / 0 - KASHI_EINVAL

# Render 'A' at row 7:
var bits = kashi_font_row(id, 0x41, 7);

# Hot path: resolve once via kashi_font_ptr, then load8 in a tight loop.
var p = kashi_font_ptr(id, 0x41);
# ... see docs/api/accessors.md ...
```

## How agnos consumes the freestanding core

agnos's kernel integrated kashi at agnos 1.38.0 (2026-05-28). Its
`cyrius.cyml`:

```toml
[deps.kashi]
path    = "../kashi"
modules = ["src/font_data.cyr"]
```

Only `src/font_data.cyr` is `include`d — the entire library face
(parsers, allocations, file I/O) is invisible to the kernel link.
`kashi_font_init()` runs at console init; the per-glyph render loop
uses `kashi_glyph_ptr` + `load8`.

## Build & test

```sh
cyrius deps                                # resolve stdlib deps
cyrius build src/lib.cyr build/kashi       # build the library
cyrius run   src/main.cyr                  # demo: render 'A' in both 8-wide built-ins
cyrius test  src/test.cyr                  # unit tests
cyrius test  tests/kashi.tcyr              # integration tests
cyrius bench tests/kashi.bcyr              # accessor benchmarks
cyrius vet   src/font_data.cyr             # proves freestanding core has zero deps
```

## Documentation

- **[`docs/api/`](docs/api/)** — public API reference (frozen 1.0).
  6 files covering all 45 functions and 20 enum groups, with
  signature, params, return convention, code example, stability
  note per symbol.
- [`docs/guides/`](docs/guides/) — task-oriented how-tos: getting
  started, loading [PSF](docs/guides/loading-psf-fonts.md),
  [BDF](docs/guides/loading-bdf-fonts.md),
  [PCF](docs/guides/loading-pcf-fonts.md).
- [`docs/adr/`](docs/adr/) — 10 architecture decisions, *why* each
  major choice (the freestanding split, codepoint addressing,
  wide-glyph rows, BDF / PCF / PSF u-variant scope).
- [`docs/architecture/`](docs/architecture/) — non-obvious invariants
  (the u64-unit BSS byte-addressing quirk).
- [`docs/benchmarks.md`](docs/benchmarks.md) — hot-path numbers, 0.x
  trend table.
- [`docs/audit/`](docs/audit/) — three security audits
  (2026-05-27, 2026-05-28 P(-1), 2026-05-28-audit-0.8.0 — the 0.8.0
  CVE-research-driven pass).
- [`docs/development/state.md`](docs/development/state.md) — live
  state snapshot.
- [`CHANGELOG.md`](CHANGELOG.md) — full release history.

## Stability

- **API frozen** as of 0.9.0; locked-in at 1.0.0.
- Within the 1.x line: no signature changes, no removals, no
  semantic changes to documented behavior. New functions and new
  enum members are additive.
- Anything prefixed with `_kashi_` is internal and may change at
  any time.
- See [`docs/api/README.md`](docs/api/README.md) for the full
  stability promise.

## License

GPL-3.0-only.
