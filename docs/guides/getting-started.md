# Getting started with kashi

> **Last Updated**: 2026-09-21 (1.0.9 — rewritten for the frozen 1.0 surface;
> the previous revision described the 0.1.0 scaffold)

kashi is the AGNOS console-font subsystem: three built-in CP437 bitmap fonts
that a framebuffer console paints straight from BSS, plus runtime loading of
PSF / BDF / PCF fonts for userland. The public API has been **frozen since
1.0.0** — the reference is [`../api/`](../api/), and nothing in this guide
goes beyond it.

kashi has **two faces** with a hard, file-level boundary. If you read one
thing first, read [ADR 0001](../adr/0001-freestanding-font-data-core.md):
it explains why the glyph tables live in a file that links nothing, and why
that file is the one the kernel takes.

## Prerequisites

- The Cyrius toolchain at the version pinned in `cyrius.cyml`
  (`[package].cyrius`). The `cyrius` shim re-execs the pinned version when
  run inside the repo, so `cyrius --version` here reports the pin, not
  whatever `~/.cyrius/current` says. CI installs exactly that pin:

  ```sh
  CYRIUS_VERSION="$(grep '^cyrius = ' cyrius.cyml | sed 's/cyrius = "\(.*\)"/\1/')"
  curl -sSf https://raw.githubusercontent.com/MacCracken/cyrius/main/scripts/install.sh | CYRIUS_VERSION="$CYRIUS_VERSION" sh
  ```

- Nothing else. kashi has no git-deps; its only dependencies are the eight
  stdlib leaves in `cyrius.cyml [deps].stdlib`, which `cyrius deps` vendors
  into the (gitignored) `lib/` directory.

## Build, test, verify

```sh
cyrius deps                              # vendor the stdlib leaves into lib/
cyrius build src/lib.cyr build/kashi     # the library face
cyrius run   src/main.cyr                # demo: 'A' in VGA 8x16 and CGA 8x8, as ASCII art
cyrius test                              # unit (src/test.cyr) + integration (tests/kashi.tcyr)
cyrius vet   src/font_data.cyr           # MUST print "no dependencies" — the freestanding boundary
cyrius fuzz  tests/kashi.fcyr            # parser fuzz + accessor bounds contract
cyrius bench tests/kashi.bcyr            # hot-path timings (CI runs it with CYRIUS_DCE=1)
cyrius fmt   src/lib.cyr --check         # per file; CI checks src/*.cyr and tests/*
cyrius lint  src/lib.cyr                 # per file; expected: 0 warnings, 0 untracked deferrals
cyrius distlib                           # regenerate dist/kashi.cyr + dist/kashi.deps (see "Library face")
```

`cyrius test` with no argument runs both suites: the unit suite is the
`[build].test` entry (`src/test.cyr`, ~390 assertions, including the
byte-fidelity pins for the built-in tables) and `tests/*.tcyr` is
auto-discovered (`tests/kashi.tcyr`, structural invariants + file
round-trips). `cyrius test src/test.cyr` / `cyrius test tests/kashi.tcyr`
run them individually.

