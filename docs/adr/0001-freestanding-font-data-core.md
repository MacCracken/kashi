# 0001 — Freestanding font-data core, separate from the stdlib library face

**Status**: Accepted
**Date**: 2026-05-27

## Context

kashi is the AGNOS console-font subsystem. Its first and most important
consumer is the **agnos kernel**, which renders glyph text on the
framebuffer console (`kernel/arch/x86_64/fb_console.cyr`). kashi is being
split out of that kernel so the glyph data and font machinery live in one
owned place instead of being inlined in the kernel.

But agnos is a **freestanding kernel**: its `cyrius.cyml` declares
`[deps] stdlib = []` and it vendors its own `klib/`. It physically cannot
link the Cyrius stdlib — no `alloc`, no `io`, no `fmt`, no syscalls beyond
its own inline asm, no sakshi. So agnos cannot consume any kashi surface
that touches the stdlib.

Meanwhile the *other* future consumers — userland TTY/console tooling, the
PSF font-import path, runtime font loading — want the full stdlib (file I/O
to read a `.psf`, heap to hold a loaded font, fmt for diagnostics). Those
two consumer classes have irreconcilable link environments.

A naive single-module library would force one of two bad outcomes: either
the whole library avoids the stdlib forever (crippling the userland face),
or it uses the stdlib and the kernel can't consume it at all (defeating the
extraction).

## Decision

Split kashi into **two faces** with a hard, file-level boundary:

1. **Freestanding font-data core** — `src/font_data.cyr`. Pure Cyrius using
   only the `store8` / `load8` pointer intrinsics and integer arithmetic.
   **No** stdlib includes, **no** heap, **no** syscalls, **no** sakshi. It
   carries the two glyph byte tables (IBM VGA 8x16, hand-drawn CGA 8x8) and
   the no-stdlib accessors (`kashi_font_init`, `kashi_glyph_row`,
   `kashi_glyph_ptr`, the `kashi_font_*` metadata helpers). A freestanding
   kernel `include`s **this file only**, via:

   ```toml
   [deps.kashi]
   path    = "../kashi"
   modules = ["src/font_data.cyr"]
   ```

2. **Full library face** — `src/lib.cyr` and future modules. stdlib-using.
   Re-exports the freestanding core and adds the runtime surface (PSF1/PSF2
   import, runtime font registration, font registries). This is what
   stdlib-linked consumers `include`.

The boundary is enforced by convention and by `cyrius vet`: `font_data.cyr`
must report "no dependencies". `src/lib.cyr` may include `font_data.cyr` but
**never** the reverse.

In scope: the two built-in fonts and their accessors are the 0.1.0
deliverable. Out of scope for this ADR: the contents of the full library
face (PSF import etc.) — that's roadmap M1+ and a separate concern.

## Consequences

- **Positive** — agnos gets a single-file, zero-dep `include` that drops in
  where its inlined font table is today, with identical bytes (verified
  byte-for-byte against the kernel source). The userland face is free to use
  the full stdlib without compromising the kernel consumer.
- **Positive** — the freestanding contract is *testable*: `cyrius vet
  src/font_data.cyr` reporting "no dependencies" is a CI-checkable proof the
  boundary hasn't leaked.
- **Negative** — two surfaces to keep coherent. A new built-in font must be
  added to the freestanding core (not the library face) to be visible to the
  kernel. The roadmap documents which face owns which capability.
- **Negative** — the freestanding core can't use convenient stdlib helpers
  (`fmt`, bounds-checked slices); it open-codes shift/mask packing. This is
  the price of being kernel-consumable and is contained to one file.
- **Neutral** — the core uses module-scope BSS buffers (`var kashi_font16
  [1536]`) populated by an explicit `kashi_font_init()` call, mirroring how
  agnos's `fb_console_init()` already populates `fb_font`. Freestanding code
  has no constructor mechanism, so an explicit init is the idiomatic shape.

## Alternatives considered

- **Single stdlib-using library.** Rejected: agnos literally cannot link it.
  This is the whole reason for the split.
- **Compile-time-constant accessor (giant `if`/`match` over u64 literals,
  no buffer, no init call).** Considered — it would remove the init step.
  Rejected for 0.1.0: 96 glyphs × 2 fonts of branch dispatch is larger and
  slower than a BSS table + indexed load, and it diverges from agnos's
  existing `fset16`-into-buffer model that we want to mirror exactly for a
  clean drop-in. May revisit if the init call ever proves awkward for a
  consumer.
- **Embed the font data in agnos's klib and have kashi depend on agnos.**
  Rejected: inverts the ownership (kashi owns fonts, not the kernel) and
  couples kashi to a freestanding kernel's build, which is backwards.
- **Two separate repos (a `kashi-core` and a `kashi`).** Rejected as
  over-engineering: one repo with a file-level boundary is enough, and the
  monolithic-by-design AGNOS principle says coupling lives at the contract
  (the `modules = [...]` list), not at the repo split.
