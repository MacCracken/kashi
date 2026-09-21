# kashi — Roadmap

> **Last Updated**: 2026-09-21 (1.0.10)
>
> **Status: No Open Work.**
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

## What would legitimately reopen this file

- A real consumer hits per-glyph metric variation in a PCF / BDF
  they need to render → lenient-BBX cut.
- A kernel finds an actual bounds bug → patch + audit entry.
- agnos picks up a fourth built-in cell size → freestanding-core
  additive.
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
2. **Bump** `cyrius.cyml [package].cyrius`; `cyrius deps` (6.6.5+ requires
   a re-vendor at every bump — the syscall peers move). On the dev box
   `cyrius lib sync --full` refreshes the whole snapshot; `lib/` is
   ignored, so this is hygiene, not a repo change.
3. **Gate set**: `cyrius build` (library + `CYRIUS_DCE=1` demo; record
   both sizes), `cyrius test` (record the assertion counts),
   `cyrius fuzz tests/kashi.fcyr`, `cyrius vet src/font_data.cyr` →
   "no dependencies", `cyrius fmt --check` and `cyrius lint` on every
   source file, `CYRIUS_DCE=1 cyrius bench tests/kashi.bcyr` (append the
   rows to `docs/benchmarks/history.csv` — the toolchain is the only
   thing that has moved those numbers since 0.4.0).
4. **Regenerate the tracked bundle**: `cyrius distlib`, commit
   `dist/kashi.cyr` + `dist/kashi.deps` (the version stamp moves; CI
   fails on drift).
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

⚠ A refreshed `lib/` carries the toolchain's `sigil.cyr` and `mabda.cyr`
folds, which do not pass `cyrfmt --check`. Harmless here — kashi's format
gate never walks `lib/` — and not ours to fix: the repair belongs in the
sigil and mabda source repos.

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
