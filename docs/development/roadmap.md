# kashi — Roadmap

> **Last Updated**: 2026-10-30 (after 1.3.0)
>
> Forward-facing only. What shipped is in [`CHANGELOG.md`](../../CHANGELOG.md);
> the live state is in [`state.md`](state.md).
>
> **1.3.0 shipped the hardening pass** (audit 2026-10-30). One follow-up
> remains under 1.3.x; the milestones after it each wait for a consumer to
> need them.
>
> The version numbers set an order, not a schedule. If a later milestone's
> trigger arrives first, that milestone ships first and the rest renumber.
> Toolchain pin moves fall in between as patch releases (see *Moving the
> cyrius pin* below).

## 1.3.x — Follow-ups to the 1.3.0 audit

- [ ] **aarch64 CI leg** (audit T8). Every suite passes under
  qemu-aarch64 and the test files compile for `--aarch64` since 1.3.0. CI
  still runs x86_64 only, though agnos builds the core for aarch64 too.

## 1.4.x — Unicode coverage (when a consumer names a glyph)

Additive, the [ADR 0011](../adr/0011-unicode-to-cp437-and-extension-glyphs.md)
way: `0x20..0xFF` stays untouched, and new cells are reachable only by
codepoint.

- **More extension glyphs** after `KASHI_EXT_LAST`.
  - **Candidates:** CP437's `0x01..0x1F` pictures (☺ ♥ ♪ ► ↑ …, which kashi
    has never encoded), or the punctuation a UI emits (‘ ’ “ ” – — •).
  - **Each cell needs:**
    - drawing in all three built-ins;
    - an entry in `kashi_cp437_from_unicode`;
    - grown storage sizes (architecture note 001);
    - a re-pinned glyph sheet.
- **Aliases in `kashi_cp437_from_unicode`**, as Linux's consolemap has:
  U+03B2 'β' → `0xE1` 'ß', U+2126 'Ω' → `0xEA`.
  - Today's map is exactly the CP437 codec's inverse, which is what keeps it
    checkable against an external oracle.
  - Aliases would need their own test table.

**Trigger:** a consumer shows a blank cell or mojibake for a codepoint it
actually draws.

## 1.5.x — Loader breadth (when a font a consumer needs won't load)

- **BDF lenient-BBX.** Accept per-glyph BBX variation by padding into the
  FONTBOUNDINGBOX. Today the parser is strict
  ([ADR 0008](../adr/0008-bdf-import.md)), which rejects display BDFs with
  italic overhangs or descender variation.
- **PCF per-glyph metrics.** The same for PCF, which today requires uniform
  metrics ([ADR 0009](../adr/0009-pcf-import.md)).
- **Streaming loads** (`kashi_load_*_stream`), for a consumer that cannot
  hold the whole file in one buffer.

**Trigger:** a real font a consumer must render fails to load, or needs a
bigger buffer than the consumer can give.

## 1.x.x — Later, gated on something outside kashi

- **A thread-safe library face** (audit L12). Today the contract is
  "serialize registry, load and attach calls" (`docs/api/README.md`,
  *Thread safety*), because kashi has no lock in its dependency set.
  Publishing a font's map and count as one pointer would close the torn-read
  window. **Gate:** a consumer that loads or attaches fonts from more than
  one thread.

- **A fourth built-in font** (id 3, e.g. a denser cell for agnos). It is
  additive in the freestanding core. `KASHI_RT_FONT_BASE` moves 3 → 4, which
  is semver-compatible because consumers reference the constant by name.
  **Gate:** agnos asks for the cell size.
- **6.7 syntax in the core**, e.g. glyph-buffer sizes written as
  `[bytes / 8]` (cyrius 6.7.2). **Gate:** every consumer that compiles
  `src/font_data.cyr` pins cyrius ≥ 6.7.2. Today they are on 6.6.3 – 6.6.20
  ([`state.md`](state.md), *Consumers*).

## 2.0 candidates (breaking; not booked)

- **Prefix the parser struct constants** (audit L11). `BDF_PH_*`,
  `BDF_CTX_*`, `BG_*`, `PCF_PH_*`, `PCF_CTX_*` and the like are documented
  API, but the published bundle exports them unprefixed, so a consumer's
  same-named global silently rebinds them. 1.3.0 prefixed the internal ones
  (`_kashi_rt_registry`, `_KASHI_RT_*`).

## Any time — reactive patches

- A bounds bug in an accessor or parser → a patch plus an audit entry.
- A CVE against PSF, BDF or PCF → an audit and a targeted fix.
- A consumer of `dist/kashi.cyr` hits a problem the scratch-consumer check
  missed → fix it, and add the case to the CI gate.
- The glyph-sheet pin moves under a toolchain bump with no source change →
  stop (procedure step 5).

## Out of scope — will not be added in 1.x

These are explicit non-goals. Bringing them in would change kashi from a
bitmap-font data provider into something else.

- **Outline / vector fonts.** TrueType / OpenType is a different subsystem
  entirely; kashi is bitmap-only.
- **Text shaping / BiDi / complex scripts.** kashi hands back glyph bitmaps.
  Layout and shaping belong in a separate library, which could use kashi as
  its glyph-data provider.
- **Anti-aliasing / subpixel rendering.** Monochrome, 1 bit per pixel only.
- **Rendering policy** (color, scaling, scrolling). The consumer owns it
  (agnos `fb_console.cyr` does this).

## Where to look for…

- **Release history** → [`../../CHANGELOG.md`](../../CHANGELOG.md).
- **Live state** → [`state.md`](state.md).
- **Public API** → [`../api/`](../api/).
- **Decisions** → [`../adr/`](../adr/).
- **Security posture** → [`../../SECURITY.md`](../../SECURITY.md) +
  [`../audit/`](../audit/).
