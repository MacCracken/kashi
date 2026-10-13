# kashi — Roadmap

> **Last Updated**: 2026-10-13 (1.1.1)
>
> **Status: No Open Work.**
>
> 1.1.1 (2026-10-13) moved the pin 6.6.6 → 6.6.20 and fixed the one issue
> filed against 1.1.0: the three glyph buffers reserved 8× the bytes they
> use — 88 KB of zeros in every binary that links the core, found by agnos
> against its kernel-image ceiling
> ([issue](issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md),
> architecture note 001). No glyph, symbol or accessor result changed.
>
> 1.1.0 (2026-10-09) answered the first concrete consumer need since 1.0:
> the compositor truncates window titles with U+2026 '…', which CP437 has
> no cell for, and every consumer was drawing UTF-8 as CP437 mojibake.
> kashi now owns the Unicode → CP437 mapping (`kashi_cp437_from_unicode`)
> and an extension block for glyphs CP437 lacks ('…', '�'), and the
> library face's built-in codepoint bug is fixed
> ([ADR 0011](../adr/0011-unicode-to-cp437-and-extension-glyphs.md)).
> Additive; nothing is booked after it.
>
> kashi reached 1.0.0 with every v1 criterion met. The release
> narrative lives in [`CHANGELOG.md`](../../CHANGELOG.md); this file
> tracks only **forward-facing** work.
>
> The 2026-09-21 review (at 1.0.9) found three items that were not
> feature work but were real — the 1.0.6 bundle had never actually been
> published, the fuzz harness false-failed on the 9×16 and never drove
> the core accessors randomly, and the toolchain-bump verification tool
> lived outside the repo — and 1.0.10 closed all three along with the
> documentation-currency backlog from `docs/doc-health.md`. Nothing is
> booked after it.
>
> Future work will be added here when a concrete need surfaces —
> not before. The shape below is "what could reopen this file,"
> not "what's planned."

## Out of scope — committed (will not be added in 1.x)

These are explicit non-goals. Bringing them in would change kashi
from a bitmap-font data provider into something else.

- **Outline / vector fonts** — TrueType / OpenType is a different
  subsystem entirely. kashi is bitmap-only.
- **Text shaping / BiDi / complex scripts** — kashi hands back glyph
  bitmaps; layout / shaping belongs in a separate library (one that
  could consume kashi as its glyph-data provider).
- **Anti-aliasing / subpixel rendering** — monochrome 1-bit-per-pixel
  only.
- **Rendering policy** (color, scaling, scrolling) — owned by the
  consumer (agnos `fb_console.cyr` does this).

## Out of scope — unbooked (no plans, but additive if asked)

These would be additive on top of the 1.0 surface and don't break
the API freeze. Each is named so that *if* a concrete consumer
need shows up, the scope is already roughly understood.

- **BDF lenient-BBX with cell padding** — accept per-glyph BBX
  variation by padding into the FONTBOUNDINGBOX. Currently strict
  (ADR 0008). Would bring display BDFs with italic overhangs /
  descender variation into scope.
- **PCF per-glyph metric variation** — analog of the above for PCF.
  Currently strict-uniform metrics (ADR 0009).
- **Streaming load APIs** (`kashi_load_*_stream`) — for consumers
  that can't fit the whole file in a single buffer. No concrete
  need yet.
- **Additional built-in fonts** — adding a font id 3 (e.g., a
  higher-density cell or a different character set) is an additive
  change kashi-side; would bump `KASHI_RT_FONT_BASE` from 3 to 4.
  Semver-compatible because the constant is referenced symbolically.