`.github/workflows/ci.yml` runs exactly these — build smoke (library +
DCE'd demo), tests, fmt/lint/vet, bench, fuzz — so a green local run is a
green CI run.

## Layout

| Path | Face | What it is |
|---|---|---|
| `src/font_data.cyr` | **Freestanding core** | The three built-in glyph tables + the 11 accessors. NO stdlib, NO heap, NO syscalls — `store8`/`load8` + arithmetic only. The agnos kernel `include`s this file directly. |
| `src/font_psf.cyr`, `src/font_bdf.cyr`, `src/font_pcf.cyr` | Library (parsers) | Heapless format parsers — also dependency-free (`load8` + arithmetic), but they serve the library face. |
| `src/lib.cyr` | **Library face** root (`[build].entry`) | Runtime font registry, `kashi_load_*` / `kashi_register_font` / `kashi_attach_unicode_*`, and the unified `kashi_font_*` accessors. Uses the stdlib (allocation, file I/O). `include`s the core and the parsers — never the reverse. |
| `src/main.cyr` | demo | Renders `'A'` in the two 8-wide built-ins. |
| `src/test.cyr` | tests | Unit suite (`[build].test`). |
| `tests/kashi.tcyr` / `.bcyr` / `.fcyr` | tests | Integration suite, benchmarks, fuzz harness. |
| `dist/kashi.cyr`, `dist/kashi.deps` | Library face, bundled | Output of `cyrius distlib` from the `[lib]` module list — the single-file, vendorable form of the library face, plus a sidecar naming the stdlib leaves it needs. Gitignored today (see below). |
| `docs/api/` | reference | The frozen surface: [`core.md`](../api/core.md) for kernels, [`loading.md`](../api/loading.md) / [`accessors.md`](../api/accessors.md) / [`attach.md`](../api/attach.md) for userland, [`parsers.md`](../api/parsers.md), [`codes.md`](../api/codes.md). |
| `docs/adr/`, `docs/architecture/`, `docs/guides/` | docs | Decisions, non-obvious invariants, how-tos. |

## Pick a face

| You want… | Take | Because |
|---|---|---|
| Glyph bytes for a console, splash, or BBS-style art — including in a freestanding kernel | **Freestanding core** (`src/font_data.cyr`) | It is the whole built-in font set with no link cost beyond its own BSS. Every current consumer takes this: the agnos kernel, dhancha, crab, puka, aethersafha, jalwa. |
| To **load fonts at runtime** (PSF / BDF / PCF / raw tables) or attach Unicode sidecar tables | **Library face** (`dist/kashi.cyr`, or `src/lib.cyr` in-repo) | The loaders, the registry, and the codepoint-addressed accessors live here. |

The library face is a measured cost, not a free upgrade: on dhancha the
full face was **+183,360 bytes (+50 %)** over the core, and `CYRIUS_DCE=1`
reclaims none of it (ADR 0001, amendment). Wanting glyph bytes is not a
reason to take it.

## The freestanding core

### Consume it

Declare the core as a single-module dep. This is the shape every consumer
in the stack uses (agnos uses `path` only; the others carry `git` + `tag`
and, by stack convention, `path` too so a sibling checkout wins):

```toml
[deps.kashi]
git     = "https://github.com/MacCracken/kashi.git"
tag     = "1.0.9"
path    = "../kashi"                  # optional; a sibling checkout overrides the clone
modules = ["src/font_data.cyr"]       # vendored as lib/kashi_font_data.cyr
```

`cyrius deps` vendors it as `lib/kashi_font_data.cyr` (named deps are
namespaced `lib/<dep>_<basename>`). Then, in your source:

```cyrius
include "lib/kashi_font_data.cyr"
```

That is the only line a `[deps] stdlib = []` kernel needs; the file has no
dependencies, and `cyrius vet lib/kashi_font_data.cyr` on your side will
say so.

### Render a glyph

Populate the tables once, then read rows. The tables are zeroed BSS until
`kashi_font_init()` runs, so an early access renders blank rather than
crashing; `kashi_font_is_ready()` tells you which state you are in.

```cyrius
kashi_font_init();                                    # once, before the first render

# Hot path: resolve the glyph base once, then read rows with load8.
var p = kashi_glyph_ptr(KASHI_FONT_VGA_8X16, 0x41);   # 'A'; 0 (null) if out of range
if (p != 0) {
    var h = kashi_font_height(KASHI_FONT_VGA_8X16);   # 16
    var row = 0;
    while (row < h) {
        var bits = load8(p + row);                     # bit 7 = leftmost pixel
        # blit `bits` at (x, y + row) ...
        row = row + 1;
    }
}

# Or, per row, bounds-checked every time:
var bits7 = kashi_glyph_row(KASHI_FONT_VGA_8X16, 0x41, 7);   # one byte; 0 if out of range
```

The built-ins:

| id | Constant | Cell | Bytes / row | Source |
|---|---|---|---|---|
| 0 | `KASHI_FONT_VGA_8X16` | 8×16 | 1 | IBM VGA BIOS ROM (PD) — the table agnos's `fb_console.cyr` renders |
| 1 | `KASHI_FONT_CGA_8X8` | 8×8 | 1 | Hand-drawn AGNOS ASCII low half + Linux PD `font_8x8.c` high half |
| 2 | `KASHI_FONT_VGA_9X16` | 9×16 | 2 | Derived at init from font 0 + the VGA col-9 rule (ADR 0006) |

All three cover `0x20..0xFF` (full CP437, `KASHI_GLYPH_COUNT = 224`).
Metadata comes from `kashi_font_width` / `_height` / `_first` / `_count` /
`_stride(font_id)`; `kashi_glyph_encoded(ch)` is the caller-facing range
gate.

**Wide glyphs.** `kashi_font_stride(id)` is the bytes per row — `1` for the
8-wide fonts, `2` for the 9×16. `kashi_glyph_row` returns the *leading*
byte only; for stride 2 read the second byte with
`kashi_glyph_row_byte(id, ch, row, 1)`, or walk `load8(p + row * stride + b)`
from the pointer. For the 9×16, byte 1 holds column 8 in its bit 7.

**Bounds.** Every accessor is safe for the full `i64` input domain:
out-of-range `font_id` / `ch` / `row` / `byte_idx` returns the sentinel
(`0` row byte, `0` null pointer), never a wild dereference. A kernel renders
whatever comes back, so this is the trust boundary — see `SECURITY.md`.

## The library face (userland)

### Consume it

`cyrius distlib` folds the `[lib]` module list into a single file,
`dist/kashi.cyr`, and writes `dist/kashi.deps` beside it naming the eight
stdlib leaves the fold needs. That bundle is what a consumer declares —
**not** `src/lib.cyr`, whose `include "src/font_data.cyr"` is a path
relative to kashi's own root and cannot resolve once vendored (ADR 0001,
amendment):

