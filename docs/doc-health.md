---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-09-21 (**1.0.10** — the 1.0.9 sweep's backlog
> cleared in the same day: every 🟡 row re-graded after its fix, the
> `docs/examples/` question resolved, the bundle-publishing finding
> recorded in ADR 0001).
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

**Status key**: ✅ Fresh = describes the 1.0.9 tree. 🟡 Stale = at least
one statement no longer describes the tree (the note says which). 🔵
Evergreen = standard text, re-read annually. ❓ Open question = a decision
is owed, not an edit. Historical records (ADRs, audits) are ✅ when they
accurately record the decision/pass as made; later changes are noted, not
counted against them.

---

## At a glance — 2026-09-21 inventory (1.0.10, after the sweep's fixes)

44 files in scope (was 29 at the 0.6.0 sweep; +1 for the example pair
replacing `.gitkeep`). The 1.0.9 sweep found 10 stale rows and one open
question; 1.0.10 fixed every one of them, so the ledger is clean at the
cut. Next full sweep: the next minor, or sooner if a feature cut reopens
the tree.

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 42 | Everything with a date: root files, all 12 `adr/`, `architecture/`, `development/`, the 4 guides, the example pair, the 3 audits, benchmarks, all 7 `api/`, this ledger. |
| 🟡 **Stale** | 0 | — (10 at the 1.0.9 sweep; all fixed in 1.0.10) |
| 🟠 **Read-through outstanding** | 0 | — |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — (`docs/examples/` resolved: `glyph_sheet.cyr` shipped) |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-09-21 | ✅ Fresh | Status line version-agnostic (1.x, points at `VERSION`); `cyrius vet`; the bundle named as the out-of-repo library face; userland quick start declares `modules = ["dist/kashi.cyr"]` + `alloc_init`; build block has `cyrius test` (bare), `fuzz`, `distlib`; the glyph-sheet pin; 43 public functions; `docs/examples/` listed. |
| `CHANGELOG.md` | 2026-09-21 | ✅ Fresh | `[0.1.0]`–`[1.0.10]`. 1.0.10 = published bundle + fuzz driver + glyph-sheet pin + docs currency, each with its proof. SemVer + Keep a Changelog; frozen-API note in the preamble. |
| `CLAUDE.md` | 2026-09-21 | ✅ Fresh | Durable-only. "Booked library skeleton" sentence replaced (both faces frozen; bundle tracked since 1.0.10); Quick Start has `cyrius test` (bare), `fuzz`, `distlib`; Work Loop build check includes `distlib`; glyph-fidelity principle names the sheet pin. State still deferred to `state.md`. |
| `CONTRIBUTING.md` | 2026-09-21 | ✅ Fresh | Eight-step process incl. `cyrius vet`, `distlib` + commit, the pin update; `cyrius vet` leads (parenthetical explains the `cyaudit` dispatch); glyph-fidelity rule names the pin; CI scope (`src/`, `tests/`, `docs/examples/`). |
| `SECURITY.md` | 2026-09-21 | ✅ Fresh | 1.0.0 refresh holds: current BSS sizes, all four parse surfaces, UTF-8 strictness, three-audit trail, supported = 1.x. `cyrius vet` naming. |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only **short notice** pointing at gnu.org for the full text (not the verbatim license — previous row overstated it). |
| `VERSION` | 2026-09-21 | ✅ Fresh | `1.0.10`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-09-21 | ✅ Fresh | Pin `cyrius = "6.6.6"`; description names all three built-ins + the three formats; `[deps]` comment explains `io` / `alloc` / `vec` in the present tense; `[lib]` comment records the bundle as tracked + gated since 1.0.10. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-09-21 | ✅ Fresh | Index covers 0001–0010; the 0001 entry now mentions the 2026-08-17 amendment and the 1.0.10 tracking. |
| `adr/template.md` | 2026-05-27 | ✅ Fresh | Standard ADR template. |
| `adr/0001-freestanding-font-data-core.md` | 2026-09-21 | ✅ Fresh | The load-bearing split. Accepted. Amendment 2026-08-17 (1.0.6): the library face is the `dist/kashi.cyr` bundle. **Addendum 2026-09-21 (1.0.10)**: the bundle was gitignored and unreleased 1.0.6–1.0.9 (tagged clones had no `dist/`); now tracked, released, CI-gated, and proven with a scratch `git` + `tag` consumer. Body's "two glyph byte tables" / `[1536]` is 0.1.0 as-decided. |
| `adr/0002-runtime-font-registry.md` | 2026-09-21 | ✅ Fresh | M1 registry, unified dispatch, negative-error convention. Accepted. Status line now carries the forward pointer: `KASHI_RT_FONT_BASE` 2 → 3 at ADR 0006; body records the value as decided. |
| `adr/0003-codepoint-addressing-runtime-fonts.md` | 2026-05-27 | ✅ Fresh | M2: PSF Unicode table → codepoint→glyph map; codepoint-addressed accessors + raw-index `kashi_rt_glyph_*`. Accepted. |
| `adr/0004-cp437-glyph-range.md` | 2026-09-21 | ✅ Fresh | 0.4.0: freestanding range widened to full CP437. Accepted. Header now says "Superseded in part by ADR 0007" for the CGA high half the body leaves blank. |
| `adr/0005-wide-glyph-and-ligatures.md` | 2026-05-27 | ✅ Fresh | 0.5.0: multi-byte rows (widths 1–32) via `kashi_font_stride` + `*_row_byte`; PSF Unicode sequences harvested via `kashi_font_seq_glyph`. Refines 0002/0003. Accepted. |
| `adr/0006-vga-9x16-derived-builtin.md` | 2026-05-27 | ✅ Fresh | 0.5.1: `KASHI_FONT_VGA_9X16 = 2` derived at init via the VGA col-9 rule; `KASHI_RT_FONT_BASE` 2→3. Accepted. |
| `adr/0007-cga-high-half-from-linux-pd.md` | 2026-05-28 | ✅ Fresh | 0.5.2: CGA `0x80..0xFF` filled from Linux PD `font_8x8.c`; CGA font dual-sourced. Accepted. |
| `adr/0008-bdf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.0: heapless BDF parser (`src/font_bdf.cyr`), strict-BBX, per-glyph cursor, `ENCODING -1` skipped, 4 MiB cap, `KIND=3` in the shared parsed-header struct, no new result codes. Lenient-BBX noted as the unbooked follow-on (roadmap). Accepted. |
| `adr/0009-pcf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.1: heapless TOC-driven PCF parser (`src/font_pcf.cyr`), strict-uniform metrics, both metric layouts, all four byte×bit-order combos canonicalized on load, required tables METRICS/BITMAPS/BDF_ENCODINGS, `KIND=4`, 4 MiB cap. Accepted. |
| `adr/0010-psf-u-variant.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.2: four `kashi_attach_unicode_*` APIs (binary PSF1/PSF2 table bytes + `psfgettable` text), replace semantics, runtime ids only, 256 KiB text cap; overlong-UTF-8 rejection per RFC 3629 (closes 0.6.0 audit F2); "larger table-walk caps" roadmap item dropped as a non-issue. Accepted. Uses the `cyaudit vet` name (historical). |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-09-21 | ✅ Fresh | Index line for 001 cites the real sizes (`[3584]` / 28 KiB, `[1792]`, `[7168]` stride 2). |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-09-21 | ✅ Fresh | The invariant (module-scope `var X[N]` = N×u64, byte-addressed) unchanged since 0.1.0; numbers refreshed to `3584` / `1792` / `7168`, 224 glyphs, the stride-2 `kashi_font9_16`; "Affects" lists the derivation loop, `kashi_glyph_row` / `_row_byte`; accessor consequence is stride-aware and names `test_vga_9x16_structural`. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-09-21 | ✅ Fresh | Header 2026-09-21 / No Open Work with the 1.0.10 closure recorded; out-of-scope lists; reopen triggers + two new ones (sheet pin moves under a bump; the bundle's first real consumer); durable seven-step "Moving the cyrius pin" procedure with the 6.6.6 analysis as precedent. |
| `development/state.md` | 2026-09-21 | ✅ Fresh | 1.0.10 entry; 1.0.4–1.0.7 line; interior reconciled: `cyrius vet`, the bundle + example in What's implemented, real sizes, 450 assertions (393 + 57) + the fuzz driver, Cleanliness incl. `distlib`, Dependencies present-tense, Consumers = the six with their kashi / cyrius pins. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-09-21 | ✅ Fresh | Rewritten for the 1.0 surface (see 1.0.10): prerequisites, full gate set incl. the sheet pin, layout table incl. `dist/` and the example, pick-a-face with the +50 % measurement, core consumption + render loop + stride note, library-face consumption via the tracked bundle, 8-step add-a-font checklist (RT base, `fset` guard, tcyr group, fuzz list, example + re-pin, ≥64 KB caveat, `distlib`), add-a-format, pin-bump pointer. |
| `guides/loading-psf-fonts.md` | 2026-09-21 | ✅ Fresh | 0.7.2 content holds; the `# always 8 in M1` comment on `kashi_rt_font_width` replaced with the 0.5.0 width / stride rule. |
| `guides/loading-bdf-fonts.md` | 2026-09-21 | ✅ Fresh | 0.7.0 content holds; `cyrius vet` naming. |
| `guides/loading-pcf-fonts.md` | 2026-09-21 | ✅ Fresh | 0.7.1 content holds; `cyrius vet` naming. |
| `examples/glyph_sheet.cyr` + `.sha256` | 2026-09-21 | ✅ Fresh | *New in 1.0.10; replaces the `.gitkeep` open since 0.1.0.* Freestanding-core example dumping all 672 built-in glyphs as hex; the sha256 pin CI compares against. Header documents the load-bearing format and the re-pin rule. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*` + this ledger

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-05-27 | ✅ Fresh | First P(-1) pass on the freestanding-core ASCII surface at 0.1.0 (SECURITY.md / state.md label it "0.2.0 P(-1)" — same pass, opening the post-0.1.0 cycle). 3 fixed (F1 fset guard, F2 ready flag, F3 comment), F4 no-change, F5 mitigated. Historical. |
| `audit/2026-05-28-audit.md` | 2026-05-28 | ✅ Fresh | 0.6.0 P(-1): PSF parser + library face + 0.4.0–0.5.2 core extensions. F1 `glyph_count` overflow guard, F2 strict UTF-8 cp range, F3 stale comments, F4 perf note. Historical. |
| `audit/2026-05-28-audit-0.8.0.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.8.0 hardening: CVE-research-driven 17-point checklist over PSF/BDF/PCF + attach paths. F1–F9 all fixed (format-byte unknown bits, BDF int truncation, PCF duplicate TOC, F9 attach atomic-on-failure per CVE-2015-1803); +13 regression assertions. Followups (dry-run attach API; strictness relaxation if a benign undocumented bit appears) remain unbooked. Most recent audit in the trail; no source change since. |
| `benchmarks.md` | 2026-09-21 | ✅ Fresh | "Current" at 1.0.10 (cyrius 6.6.6); trend table through 1.0.10; new structural-shift note (1.0.0 → 1.0.9 is toolchain codegen, sheet pin as proof); Updating rule says a toolchain bump is a release to record. |
| `benchmarks/history.csv` | 2026-09-21 | ✅ Fresh | Rows 0.1.0 → 1.0.10 (1.0.9 and 1.0.10 appended; 1.0.1–1.0.8 not measured, noted in the doc). |
| `doc-health.md` | 2026-09-21 | ✅ Fresh | This ledger. Previous sweep 2026-09-21 (1.0.9 sweep, first since 0.6.0); rows re-graded at 1.0.10. |

