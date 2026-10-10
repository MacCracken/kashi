---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-10-09 (**1.1.0** — every doc the Unicode → CP437
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
`docs/` is now 34 files — at the ~30-file threshold where the agnosticos
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
| ✅ **Fresh** | 42 | Everything with a date except `CLAUDE.md`: the other root files, all 13 `adr/`, `architecture/`, `development/`, the 4 guides, the example pair, the 3 audits, benchmarks, all 7 `api/`, this ledger. |
| 🟡 **Stale** | 1 | `CLAUDE.md` — one false Quick Start comment about bare `cyrius test`, left for the operator (row below). (10 at the 1.0.9 sweep; all fixed in 1.0.10.) |
| 🟠 **Read-through outstanding** | 0 | — |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — (`docs/examples/` resolved: `glyph_sheet.cyr` shipped) |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-10-09 | ✅ Fresh | 1.1.0: status line names the additive minor; built-in section says CP437 cells + two extension cells, with a "Unicode text (1.1.0)" table (`kashi_cp437_from_unicode`, `KASHI_EXT_*`) and the library-face fix; freestanding quick start shows the Unicode path; `tag = "1.1.0"`; demo line; Build & test block names both suites (a bare `cyrius test` is integration-only on 6.6.6; `cyrius test src/test.cyr` for the unit suite); 44 public functions / 21 enum groups / 11 ADRs. |
| `CHANGELOG.md` | 2026-10-09 | ✅ Fresh | `[0.1.0]`–`[1.1.0]`. 1.1.0 = Unicode → CP437 map, extension glyphs, library-face built-in fix; byte-identity proof, sizes (incl. agnos image headroom), same-session bench A/B with `font_init` (+1.2 µs; the index is pre-sorted, no sort at init), two mutation rounds, the explicit 0x7F behaviour note, the CI unit-suite step, the deliberate re-pin. The 1.0.10-era `[Unreleased]` `cyrius.cyml` comment item folded into 1.1.0. |
| `CLAUDE.md` | 2026-09-21 | 🟡 Stale | ⚠ Quick Start's comment `cyrius test  # unit (src/test.cyr) + integration (tests/kashi.tcyr)` is false on the 6.6.6 pin: a bare run executes `tests/*.tcyr` only (measured 2026-10-09; 6.6.17 runs both). Left for the operator — CLAUDE.md is the operator's instruction file; the fix is the two-line form README / getting-started now use. Otherwise: durable-only. "Booked library skeleton" sentence replaced (both faces frozen; bundle tracked since 1.0.10); Quick Start has `cyrius test` (bare), `fuzz`, `distlib`; Work Loop build check includes `distlib`; glyph-fidelity principle names the sheet pin. State still deferred to `state.md`. |
| `CONTRIBUTING.md` | 2026-10-09 | ✅ Fresh | Step 4 names both suites (`cyrius test src/test.cyr` + bare `cyrius test`; a bare run skips the unit suite below 6.6.17). Eight-step process incl. `cyrius vet`, `distlib` + commit, the pin update; `cyrius vet` leads (parenthetical explains the `cyaudit` dispatch); glyph-fidelity rule names the pin; CI scope (`src/`, `tests/`, `docs/examples/`). |
| `SECURITY.md` | 2026-10-09 | ✅ Fresh | 1.1.0 buffer sizes (`[3616]` / `[1808]` / `[7232]`), `fset` guard range `[0x20, KASHI_EXT_LAST]`, and a Unicode-mapping bullet (total over i64, only 0 or an encoded cell, bounded index, fuzzed + swept). Audit trail unchanged (no new audit file for 1.1.0). |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only **short notice** pointing at gnu.org for the full text (not the verbatim license — previous row overstated it). |
| `VERSION` | 2026-10-09 | ✅ Fresh | `1.1.0`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-09-21 | ✅ Fresh | Pin `cyrius = "6.6.6"`; description names all three built-ins + the three formats; `[deps]` comment explains `io` / `alloc` / `vec` in the present tense; `[lib]` comment records the bundle as tracked + gated since 1.0.10. |

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
| `adr/0011-unicode-to-cp437-and-extension-glyphs.md` | 2026-10-09 | ✅ Fresh | *New in 1.1.0.* `kashi_cp437_from_unicode` (ASCII; CP437 high half per the codec / CP437.TXT; U+2302 → `0x7F`, U+007F → 0; 226 mapped), the extension block `0x100..0x101` ('…', '�') reachable only by codepoint, the library face's built-in fix, the index's pre-sorted fixed-slot fill (and why: sorting at init doubled `kashi_font_init`; re-init race-benign), glyph art for all three fonts and the 8-wide ellipsis's no-spacing-column trade-off (kbd oracle, consumer advances), the additive relaxation of the 1.0.0 freeze, costs incl. init time and agnos headroom, alternatives. Accepted. |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-10-09 | ✅ Fresh | Index line for 001 cites the 1.1.0 sizes (`[3616]` / 28.25 KiB, `[1808]`, `[7232]` stride 2; 224 CP437 + 2 extension glyphs) and the `kashi_cp437_rev[128]` exception. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-10-09 | ✅ Fresh | The invariant (module-scope `var X[N]` = N×u64, byte-addressed) unchanged since 0.1.0; numbers refreshed to `3616` / `1808` / `7232` (226 glyphs: CP437 + the extension, contiguous); the reservation is in every binary; new section: the deliberate exception `kashi_cp437_rev[128]` (N = entries, one u64 slot each). |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-10-09 | ✅ Fresh | Header 2026-10-09 / No Open Work, with 1.1.0 recorded as the first consumer-driven cut since 1.0; two unbooked additive items (more extension glyphs, mapping aliases) and a reopen trigger (a glyph CP437 lacks); the 6.6.17 bare-`cyrius test` note checked on 6.6.17 (448 + 145) and rewritten for CI's explicit unit-suite step (drop it at that pin move); the pin procedure's gate set names both test commands. |
| `development/state.md` | 2026-10-09 | ✅ Fresh | 1.1.0 entry; What's implemented (cells + extension, `kashi_cp437_from_unicode`, library-face built-in mapping, bundle 3,985 lines by distlib's count / 4,004 by `wc -l`, demo, sheet pin `5544d63f…`); sizes 262,656 / 144,072 B with hashes (`6ce819b9…` / `74e2ceef…`) and the core's +6,064 B; `kashi_font_init` +1.2 µs; 593 assertions (448 + 145) + the fuzz Unicode contract; the 6.6.6 bare-`cyrius test` caveat and CI's unit step; consumers' 1.1.0 note (per-draw init cost for dhancha, agnos headroom, `KASHI_REF`). |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-10-09 | ✅ Fresh | 1.1.0: "Draw Unicode text" section (cells vs codepoints, `kashi_cp437_from_unicode`, '…' / '�'), layout/demo/example rows, `tag = "1.1.0"`, add-a-font checklist sizes (`[3616]`…) and the `[FIRST, EXT_LAST]` guard + extension glyphs per font, eleven ADRs. Build/test block and its paragraph corrected: on 6.6.6 a bare `cyrius test` runs only `tests/*.tcyr` (~145), the unit suite (~450) needs `cyrius test src/test.cyr`; demo comment names '…' / '�'. |
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
| `benchmarks.md` | 2026-10-09 | ✅ Fresh | "Current" at 1.1.0 with the same-session 1.0.10 re-run beside it (quiet host; a loaded second session agreed); four new benches incl. `font_init`; what moved and why (`font_row_builtin` +3 ns = the mapping; `font_init` +1.2 µs = extension glyphs + index fill, attributed; `scan_vga_8x16` +3 % = code placement, shown); trend rows; render-pattern note. |
| `benchmarks/history.csv` | 2026-10-09 | ✅ Fresh | Rows 0.1.0 → 1.1.0; `1.0.10-rerun` rows (incl. `font_init`) are the same-session baseline for 1.1.0 (documented in the doc). |
| `doc-health.md` | 2026-10-09 | ✅ Fresh | This ledger. Touched-rows refresh at 1.1.0; last full sweep 2026-09-21 (1.0.9). |