```toml
[deps.kashi]
git     = "https://github.com/MacCracken/kashi.git"
tag     = "1.0.9"
path    = "../kashi"
modules = ["dist/kashi.cyr"]          # vendored as lib/kashi.cyr
```

`cyrius deps` copies the bundle to `lib/kashi.cyr`, reads the sidecar next
to it, pulls those stdlib leaves into your `lib/` and cross-checks them
against your own `[deps].stdlib`. Then:

```cyrius
include "lib/kashi.cyr"
```

> ⚠ **As of 1.0.9 the bundle is not tracked in git or attached to releases**
> (`dist/` is gitignored), so the `git` + `tag` form above cannot find it —
> the tagged clones have no `dist/`. Until that changes, consume the library
> face through a sibling `path` checkout after running `cyrius distlib`
> there. Tracking `dist/` (the convention every other bundle-publishing
> library in the stack follows) is the open item in
> [`../development/roadmap.md`](../development/roadmap.md).

Inside this repo — the demo, the tests, the benchmarks — `include
"src/lib.cyr"` is the right spelling; that is the one place the
repo-relative include resolves.

### Load a font and read it

```cyrius
include "lib/kashi.cyr"       # or "src/lib.cyr" inside this repo

alloc_init();                 # the library face allocates glyph stores
kashi_font_init();            # built-ins are still there, ids 0..2

var id = kashi_load_psf_file("/usr/share/kbd/consolefonts/default8x16.psf");
if (id < 0) { return 1; }     # 0 - KASHI_EFORMAT / 0 - KASHI_EINVAL

# Unified accessors work for built-in AND runtime ids, addressed by codepoint:
var h = kashi_rt_font_height(id);
var bits = kashi_font_row(id, 0x41, 7);          # one row byte; 0 if out of range

# Hot path, same shape as the core:
var p = kashi_font_ptr(id, 0x41);
```

Runtime fonts get ids `≥ KASHI_RT_FONT_BASE` (`3`), in load order. Every
loader — `kashi_load_psf` / `_bdf` / `_pcf`, their `_file` variants, and
`kashi_register_font` — returns a font id on success or a **negative**
`0 - <KashiResult>` on failure, so `if (id < 0)` is the whole error check.

Format specifics, accepted subsets, and caps are in the per-format guides:
[PSF](loading-psf-fonts.md) (incl. Unicode sidecar tables),
[BDF](loading-bdf-fonts.md), [PCF](loading-pcf-fonts.md). The full
signatures are in [`../api/loading.md`](../api/loading.md) and
[`../api/accessors.md`](../api/accessors.md).

## Adding a built-in font

A new built-in goes in the **freestanding core** so the kernel sees it. The
9×16 (0.5.1, ADR 0006) and the CGA high half (0.5.2, ADR 0007) are the
precedents; follow their shape.

1. **Id.** Add the next `KashiFontId` member (`3`), and bump
   `KASHI_RT_FONT_BASE` in `src/lib.cyr` from `3` to `4` — runtime ids start
   above the built-ins, and the constant is referenced symbolically
   everywhere, so this is semver-compatible.
