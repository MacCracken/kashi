---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-10-29 (**1.2.0** — the pin move to cyrius 6.7.9, no
> source change: the rows it touched re-graded — root files, architecture
> note 001, roadmap, state, getting-started, benchmarks).
> Previous: 2026-10-13 (**1.1.1** — the pin move to cyrius 6.6.20 and
> the glyph-storage fix: architecture note 001 rewritten, every doc that
> quoted the old sizes or called the storage "BSS" corrected, the issue
> resolved, `CLAUDE.md` Fresh again without an edit).
> Previous: 2026-10-09 (**1.1.0** — every doc the Unicode → CP437
> map and the extension glyphs touched re-graded in the same change: ADR
> 0011 added, 0003 / 0004 / 0010 forward-pointed, the API reference,
> guides, architecture note, SECURITY, state, roadmap, benchmarks).
> Previous: 2026-09-21 (1.0.10 — the 1.0.9 sweep's backlog cleared).
> **Refresh cadence**: when a doc is touched, update its row. Full-tree
> sweep at minor-version closeouts.
>
> **Scope**: this repo only (`kashi`) — the `docs/` tree plus root-level
> files (README, CLAUDE.md, CHANGELOG, CONTRIBUTING, SECURITY,
> CODE_OF_CONDUCT, LICENSE, VERSION, cyrius.cyml). Sibling-repo docs are not
> audited here. Cross-repo version drift lives in [`development/state.md`](development/state.md).
>
> **Location**: `docs/doc-health.md` (whole-tree scope), **not** under
> `docs/development/` — the ledger sweeps the whole tree and the location
> matches the scope.

This is a **ledger**, not a one-time audit. Rewrite-in-place as docs change.
`docs/` is now 37 files (2026-10-13) — at the ~30-file threshold where the agnosticos
pattern switches to tier roll-ups. Per-file rows are kept this sweep
because the API is frozen and churn since 1.0.0 has been toolchain-only;
revisit if the tree grows again.

**Status key**: ✅ Fresh = describes the 1.1.0 tree (rows not touched by
1.1.0 were last graded against 1.0.9/1.0.10 and nothing in them moved). 🟡 Stale = at least
one statement no longer describes the tree (the note says which). 🔵
Evergreen = standard text, re-read annually. ❓ Open question = a decision
is owed, not an edit. Historical records (ADRs, audits) are ✅ when they
accurately record the decision/pass as made; later changes are noted, not
counted against them.

---

## At a glance — 2026-10-09 inventory (1.1.0)

45 files in scope (44 at 1.0.10, + ADR 0011). 1.1.0 is a feature cut
that reopened the tree; every file it made stale was fixed in the same
change. One pre-existing stale line surfaced and is left open on purpose:
`CLAUDE.md`'s Quick Start (operator-owned). ⚠ This is a touched-rows
refresh, not the full-tree re-read the minor-closeout rule asks for —
the rows 1.1.0 did not touch keep their 1.0.10 grading.

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 44 | Everything with a date: the root files (`CLAUDE.md` again since 1.1.1 — the pin move made its Quick Start true), all 13 `adr/`, `architecture/`, `development/` (incl. the one issue, resolved), the 4 guides, the example pair, the 3 audits, benchmarks, all 7 `api/`, this ledger. |
| 🟡 **Stale** | 0 | — (1 at 1.1.0: `CLAUDE.md`'s bare-`cyrius test` comment, false on 6.6.6, true on the 6.6.20 pin; 10 at the 1.0.9 sweep, all fixed in 1.0.10.) |
| 🟠 **Read-through outstanding** | 0 files | ⚠ The full-tree re-read that the refresh cadence asks for at a minor closeout was not done for 1.2.0. It is a touched-rows refresh, because 1.2.0 changed no source. |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — (`docs/examples/` resolved: `glyph_sheet.cyr` shipped) |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-10-29 | ✅ Fresh | 1.2.0: status line adds the move to the cyrius 6.7 line (no source change); `tag = "1.2.0"`. Before: 1.1.1: status line adds the storage cut (101 KB → 12.7 KB, −88 KB per binary); the bundle paragraph names the four sidecar leaves and the requires block (was "eight leaves"); `tag = "1.1.1"`; quick start's `cyrius test` is one line (both suites from 6.6.17); architecture pointer describes the u64-slot unit (was "u64-unit BSS quirk"). 1.1.0 content (Unicode section, cells + extension) holds. |
| `CHANGELOG.md` | 2026-10-29 | ✅ Fresh | `[0.1.0]`–`[1.2.0]`. 1.2.0 = cyrius 6.6.20 → 6.7.9 (minor, to mark the 6.7 language line): the 6.7.0–6.7.9 shape check, the re-lock (14 of 24 hashes), sizes (+16 B each), the guard mutation re-check, the bench A/B, the kernel-side and consumer-pin note. Before: `[0.1.0]`–`[1.1.1]`. 1.1.1 = cyrius 6.6.6 → 6.6.20, the glyph-storage fix (sizes, the guard-test rework, mutation results, agnos headroom), the tracked lock, distlib's 4-leaf sidecar, the CI lint gate armed, bench three-way A/B. |
| `CLAUDE.md` | 2026-09-21 | ✅ Fresh | Not edited for 1.1.1: its Quick Start comment `cyrius test  # unit (src/test.cyr) + integration (tests/kashi.tcyr)`, false on the 6.6.6 pin (Stale at 1.1.0), is true on 6.6.20 (measured: 454 + 145, two files). "Every buffer declaration is a contract: module-scope `var X[N]` = N×u64" matches the 1.1.1 sizing. Durable-only; state deferred to `state.md`. |
| `CONTRIBUTING.md` | 2026-10-29 | ✅ Fresh | 1.2.0: step 4 names the 6.7.9 pin. Before: 1.1.1: step 4 is a bare `cyrius test` (both suites from 6.6.17); CI paragraph names `cyrius lint --exit-with-count --strict-deferrals` (CI now does lint `tests/`, as this file already said); new `cyrius.lock` paragraph (tracked; regenerate with `cyrius deps` on a `lib/` holding only declared deps); code-style line says "u64-slot unit". |
| `SECURITY.md` | 2026-10-13 | ✅ Fresh | 1.1.1 buffer sizes (452 / 226 / 904 slots = 3,616 / 1,808 / 7,232 B, exactly the bytes used), why the guards matter more with no slack, and the tests that now see each guard fail; `fset` guard range `[0x20, KASHI_EXT_LAST]`; Unicode-mapping bullet (1.1.0). Audit trail unchanged (no audit file for 1.1.1). |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only **short notice** pointing at gnu.org for the full text (not the verbatim license — previous row overstated it). |
| `VERSION` | 2026-10-29 | ✅ Fresh | `1.2.0`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). Before: `1.1.1`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-10-29 | ✅ Fresh | Pin `cyrius = "6.7.9"` (1.2.0; was 6.6.20); nothing else changed. `[build] test` is still accepted on 6.7.9 (a synonym for 6.7.6's `[test] files`; no unknown-key warning). Before: Pin `cyrius = "6.6.20"` (1.1.1; was 6.6.6); otherwise unchanged since 1.0.10 — description names all three built-ins + the three formats; `[deps]` comment explains `io` / `alloc` / `vec`; `[lib]` comment states distlib's real rule (leaves listed explicitly, aggregator last). |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-10-09 | ✅ Fresh | Index covers 0001–0011; 0003 / 0004 entries carry the 0011 pointers. |
| `adr/template.md` | 2026-05-27 | ✅ Fresh | Standard ADR template. |
| `adr/0001-freestanding-font-data-core.md` | 2026-09-21 | ✅ Fresh | The load-bearing split. Accepted. Amendment 2026-08-17 (1.0.6): the library face is the `dist/kashi.cyr` bundle. **Addendum 2026-09-21 (1.0.10)**: the bundle was gitignored and unreleased 1.0.6–1.0.9 (tagged clones had no `dist/`); now tracked, released, CI-gated, and proven with a scratch `git` + `tag` consumer. Body's "two glyph byte tables" / `[1536]` is 0.1.0 as-decided. |
| `adr/0002-runtime-font-registry.md` | 2026-09-21 | ✅ Fresh | M1 registry, unified dispatch, negative-error convention. Accepted. Status line now carries the forward pointer: `KASHI_RT_FONT_BASE` 2 → 3 at ADR 0006; body records the value as decided. |
| `adr/0003-codepoint-addressing-runtime-fonts.md` | 2026-10-09 | ✅ Fresh | M2: PSF Unicode table → codepoint→glyph map; codepoint-addressed accessors + raw-index `kashi_rt_glyph_*`. Accepted. Status now points at ADR 0011 for built-in addressing (the "ASCII codepoint" built-in clause is as-decided history). |
| `adr/0004-cp437-glyph-range.md` | 2026-10-09 | ✅ Fresh | 0.4.0: freestanding range widened to full CP437. Accepted. Header: "Superseded in part by ADR 0007" (CGA high half) and "Extended by ADR 0011" (extension cells past `0xFF`). |
| `adr/0005-wide-glyph-and-ligatures.md` | 2026-05-27 | ✅ Fresh | 0.5.0: multi-byte rows (widths 1–32) via `kashi_font_stride` + `*_row_byte`; PSF Unicode sequences harvested via `kashi_font_seq_glyph`. Refines 0002/0003. Accepted. |
| `adr/0006-vga-9x16-derived-builtin.md` | 2026-05-27 | ✅ Fresh | 0.5.1: `KASHI_FONT_VGA_9X16 = 2` derived at init via the VGA col-9 rule; `KASHI_RT_FONT_BASE` 2→3. Accepted. |
| `adr/0007-cga-high-half-from-linux-pd.md` | 2026-05-28 | ✅ Fresh | 0.5.2: CGA `0x80..0xFF` filled from Linux PD `font_8x8.c`; CGA font dual-sourced. Accepted. |
| `adr/0008-bdf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.0: heapless BDF parser (`src/font_bdf.cyr`), strict-BBX, per-glyph cursor, `ENCODING -1` skipped, 4 MiB cap, `KIND=3` in the shared parsed-header struct, no new result codes. Lenient-BBX noted as the unbooked follow-on (roadmap). Accepted. |
| `adr/0009-pcf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.1: heapless TOC-driven PCF parser (`src/font_pcf.cyr`), strict-uniform metrics, both metric layouts, all four byte×bit-order combos canonicalized on load, required tables METRICS/BITMAPS/BDF_ENCODINGS, `KIND=4`, 4 MiB cap. Accepted. |
| `adr/0010-psf-u-variant.md` | 2026-10-09 | ✅ Fresh | 0.7.2: four `kashi_attach_unicode_*` APIs (binary PSF1/PSF2 table bytes + `psfgettable` text), replace semantics, runtime ids only, 256 KiB text cap; overlong-UTF-8 rejection per RFC 3629 (closes 0.6.0 audit F2); "larger table-walk caps" roadmap item dropped as a non-issue. Accepted. Uses the `cyaudit vet` name (historical). 1.1.0 note: its built-ins "ASCII codepoint scheme" sentence is superseded by ADR 0011. |
| `adr/0011-unicode-to-cp437-and-extension-glyphs.md` | 2026-10-13 | ✅ Fresh | 1.1.1: the size bullet carries a forward note (the extension cells cost 112 B of storage, not 896 B, once the buffers were right-sized). *New in 1.1.0.* `kashi_cp437_from_unicode` (ASCII; CP437 high half per the codec / CP437.TXT; U+2302 → `0x7F`, U+007F → 0; 226 mapped), the extension block `0x100..0x101` ('…', '�') reachable only by codepoint, the library face's built-in fix, the index's pre-sorted fixed-slot fill (and why: sorting at init doubled `kashi_font_init`; re-init race-benign), glyph art for all three fonts and the 8-wide ellipsis's no-spacing-column trade-off (kbd oracle, consumer advances), the additive relaxation of the 1.0.0 freeze, costs incl. init time and agnos headroom, alternatives. Accepted. |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-10-13 | ✅ Fresh | Index line for 001 rewritten for 1.1.1: `[452]` / `[226]` / `[904]` u64 slots = the bytes used, named in `_KashiFontStore`, the 8× it replaced (88,576 B per binary, file image not BSS), how a new cell grows N; `kashi_cp437_rev[128]` counts entries. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-10-29 | ✅ Fresh | 1.2.0: the array-size sentence now says expressions compile from 6.7.2 (measured on 6.7.9: `[3616 / 8]` = 452 slots) and why the core keeps the enum — its consumers compile it on 6.6.x pins. Before: Rewritten for 1.1.1 — the invariant inverted: N counts u64 slots and each buffer declares bytes ÷ 8 (was: N = bytes, 8× reserved "on purpose"). Sections: the invariant + `_KashiFontStore`; adding a cell; the size and who pays it (file image, measured −88,576 B on 6.6.20 and on agnos's 6.6.6); why the old "mirrors agnos / would under-allocate" reasoning did not hold; layout is the compiler's (6.6.13) and the two tests that look at it cannot false-fail; the reverse index (no longer "the exception"); accessors unchanged. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-10-29 | ✅ Fresh | 1.2.0: header records the 6.6.20 → 6.7.9 move; the procedure's "bumps are patches" line names the language-minor exception; new 6.6.20 → 6.7.9 precedent (the shape check, notes for the next move: `--locked`, `[test] files`, local-`path`, the core stays on 6.6.x syntax, re-run the guard mutation). Before: Header 2026-10-13 / No Open Work, 1.1.1 recorded (pin + the storage issue). Pin procedure: step 2 = bump + `cyrius deps` (re-vendors and re-locks; commit the tracked lock; no `lib sync --full`), step 3 = bare `cyrius test` + strict lint, step 4 notes distlib can move more than the stamp; the sigil/mabda `lib/` note retired. New precedent 6.6.6 → 6.6.20 (what the 6.6.7–6.6.20 changelog touched, results). The 6.6.17 note marked adopted. |
| `development/state.md` | 2026-10-29 | ✅ Fresh | 1.2.0 entry; pin 6.7.9; sizes 199,968 / 56,816 B; bench line; `deps --locked`; consumers table re-read 2026-10-29 (agnos now 6.6.20) and a "1.2.0 for consumers" paragraph (the 6.7.6 `path` change when they move their own pins). Before: 1.1.1 entry; toolchain 6.6.20 + the tracked lock; bundle 4,001 lines by distlib (4,026 by `wc -l`) with the requires block and 4-leaf sidecar; sizes split toolchain vs fix (199,952 / 56,800 B); tests 454 + 145 = 599 from one bare run; strict-lint and `deps --verify` gates; dependencies (lock tracked, sidecar 4 leaves); consumers re-read 2026-10-13 (agnos 6.6.6, aethersafha 1.1.0, agnos `KASHI_REF` 1.1.0) + "1.1.1 for consumers" (incl. agnos's measured headroom). |
| `development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md` | 2026-10-13 | ✅ Fresh | Filed by agnos against 1.1.0 (the glyph buffers reserved 8× — 88,592 B of its kernel image). **Resolved in 1.1.1**: the resolution section records what landed against each suggested step and where it differs (named slot counts — an array size takes an enum constant, not an expression; the guard test reworked rather than dropped). |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-10-29 | ✅ Fresh | 1.2.0: header; the pin is 6.7.9; `tag = "1.2.0"` (both snippets); a paragraph on 6.7.6's local-only `path`; the bundle's sidecar named as four leaves (was "eight", stale since 1.1.1). Before: 1.1.1: build/test block is one bare `cyrius test` (both suites, the 6.6.20 pin) and the paragraph says why it was two commands through 1.1.0; CI paragraph mentions the lock check; `tag = "1.1.1"`; "static tables" / "zero-filled" (was "BSS"); add-a-font step 2 sizes a buffer in u64 slots via `_KashiFontStore` and adds it to the storage tests, step 3 says the neighbour sits flush. 1.1.0's "Draw Unicode text" section holds. |
| `guides/loading-psf-fonts.md` | 2026-10-09 | ✅ Fresh | 0.7.2 content holds; the two "built-ins take an ASCII codepoint" notes now describe the 1.1.0 mapping (and the 1.0.x cell-reading bug). |
| `guides/loading-bdf-fonts.md` | 2026-09-21 | ✅ Fresh | 0.7.0 content holds; `cyrius vet` naming. |
| `guides/loading-pcf-fonts.md` | 2026-09-21 | ✅ Fresh | 0.7.1 content holds; `cyrius vet` naming. |
| `examples/glyph_sheet.cyr` + `.sha256` | 2026-10-09 | ✅ Fresh | 1.1.0: 678 lines — the 672 CP437 lines unchanged, then the 6 extension lines (three-digit cells); header documents the format and why the extension comes last (`head -672` still hashes to `b68ca4d9…`); pin `5544d63f…`. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*` + this ledger

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-05-27 | ✅ Fresh | First P(-1) pass on the freestanding-core ASCII surface at 0.1.0 (SECURITY.md / state.md label it "0.2.0 P(-1)" — same pass, opening the post-0.1.0 cycle). 3 fixed (F1 fset guard, F2 ready flag, F3 comment), F4 no-change, F5 mitigated. Historical. |
| `audit/2026-05-28-audit.md` | 2026-05-28 | ✅ Fresh | 0.6.0 P(-1): PSF parser + library face + 0.4.0–0.5.2 core extensions. F1 `glyph_count` overflow guard, F2 strict UTF-8 cp range, F3 stale comments, F4 perf note. Historical. |
| `audit/2026-05-28-audit-0.8.0.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.8.0 hardening: CVE-research-driven 17-point checklist over PSF/BDF/PCF + attach paths. F1–F9 all fixed (format-byte unknown bits, BDF int truncation, PCF duplicate TOC, F9 attach atomic-on-failure per CVE-2015-1803); +13 regression assertions. Followups (dry-run attach API; strictness relaxation if a benign undocumented bit appears) remain unbooked. Most recent audit in the trail; no source change since. |
| `benchmarks.md` | 2026-10-29 | ✅ Fresh | "Current" at 1.2.0: a two-way interleaved A/B (1.1.1 / 6.6.20 vs 1.2.0 / 6.7.9), all ten flat within ±1.6 %; 1.1.1 is "Previous", 1.1.0 "Earlier"; Trend gains the two 2026-10-29 rows. Before: "Current" at 1.1.1: a three-way interleaved A/B (1.1.0 / 6.6.6, 1.1.0 / 6.6.20, 1.1.1 / 6.6.20), all ten benches flat within ±2 %; 1.1.0's section kept as "Previous"; Trend gains the three 2026-10-13 rows; Updating says measure old and new in one session. |
| `benchmarks/history.csv` | 2026-10-29 | ✅ Fresh | Rows 0.1.0 → 1.2.0; the 2026-10-29 session is `1.1.1-on-6.6.20` / `1.2.0` (ten benches each, medians of nine interleaved runs; min / max across the runs). Before: Rows 0.1.0 → 1.1.1; the 2026-10-13 session is `1.1.0-rerun` / `1.1.0-on-6.6.20` / `1.1.1` (ten benches each, medians of nine interleaved runs; min / max across the runs). |
| `doc-health.md` | 2026-10-29 | ✅ Fresh | This ledger. Touched-rows refresh at 1.2.0 (the full-tree re-read owed at a minor is outstanding; see At a glance); last full sweep 2026-09-21 (1.0.9). Before: This ledger. Touched-rows refresh at 1.1.1; last full sweep 2026-09-21 (1.0.9). |

## Tier 7 — `docs/api/` (new since last sweep — frozen 0.9.0, sealed 1.0.0)

Public surface unchanged 1.0.0 → 1.0.10; 1.1.0 added one function and one
enum group. Cross-checked at 1.1.0: 46 `fn kashi_*` declared in `src/` = 44
public (all documented, `_file` variants and `kashi_active_font` under
shared headings) + 2 init-time `kashi_fset*` helpers the README declares
internal; the `_kashi_cp437_*` index builders are underscore-internal.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-10-13 | ✅ Fresh | 1.1.1: header notes no public symbol changed; "What's NOT public API" lists the internal `_KashiFontStore` slot sizes (not counted in the 21 groups). 44 public functions (46 declared − 2 internal `fset`), 21 enum groups, core row = 12 functions + 3 + 3 + 3 constants; the 1.1.0 internals (`_kashi_cp437_*`, `kashi_cp437_rev` / `_n`) and the extension cell numbers listed as not-API. |
| `api/core.md` | 2026-10-13 | ✅ Fresh | 1.1.1: `kashi_font_init` packs into "zero-filled buffers" (was "BSS"). Freestanding core: 3 font ids; "Cells vs codepoints"; CP437 range + `KashiExtRange` constants; `kashi_font_init` (re-init race-benign, first-init ordering, ~22 µs cost — boot work) / `is_ready`, `kashi_glyph_row` / `row_byte` / `ptr` / `encoded` (the two-input change noted), `kashi_cp437_from_unicode` (table, init note, example, perf), `kashi_font_{width,height,first,count,stride}`; stability note for the extension's growth. Matches `src/font_data.cyr`. |
| `api/loading.md` | 2026-05-28 | ✅ Fresh | `kashi_load_{psf,bdf,pcf}` + `_file` variants, `kashi_register_font`; return convention; what's not in the surface. Caps match README (256 KiB / 4 MiB / 4 MiB). |
| `api/accessors.md` | 2026-10-09 | ✅ Fresh | Unified `kashi_font_{ptr,row,row_byte}` — `cp` is Unicode for every font, built-ins via `kashi_cp437_from_unicode` (1.0.x bug + migration noted); raw `kashi_rt_glyph_*`, `kashi_rt_font_*`, `kashi_font_total`, active-font pair, `kashi_font_seq_glyph`; perf notes incl. the mapping cost. |
| `api/attach.md` | 2026-10-09 | ✅ Fresh | Four sidecar attach functions; replace semantics + atomic failure (audit 0.8.0 F9); binary vs text format choice; built-in rejection now cites the fixed `kashi_cp437_from_unicode` mapping. |
| `api/parsers.md` | 2026-05-28 | ✅ Fresh | Shared 56-byte parsed-header struct; PSF (`parse`, `uni_token`), BDF (`parse_header`, `next_glyph`), PCF (`parse_header`, `decode_glyph`, `cp_to_idx`); heapless guarantee. |
| `api/codes.md` | 2026-10-09 | ✅ Fresh | 21 enum groups: result codes (`KashiResult`, `KashiBdfRc`, `KashiPcfRc`), font ids, "Cell ranges" (`KashiGlyphRange` = CP437 only, `KashiExtRange`), PSF/BDF/PCF constants + struct field offsets, `KashiRtRec`, `KashiTabCap`. |

---

## Carry-forward / open items

Resolved since the 0.6.0 sweep:

- ~~**`docs/api/`** — defer until the library face lands.~~ ✅ created
  2026-05-28 at the 0.9.0 freeze (7 files); sealed 1.0.0.
- ~~Re-audit when M2 fonts / PSF import land.~~ ✅ `2026-05-28-audit.md`
  (0.6.0) and `2026-05-28-audit-0.8.0.md` (0.8.0, CVE-corpus-driven).

Closed in 1.0.10 (the 1.0.9 sweep's whole list):

- ~~`guides/getting-started.md` rewrite~~ ✅ · ~~1.0.6 `dist/kashi.cyr`
  propagation~~ ✅ (and the bundle is now actually published — ADR 0001
  addendum) · ~~`cyrius vet` naming~~ ✅ · ~~`architecture/001` +
  index~~ ✅ · ~~`state.md` interior~~ ✅ · ~~benchmarks rows +
  Current~~ ✅ · ~~`cyrius.cyml` prose~~ ✅ · ~~function-count drift~~ ✅
  (43 public) · ~~`docs/examples/`~~ ✅ (`glyph_sheet.cyr`) · ~~ADR
  forward pointers~~ ✅ · ~~`roadmap.md` header date~~ ✅.

Open at 1.1.0:

- **Ledger shape** — `docs/` crossed the ~30-file threshold (36 tracked with
  ADR 0011). Per-file rows were kept "while churn is toolchain-only; move to
  tier roll-ups if a 1.x feature cut reopens the tree" — 1.1.0 is that
  feature cut. The roll-up and the minor-closeout full-tree re-read are
  still owed; 1.1.0 only re-graded the rows it touched, to keep the change
  about the feature.