## Tier 7 — `docs/api/` (new since last sweep — frozen 0.9.0, sealed 1.0.0)

Public surface unchanged 1.0.0 → 1.0.9. This sweep cross-checked every
`fn kashi_*` in `src/` against the reference: 45 declared = 43 public (all
documented, `_file` variants and `kashi_active_font` under shared
headings) + 2 init-time `kashi_fset*` helpers the README declares internal.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-09-21 | ✅ Fresh | 43 public functions (45 declared − 2 internal `fset`), the per-file table sums to it (`accessors.md` = 14 in 13 entries); two-faces section names the `dist/kashi.cyr` bundle. |
| `api/core.md` | 2026-05-28 | ✅ Fresh | Freestanding core: 3 font ids, 3 range constants, `kashi_font_init` / `is_ready`, `kashi_glyph_row` / `row_byte` / `ptr` / `encoded`, `kashi_font_{width,height,first,count,stride}`. Matches `src/font_data.cyr`. |
| `api/loading.md` | 2026-05-28 | ✅ Fresh | `kashi_load_{psf,bdf,pcf}` + `_file` variants, `kashi_register_font`; return convention; what's not in the surface. Caps match README (256 KiB / 4 MiB / 4 MiB). |
| `api/accessors.md` | 2026-05-28 | ✅ Fresh | Unified `kashi_font_{ptr,row,row_byte}`, raw `kashi_rt_glyph_*`, `kashi_rt_font_*` metadata, `kashi_font_total`, active-font pair, `kashi_font_seq_glyph`; perf notes. |
| `api/attach.md` | 2026-05-28 | ✅ Fresh | Four sidecar attach functions; replace semantics + atomic failure (audit 0.8.0 F9); binary vs text format choice. |
| `api/parsers.md` | 2026-05-28 | ✅ Fresh | Shared 56-byte parsed-header struct; PSF (`parse`, `uni_token`), BDF (`parse_header`, `next_glyph`), PCF (`parse_header`, `decode_glyph`, `cp_to_idx`); heapless guarantee. |
| `api/codes.md` | 2026-05-28 | ✅ Fresh | 20 enum groups: result codes (`KashiResult`, `KashiBdfRc`, `KashiPcfRc`), font ids, glyph range, PSF/BDF/PCF constants + struct field offsets, `KashiRtRec`, `KashiTabCap`. |

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

Open at 1.0.10:

- **Ledger shape** — `docs/` crossed the ~30-file threshold (34). Per-file
  rows kept while churn is toolchain-only; move to tier roll-ups if a 1.x
  feature cut reopens the tree.