2. **Buffer.** Add a module-scope BSS buffer sized in **bytes**:
   `var kashi_fontX[glyphs * rows * stride];` (the existing three are
   `kashi_font16[3584]`, `kashi_font8[1792]`, `kashi_font9_16[7168]`). Mind
   the u64-unit rule: module-scope `var X[N]` reserves 8N bytes and kashi
   addresses the first N — see
   [`../architecture/001-*`](../architecture/001-module-scope-var-array-byte-addressing.md).
3. **Table.** Add a packer in the shape of `kashi_fset16` / `kashi_fset8`,
   **keeping the `[KASHI_GLYPH_FIRST, KASHI_GLYPH_LAST]` guard** (audit
   2026-05-27, F1 — it costs nothing at init and stops a table typo from
   corrupting neighbouring BSS), then the literal wall in
   `kashi_font_init()`. A derived font (like the 9×16) computes its rows
   there instead of shipping a table.
4. **Accessors.** Wire the new id into every metadata accessor —
   `kashi_font_width` / `_height` / `_first` / `_count` / `_stride` — and
   `kashi_glyph_ptr`. `kashi_glyph_row` inlines the stride on the hot path:
   extend its `stride` branch if yours is not 1.
5. **Fidelity.** Pin the bytes in `src/test.cyr` the way the VGA `'A'` rows
   are pinned — byte-for-byte against the source table. The built-in tables
   are load-bearing; a glyph that drifts from what agnos renders is a bug
   even if every test still "passes". `tests/kashi.tcyr`'s structural
   invariants (row byte-bounded, pointer monotone, reinit idempotent) must
   hold for the new id, and add the id to the fuzz harness's known-font list
   in `tests/kashi.fcyr` so its bounds contract knows the font exists.
6. **Toolchain caveat.** A single literal over **64 KB** needs a pin
   `≥ 6.6.4` (older read-back defect: the table silently reads from its
   second byte with `rc=0`). The largest literal today is ~17 KB — well
   clear, but font tables only grow. Recorded in the roadmap's pin-bump
   section.
7. **Boundary.** `cyrius vet src/font_data.cyr` must still print
   `no dependencies`. Then `cyrius distlib` so the library-face bundle
   carries the font too.
8. **Docs.** `docs/api/core.md` (the id and the cell), the README's
   built-in table, `CHANGELOG.md`, `docs/development/state.md`, and an ADR
   — a new font is a decision, not a detail.

## Adding a runtime format

Runtime *loading* goes in the **library face**: a heapless parser module
`src/font_<fmt>.cyr` (dependency-free — `cyrius vet` should say so, like
the PSF / BDF / PCF ones), a `kashi_load_<fmt>` / `_file` pair in
`src/lib.cyr` that registers through the existing runtime registry, and the
module added to `cyrius.cyml`'s `[lib] modules` list so `cyrius distlib`
folds it. Add a fuzz loop for the parser in `tests/kashi.fcyr`, a guide
beside this one, and an ADR — [0008 (BDF)](../adr/0008-bdf-import.md) and
[0009 (PCF)](../adr/0009-pcf-import.md) show the expected shape and depth.

## Moving the toolchain pin

kashi emits fixed data, which makes a toolchain bump unusually easy to
verify: bump `[package].cyrius`, `cyrius deps`, build, test, `cyrius
distlib`, then render every glyph of every built-in on both toolchains and
compare byte-for-byte. Any difference means codegen changed what the font
says — stop there. The recipe and the 1.0.9 result are in
[`../development/roadmap.md`](../development/roadmap.md).

## When a change earns an ADR

A new storage model, any change at the freestanding boundary, a new
built-in font, a new format, or anything that touches the frozen surface.
Use [`../adr/template.md`](../adr/template.md); the index in
[`../adr/README.md`](../adr/README.md) lists the ten so far.

## Where to go next

- [`../api/README.md`](../api/README.md) — the audience guide: which
  reference file to read for which job, and the stability promise.
- [`../adr/0001`](../adr/0001-freestanding-font-data-core.md) — why the
  two faces exist, and the 1.0.6 amendment on consuming the library one.
- [`../benchmarks.md`](../benchmarks.md) — what the accessors cost.
- [`../../SECURITY.md`](../../SECURITY.md) and [`../audit/`](../audit/) —
  the bounds contract and the three audits behind it.
- [`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) — the rules of the road.