- **Doc currency** → [`../doc-health.md`](../doc-health.md).

---

## Moving the cyrius pin — the procedure

kashi emits fixed data, so a toolchain bump is unusually checkable: the
built-in glyph tables rendered by the new compiler must be byte-identical to
the old render, and the rest is the normal gate set. A pin move is a patch
release. The exception is a move into a new cyrius language minor, which is
cut as a kashi minor (1.2.0, for 6.6.x → 6.7.x). Each past move is recorded in
its CHANGELOG entry.

1. **Screen the toolchain's consumer notes.**
   - **What to read:** cyrius's `docs/development/ecosystem-migration.md`
     lists, release by release, what changes for code built against it.
     Read every section from the current pin up to the new one.
   - **What to grep:** `src/`, `tests/` and `docs/examples/`, for each
     shape the notes name. Also check every new reserved word and every new
     public stdlib name against kashi's identifiers.
   - **The standing profile:**
     - no `struct`, `impl`, `trait`, generic, closure, `async`, tuple,
       slice, typed `*T`, `bool` or `f64`;
     - no `for` loop, no `!`, no compound assignment;
     - negatives spelled `0 - N`, enum members separated, `include` lines
       bare;
     - the longest literal about 17 KB (a single literal over 64 KB needs a
       pin ≥ 6.6.4).

     A change to a shape outside that profile is not kashi's.
   - **A range thousands of lines long** is a subagent's job. Give it the
     file list, the manifest and CI, then settle each item it marks "needs a
     run" with the gates below.
2. **Bump and re-vendor.**
   - Set `cyrius.cyml [package].cyrius`, then run a **plain** `cyrius
     deps`. It re-vendors `lib/` from the new snapshot and re-locks. Commit
     the rewritten `cyrius.lock`.
   - Check: `cyrius deps --verify` reports 0 failed, and `cyrius deps
     --locked` (6.7.6+) is clean.
   - **Not `cyrius deps --lock`.** With the old `lib/` present, it leaves
     the old pin's lock in place and still exits 0. With no `lib/`, it
     writes an empty lock.
   - **Not `cyrius lib sync --full`.** It vendors the whole ~113-file
     snapshot, and the lock then covers files CI never vendors.
   - If `lib/` holds strays, `rm -rf lib && cyrius deps`. `lib/` is
     generated and ignored.
3. **Gate set**, run the way CI runs it:
   - **Build.** `cyrius build` the library and the `CYRIUS_DCE=1` demo, and
     record both sizes. `tests/kashi.fcyr` also builds to `build/kashi`, so
     rebuild the library before comparing sizes or bytes.
   - **Test.** `cyrius test` runs every suite; record each assertion
     count. Then `cyrius fuzz tests/kashi.fcyr`. Repeat both under the
     allocator poison mode, `CYRIUS_POISON=1 cyrius test` and `cyrius fuzz
     --poison tests/kashi.fcyr`, as CI does since 1.3.0. Where qemu is
     available, `cyrius test --aarch64` too.
   - **Vet.** `cyrius vet` on the four freestanding files must print "no
     dependencies".
   - **Format and lint.** `cyrius fmt --check` and `cyrius lint
     --exit-with-count --strict-deferrals` on every source file. A bare
     `cyrius lint` exits 0 on warnings.
   - **Bench.** Build the previous tag's `tests/kashi.bcyr` in a scratch
     copy and run nine interleaved rounds against the new one, both
     `CYRIUS_DCE=1`. Append both sets of rows to
     `docs/benchmarks/history.csv`. The host drifts more between sessions
     than the deltas worth reporting.
   - **Guard mutation.** The fset-guard group sees a removed guard only
     while cycc lays globals out in declaration order. In a scratch copy,
     remove each of the four guards in turn and confirm "guard: every core
     byte unchanged" fails.
4. **The bundle.**
   - **Regenerate:** `cyrius distlib`, then commit `dist/kashi.cyr` +
     `dist/kashi.deps`. CI fails on drift.
   - **Scratch consumer:** build one against the bundle on the oldest pin a
     consumer uses and on the new pin. It declares `[deps.kashi]` with
     `git`, an existing `tag`, `path` = this checkout, and `modules =
     ["dist/kashi.cyr"]`.
   - **Below cyrius 6.7.6,** the `path` wins on a plain `cyrius deps`.
   - **From 6.7.6,** a plain resolve builds the tag. Set `CYRIUS_LOCAL=1`
     for both `deps` and `build`; it resolves into `build/local-deps/lib/`.
     A path-only entry is refused.
   - **Check** that `head -2` of the vendored bundle names the new version.
5. **The glyph sheet.** `cyrius build docs/examples/glyph_sheet.cyr
   build/glyph_sheet && ./build/glyph_sheet | sha256sum` must equal
   `docs/examples/glyph_sheet.sha256`, which CI checks on every push.
   **If the hash moves, stop:** the compiler changed what the font says.
   Bisect the toolchain and file it upstream. Do not re-pin.
6. **Kernel side and consumer pins.** Consumers compile `src/font_data.cyr`
   with *their* pins, not kashi's. Build it freestanding (`cyrius build
   --no-deps`) and run it on each consumer's pin (`state.md`,
   *Consumers*). The core keeps to syntax the oldest of those pins accepts.
7. **Record.**
   - A CHANGELOG entry: what the screen found, the bench table, sizes,
     assertion counts, and the sheet result.
   - A `state.md` Version entry, and `VERSION`.
