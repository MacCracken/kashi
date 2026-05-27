# kashi — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version, binary
> size, test/assertion counts, in-flight work, consumers) lives in
> [`docs/development/state.md`](docs/development/state.md), bumped every
> release. Do not inline state here — inlined state rots within a minor.

## Project Identity

**kashi** (Sanskrit: काशि, "shining") — the AGNOS console-font subsystem:
bitmap glyph data plus the machinery to load and query console fonts.

- **Type**: Shared library (with a small demo binary)
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Genesis repo**: [agnosticos](https://github.com/MacCracken/agnosticos)
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-documentation.md)
- **Shared crates**: [shared-crates.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/shared-crates.md)

## Goal

Own AGNOS's bitmap console fonts. Ship the glyph byte tables and a
no-stdlib accessor that a **freestanding kernel can `include` directly**,
plus a stdlib-using library face for userland font loading. The glyph data
is the substrate for the framebuffer console, BBS/MUD art, and `iam`
splashes.

## The two faces (load-bearing — read before touching code)

kashi has a **hard, file-level boundary** (see [`docs/adr/0001`](docs/adr/0001-freestanding-font-data-core.md)):

1. **Freestanding core** — `src/font_data.cyr`. NO stdlib, NO heap, NO
   syscalls, NO sakshi — only `store8`/`load8` intrinsics + arithmetic. The
   **agnos kernel** (`[deps] stdlib = []`, vendored klib) `include`s THIS
   FILE ONLY. `cyrius vet src/font_data.cyr` must report "no dependencies".
   **Never add an include or stdlib call here.**
2. **Full library** — `src/lib.cyr` + future modules. stdlib-using; runtime
   font loading, PSF1/PSF2 import, registries. May `include
   "src/font_data.cyr"`, never the reverse.

A new built-in font → freestanding core (so the kernel sees it). Runtime
loading / PSF import → library face.

## Parallel development

A **separate agent develops the full library face** (PSF import, runtime
loading, additional fonts). This repo's 0.1.0 baseline ships the
freestanding core + the booked library skeleton. When working here, do not
assume the library face is unowned — coordinate via the roadmap, and keep
the freestanding boundary intact so the parallel work can't accidentally
break the kernel consumer.

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, binary size, test/assertion counts, in-flight work,
> consumers. Refreshed every release.
> Historical release narrative lives in [`CHANGELOG.md`](CHANGELOG.md).

## Scaffolding

Project was scaffolded with `cyrius init`. **Do not manually create project
structure** — use the tools. If a tool is missing something, fix the tool.

## Quick Start

```sh
cyrius deps                              # resolve stdlib deps
cyrius build src/lib.cyr build/kashi     # build the library
cyrius run   src/main.cyr                # demo (renders 'A' in both fonts)
cyrius test  src/test.cyr                # unit tests
cyrius test  tests/kashi.tcyr            # integration tests
cyrius vet   src/font_data.cyr           # confirm freestanding core has 0 deps
```

## Key Principles

- **Correctness over cleverness** — if it's wrong, the bugs own you.
- **The freestanding boundary is sacred** — `src/font_data.cyr` never gains
  a dependency. Verify with `cyrius vet`.
- **Glyph bytes are load-bearing** — the built-in tables stay byte-for-byte
  identical to what agnos renders; fidelity is asserted in `src/test.cyr`.
- Test after every change, not after the feature is "done".
- ONE change at a time — never bundle unrelated changes.
- Bound every accessor: out-of-range `(font_id, ch, row)` returns safe
  sentinels (row `0`, ptr `0`/null), never a wild deref.
- Programs call `main()` at top level: `var r = main(); syscall(SYS_EXIT, r);`.
- Build with `cyrius build`, never raw `cycc` — the manifest auto-resolves
  deps and prepends includes.
- Every buffer declaration is a contract: module-scope `var X[N]` = N×u64
  (8N bytes); function-local `var X[N]` = N bytes.

## Rules (Hard Constraints)

- **Read the genesis repo's CLAUDE.md first** — [agnosticos/CLAUDE.md](https://github.com/MacCracken/agnosticos/blob/main/CLAUDE.md).
- **Do not commit or push** — the user handles all git operations.
- **NEVER use `gh` CLI** — use `curl` to the GitHub API if needed.
- Do not add unnecessary dependencies. The freestanding core gets ZERO.
- Do not modify `lib/` files (vendored stdlib / dep symlinks).
- Do not skip tests / fuzz / benchmark verification before claiming work.
- Do not use `sys_system()` with unsanitized input — command injection.
- Do not trust external data (file / network / args) without validation.
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in
  `cyrius.cyml` is the source of truth.
- Cyrius enum values are plain non-negative integer literals (no
  arithmetic, no negatives in enum bodies); no negative literals anywhere
  (`0 - N`, not `-N`); no `break` in while loops with `var` declarations.

## Process

### P(-1): Hardening (before any new features, and at minor / v1.0 cuts)

1. **Cleanliness** — `cyrius build`, `cyrius lint`, `cyrius fmt --check`,
   `cyrius vet`; all tests pass.
2. **Benchmark baseline** — `cyrius bench`, save CSV for comparison.
3. **Internal deep review** — gaps, optimizations, correctness, docs.
4. **Security audit** — accessor bounds, buffer sizes, pointer validation.
   File findings in `docs/audit/YYYY-MM-DD-audit.md`.
5. **Additional tests / benchmarks** from findings.
6. **Documentation audit** — ADRs for decisions, architecture notes for
   non-obvious invariants, guides for the public API.

### Work Loop (continuous)

1. **Work phase** — features, roadmap items, bug fixes.
2. **Build check** — `cyrius build` + `cyrius vet src/font_data.cyr`.
3. **Test + benchmark additions** for new code.
4. **Internal review** — performance, memory, correctness, edge cases.
5. **Documentation** — CHANGELOG, `docs/development/state.md`, any ADR earned.
6. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header.

### Task Sizing

- **Low/Medium effort**: batch freely.
- **Large effort**: small bites — break into sub-tasks, verify each.
- **If unsure**: treat it as large.

## Documentation

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*).
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints (*what's true about the code?*).
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos.
- [`docs/examples/`](docs/examples/) — Runnable examples.
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot.
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0.
- [`docs/doc-health.md`](docs/doc-health.md) — Whole-tree doc-currency ledger.
- [`CHANGELOG.md`](CHANGELOG.md) — Source of truth for all changes.

## CHANGELOG Format

Follow [Keep a Changelog](https://keepachangelog.com/). Performance claims
**must** include benchmark numbers. Breaking changes get a **Breaking**
section with a migration guide. Security fixes get a **Security** section.