## Tier 7 — `docs/api/` (new since last sweep — frozen 0.9.0, sealed 1.0.0)

Public surface unchanged 1.0.0 → 1.0.10; 1.1.0 added one function and one
enum group. Cross-checked at 1.1.0: 46 `fn kashi_*` declared in `src/` = 44
public (all documented, `_file` variants and `kashi_active_font` under
shared headings) + 2 init-time `kashi_fset*` helpers the README declares
internal; the `_kashi_cp437_*` index builders are underscore-internal.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-10-09 | ✅ Fresh | 44 public functions (46 declared − 2 internal `fset`), 21 enum groups, core row = 12 functions + 3 + 3 + 3 constants; the 1.1.0 internals (`_kashi_cp437_*`, `kashi_cp437_rev` / `_n`) and the extension cell numbers listed as not-API. |
| `api/core.md` | 2026-10-09 | ✅ Fresh | Freestanding core: 3 font ids; "Cells vs codepoints"; CP437 range + `KashiExtRange` constants; `kashi_font_init` (re-init race-benign, first-init ordering, ~22 µs cost — boot work) / `is_ready`, `kashi_glyph_row` / `row_byte` / `ptr` / `encoded` (the two-input change noted), `kashi_cp437_from_unicode` (table, init note, example, perf), `kashi_font_{width,height,first,count,stride}`; stability note for the extension's growth. Matches `src/font_data.cyr`. |
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