- **More extension glyphs** (ADR 0011's block) — CP437's `0x01..0x1F`
  pictures (☺ ♥ ♪ ► ↑ …, which kashi has never encoded), or punctuation a
  consumer's UI emits (‘ ’ “ ” – — •). Append cells after `KASHI_EXT_LAST`,
  draw them in every built-in font, extend `kashi_cp437_from_unicode` and
  re-pin the sheet. Wait for a consumer to name the glyph.
- **Aliases in `kashi_cp437_from_unicode`** — e.g. U+03B2 'β' → `0xE1`
  'ß', U+2126 'Ω' → `0xEA`, as Linux's consolemap does. The 1.1.0 map is
  exactly the codec's inverse so it stays checkable against an external
  oracle; aliases would be additive.

## What would legitimately reopen this file

- A real consumer hits per-glyph metric variation in a PCF / BDF
  they need to render → lenient-BBX cut.
- A kernel finds an actual bounds bug → patch + audit entry.
- agnos picks up a fourth built-in cell size → freestanding-core
  additive.
- A consumer needs a glyph CP437 does not have → an extension cell, the
  ADR 0011 way (codepoint-reachable only; `0x20..0xFF` untouched).
- A CVE drops against a font format kashi parses → audit + targeted
  fix.
- The glyph-sheet pin (`docs/examples/glyph_sheet.sha256`) moves under
  a toolchain bump with no source change → the compiler changed what
  the font says. Stop, bisect the toolchain, file it upstream; do not
  re-pin.
- A consumer switches to `modules = ["dist/kashi.cyr"]` and hits a
  problem the scratch-consumer check in 1.0.10 did not → the bundle's
  first real user; fix and add the case to the CI gate.

Anything else is either shaping-layer work (separate library) or
out-of-scope feature creep.

## Where to look for…

- **Release history** → [`../../CHANGELOG.md`](../../CHANGELOG.md).
- **Live state** → [`state.md`](state.md).
- **Public API** → [`../api/`](../api/).
- **Decisions** → [`../adr/`](../adr/).
- **Security posture** → [`../../SECURITY.md`](../../SECURITY.md) +
  [`../audit/`](../audit/).

---

## Moving the cyrius pin — the procedure

kashi emits fixed data, which makes a toolchain bump unusually
checkable: the built-in glyph tables rendered by the new compiler must
be byte-identical to the old render, and everything else is the normal
gate set. Toolchain bumps are patch releases (1.0.1–1.0.9 were all this).

1. **Read the toolchain changelog for consumer-visible changes** and
   grep `src/` for each shape it names before bumping — the 6.6.6
   precedent below lists what that looked like. Zero `struct`s, no
   `async` / `operator` / SIMD fns, no top-level block `var`s, no file
   writes, and a longest literal of ~17 KB is the standing profile;
   anything the changelog flags outside that profile is not kashi's.
2. **Bump** `cyrius.cyml [package].cyrius`, then `cyrius deps`: it
   re-vendors `lib/` from the new pin's snapshot (6.6.5+ requires that at
   every bump — the syscall peers move) and, because the pin moved,
   re-locks — commit the rewritten `cyrius.lock` (tracked since 1.1.1; at
   an unchanged pin it refuses a snapshot file whose hash differs, which is
   CI's integrity check on the toolchain it downloads). Do **not**
   `cyrius lib sync --full`: it vendors the whole 113-file snapshot and the
   next `cyrius deps` locks all of it, a lock CI (which vendors only the
   ~24 declared files) cannot `--verify`. If `lib/` holds strays, `rm -rf
   lib && cyrius deps` — `lib/` is generated and ignored. `cyrius deps
   --verify` must report 0 failed.
3. **Gate set**: `cyrius build` (library + `CYRIUS_DCE=1` demo; record
   both sizes), `cyrius test` (runs both suites from 6.6.17 — record both
   assertion counts), `cyrius fuzz tests/kashi.fcyr`, `cyrius vet
   src/font_data.cyr` → "no dependencies", `cyrius fmt --check` and
   `cyrius lint --exit-with-count --strict-deferrals` on every source file
   (a bare `cyrius lint` exits 0 on warnings), `CYRIUS_DCE=1 cyrius bench
   tests/kashi.bcyr` (append the rows to `docs/benchmarks/history.csv` —
   until 1.1.1's storage fix the toolchain was the only thing that had
   moved those numbers since 0.4.0; build the old pin's binary in a
   scratch copy and interleave the runs).
4. **Regenerate the tracked bundle**: `cyrius distlib`, commit
   `dist/kashi.cyr` + `dist/kashi.deps` (the version stamp moves; CI
   fails on drift). A distlib change can move more than the stamp — 6.6.18
   added the requires block and shrank the sidecar to the leaves the
   compile-verify needs; check a scratch consumer still builds against the
   bundle on the oldest pin a consumer uses.
5. **The glyph sheet**: `cyrius build docs/examples/glyph_sheet.cyr
   build/glyph_sheet && ./build/glyph_sheet | sha256sum` must equal
   `docs/examples/glyph_sheet.sha256`. CI does this on every push, so a
   bump PR proves it by going green. **If the hash moves, stop** — that
   is the reopen trigger above, not a re-pin.
6. **Kernel side**: `src/font_data.cyr` is compiled by agnos with *its*
   pin, not kashi's. The bump is complete only when the kernel's pin
   accepts the table too — confirm before tagging.
7. **Record**: CHANGELOG entry with the bench comparison, sizes,
   assertion counts and the sheet result; `state.md` Version entry;
   `VERSION`.

(Through 1.1.0 step 2 said to `cyrius lib sync --full` on the dev box, and a
note here warned that the full snapshot's `sigil.cyr` / `mabda.cyr` folds fail
`cyrfmt --check`. With `lib/` holding only what `cyrius deps` vendors, neither
file is there.)

### Precedent — 6.6.6 → 6.6.20 (1.1.1, 2026-10-13)

Fourteen toolchain releases at once. **Checked against the cyrius changelog
6.6.7 – 6.6.20** (the per-release consumer notes, and every mention of
kashi in the cyrius tree): none of the new compiler refusals has a shape in
`src/`, `tests/` or `docs/examples/` — no top-level `defer`, `#define` /
`#if`, string escapes outside the known set, `sizeof` / `mulh64` /
`fncall*` identifiers, array initializer lists, typed array elements,
nested fns, redefined fns, `syscall()` with more than 6 arguments, or
stray `@`. What did reach kashi, each handled in 1.1.1:

- **6.6.17 — a bare `cyrius test` runs `[build].test` first.** 448 + 145
  in two files before the fix (454 + 145 after); CI's separate unit step
  dropped, as the 6.6.17 note below asked.
- **6.6.9 — `cyrius deps` writes a `cyrius.lock` for a stdlib-only project**
  ("commit it"). kashi's policy had been to omit one; every stdlib-only
  sibling (dhancha, crab, puka, jalwa, aethersafha, rekha) tracks it, and a
  tampered hash makes `cyrius deps` exit 1. Tracked now — step 2.
- **6.6.18 — distlib's requires block and compile-verified sidecar.**
  `dist/kashi.deps` 8 leaves → 4 (`vec alloc string io`; `fmt`,
  `syscalls`, `assert`, `bench` were never the library face's), and the
  bundle opens with the four `include "lib/…"` lines. A scratch consumer
  (`[deps.kashi] path` + `modules = ["dist/kashi.cyr"]`) builds and runs on
  6.6.20 **and** 6.6.6.
- **6.6.13 — globals are aligned (16 on x86 for anything over 8 bytes),
  and the gap between two globals is the compiler's.** kashi never reached
  one global from another; the 1.1.1 storage tests are written not to
  depend on it.
- **6.6.11 — test grading on the assert summary; lint fails on compiler
  refusals.** Both suites print the summary; lint clean.

**Result:** the glyph sheet was **byte-identical** — 1.1.0 rendered by 6.6.6
and by 6.6.20 both hash to the pin `5544d63f…`, and so does 1.1.1;
`cyrius vet` "no dependencies" for all four freestanding files; fuzz clean;
bench flat (every row within ±2 % across 1.1.0/6.6.6, 1.1.0/6.6.20 and
1.1.1/6.6.20, nine interleaved runs each). The library grew 262,656 →
288,528 B and the DCE'd demo 144,072 → 145,376 B on the toolchain alone
(stdlib growth), before the storage fix took 88,576 B off each. Kernel
side: the 1.1.1 core compiles freestanding (`--no-deps`) on agnos's 6.6.6
pin and passes both suites there. Full numbers in `CHANGELOG.md` *1.1.1*.

### Precedent — 6.6.4 → 6.6.6 (1.0.9, 2026-09-21)

**What was checked** (7 `.cyr` under `src/`; vendored `lib/` excluded):

- **Windows write corruption: not a kashi exposure.** `O_APPEND` /
  `O_TRUNC` appear 52 times, all inside the vendored `lib/` fold, zero in
  `src/`. kashi never opens a file for writing; its only file I/O is the
  read-only `file_read_all` calls behind the `_file` loaders.
- **The read-side PE fix (`file_read_all` no longer requests write
  access) matches kashi's access pattern but kashi has no Windows
  target**: CI is `ubuntu-latest` only and `src/` carries no
  `CYRIUS_TARGET_*` branch.
- **No shape the new refusals catch.** Zero `struct` declarations, no
  `async fn`, no `operator` fn, no SIMD-returning fn, no `var` inside a
  top-level block, no raw `SYS_STATFS`, no `lib/regression.cyr` consumer,
  no duplicate top-level global, no symlink in `lib/`.
- **The literal-size question, measured.** `src/font_data.cyr` carries
  kashi's longest literal at **17,121 bytes** — the largest in this slice
  of the ecosystem and well under the **64 KB** threshold of the 6.6.4
  read-back defect. ⚠ Font tables only grow: a single literal past 64 KB
  needs a pin ≥ 6.6.4 or the table silently reads back from its second
  byte with `rc=0`. Now a line in the getting-started "adding a font"
  checklist.

**Result:** every step ran as written. The glyph sheet — 3 fonts × 224
glyphs, every row byte — was **byte-identical** between the 6.6.4 and
6.6.6 builds (sha256 `b68ca4d9…`, since pinned); `dist/kashi.cyr`
differed only in its version stamp; 393 + 49 assertions passed; `cyrius
vet` stayed dependency-free; bench flat within noise (and ~12 % faster
than the 1.0.0 / 6.0.3 rows on the sweep and the codepoint search —
codegen, not source). Full numbers in `CHANGELOG.md` *1.0.9*.

## Recorded by cyrius 6.6.17 (2026-10-05) — for the next cyrius pin move

✅ **Adopted in 1.1.1** (pin 6.6.20): the bare run covers both suites and CI's separate unit step is gone. Kept for
the record. Docs-only note from the cyrius 6.6.17 lanes; each item was this repo's to adopt when it pinned ≥ 6.6.17.

- **Bare `cyrius test` runs `[build] test = "src/test.cyr"` from 6.6.17** (cyrius m5), BEFORE the `tests/`
  walk. Checked 2026-10-09 in a scratch copy pinned to 6.6.17: a bare run reports 448 + 145 passed, two files.
  Until 1.1.0 CI ran only the bare command, so on 6.6.6 it gated `tests/kashi.tcyr` alone and the unit
  assertions never ran in CI; since 1.1.0 the test job also runs `cyrius test src/test.cyr` as its own step.
  At the pin move that step only repeats the bare run — **drop it in the same change**.
