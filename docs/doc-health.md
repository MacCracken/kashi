---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-09-21 (**1.0.9** — full-tree sweep; first since
> the 0.6.0 cut. 0.7.0–1.0.0 landed BDF/PCF/sidecar import, the
> `docs/api/` freeze, and a third audit; 1.0.1–1.0.9 were toolchain bumps,
> plus 1.0.6's `[lib]` + `dist/kashi.cyr` publish fix).
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

## At a glance — 2026-09-21 inventory (1.0.9 sweep)

43 files in scope (was 29 at the 0.6.0 sweep). The 0.7.0–1.0.0 docs were
written fresh with the code; the stale set is the 0.1.0-era docs nobody
re-read after the surface finished growing, plus the volatile files whose
interiors lag their bumped headers.

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 30 | CHANGELOG, VERSION, SECURITY; all 12 `adr/` files (0008–0010 new since last sweep); `roadmap.md`; the 3 loading guides; the 3 audits (0.8.0 new); all 7 `api/` files (new — frozen surface, all 43 public symbols verified documented); this ledger. |
| 🟡 **Stale** | 10 | `README.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `cyrius.cyml`, `architecture/README.md` + `001`, `development/state.md` (interior), `guides/getting-started.md`, `benchmarks.md`, `benchmarks/history.csv`. Two root causes: **(a)** 1.0.1's `cyaudit vet` → `cyrius vet` retirement and 1.0.6's `dist/kashi.cyr` bundle never propagated past CHANGELOG / ADR 0001; **(b)** 0.1.0-era numbers (buffer sizes, "booked" language) were never refreshed. |
| 🟠 **Read-through outstanding** | 0 | — |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 1 | `examples/.gitkeep` — still empty at 1.0.9; backfill or drop the directory. |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-06-13 | 🟡 Stale | Rewritten at 1.0.0 (two faces, 3 built-ins, 3 runtime formats, sidecar attach, `docs/api/` links) and still structurally right. Lags: **Status** line says `1.0.0` (tree is 1.0.9; API unchanged); parser paragraph still says `cyaudit vet` (Build & test block says `cyrius vet`); userland quick start is `include "src/lib.cyr"` with no mention of the 1.0.6 `[lib]` / `dist/kashi.cyr` bundle that made the library face consumable downstream; says "45 functions" where `api/README.md` says 44 (actual: 43 public + 2 internal `fset` helpers). |
| `CHANGELOG.md` | 2026-09-21 | ✅ Fresh | `[0.1.0]`–`[1.0.9]`. 1.0.0 = stable seal; 1.0.1–1.0.9 = toolchain pins 6.0.3 → 6.6.6 (1.0.6 also adds `[lib]` + `dist/kashi.cyr`; 1.0.9 carries the 6.6.4→6.6.6 bench comparison + byte-identical glyph-sheet check). SemVer + Keep a Changelog; frozen-API note in the preamble. |
| `CLAUDE.md` | 2026-05-27 | 🟡 Stale | Durable-only by design and mostly holds. Lags: "0.1.0 baseline shipped … a booked library skeleton; the library face is built out along the roadmap" — roadmap closed at 1.0.0, face is built and frozen; Quick Start / Work Loop have no `cyrius distlib` → `dist/kashi.cyr` regeneration step (1.0.6; the roadmap's pin-bump recipe requires it). State correctly deferred to `state.md`. |
| `CONTRIBUTING.md` | 2026-06-13 | 🟡 Stale | Two faces, parser modules named, accessor-bounds + glyph-fidelity rules, current return-code convention (1.0.0). Lags: still leads with `cyaudit vet` (parenthetical admits `cyrius vet` works since 6.2.2); no "regenerate `dist/kashi.cyr` after touching `[lib]` modules" step (1.0.6). |
| `SECURITY.md` | 2026-05-28 | ✅ Fresh | 1.0.0 refresh holds at 1.0.9: current BSS sizes (`3584` / `1792` / `7168`), all four parse surfaces, UTF-8 strictness (F2 0.6.0, F3 0.7.2), three-audit trail, supported = 1.x. Only residue: `cyaudit vet` name (not wrong — `cyrius vet` dispatches to it). |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only **short notice** pointing at gnu.org for the full text (not the verbatim license — previous row overstated it). |
| `VERSION` | 2026-09-21 | ✅ Fresh | `1.0.9`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-09-21 | 🟡 Stale | Pin `cyrius = "6.6.6"` current; `[lib]` fold (1.0.6) present and well-commented; deps list matches `state.md`. Lags: `description` still says "(VGA 8x16 + CGA 8x8)" — three built-ins since 0.5.1; `[deps]` comment says PSF import "will add `io` / `fs` here when those modules land" — landed 0.2.0. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-05-28 | ✅ Fresh | Index covers 0001–0010 with one-line summaries; conventions + ADR/note/guide table. The 0001 entry predates the 2026-08-17 amendment (no mention of `dist/kashi.cyr`). |
| `adr/template.md` | 2026-05-27 | ✅ Fresh | Standard ADR template. |
| `adr/0001-freestanding-font-data-core.md` | 2026-08-17 | ✅ Fresh | The load-bearing split. Accepted. **Amendment 2026-08-17 (1.0.6)**: the library face was unconsumable when vendored (`include "src/font_data.cyr"` is repo-relative); resolved by `[lib]` + `dist/kashi.cyr`; desktop-stack consumers measured (+50% for the full face) and correctly keep the core. Body still says "two glyph byte tables" / `kashi_font16[1536]` — 0.1.0 as-decided; index + 0006 carry the third built-in. |
| `adr/0002-runtime-font-registry.md` | 2026-05-27 | ✅ Fresh | M1: runtime registry, unified dispatch, width≤8, negative-error convention. Accepted (addressing refined by 0003). Records `KASHI_RT_FONT_BASE = 2`; bumped to 3 by 0006 — no forward pointer in the file, the index has it. |
| `adr/0003-codepoint-addressing-runtime-fonts.md` | 2026-05-27 | ✅ Fresh | M2: PSF Unicode table → codepoint→glyph map; codepoint-addressed accessors + raw-index `kashi_rt_glyph_*`. Accepted. |
| `adr/0004-cp437-glyph-range.md` | 2026-05-27 | ✅ Fresh | 0.4.0: widen freestanding range to `0x20..0xFF` (full CP437) in VGA 8×16 from Linux PD source; CGA high half blank *as decided then* — filled by 0007 (no forward pointer in the file, the index has it). Accepted. |
| `adr/0005-wide-glyph-and-ligatures.md` | 2026-05-27 | ✅ Fresh | 0.5.0: multi-byte rows (widths 1–32) via `kashi_font_stride` + `*_row_byte`; PSF Unicode sequences harvested via `kashi_font_seq_glyph`. Refines 0002/0003. Accepted. |
| `adr/0006-vga-9x16-derived-builtin.md` | 2026-05-27 | ✅ Fresh | 0.5.1: `KASHI_FONT_VGA_9X16 = 2` derived at init via the VGA col-9 rule; `KASHI_RT_FONT_BASE` 2→3. Accepted. |
| `adr/0007-cga-high-half-from-linux-pd.md` | 2026-05-28 | ✅ Fresh | 0.5.2: CGA `0x80..0xFF` filled from Linux PD `font_8x8.c`; CGA font dual-sourced. Accepted. |
| `adr/0008-bdf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.0: heapless BDF parser (`src/font_bdf.cyr`), strict-BBX, per-glyph cursor, `ENCODING -1` skipped, 4 MiB cap, `KIND=3` in the shared parsed-header struct, no new result codes. Lenient-BBX noted as the unbooked follow-on (roadmap). Accepted. |
| `adr/0009-pcf-import.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.1: heapless TOC-driven PCF parser (`src/font_pcf.cyr`), strict-uniform metrics, both metric layouts, all four byte×bit-order combos canonicalized on load, required tables METRICS/BITMAPS/BDF_ENCODINGS, `KIND=4`, 4 MiB cap. Accepted. |
| `adr/0010-psf-u-variant.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.2: four `kashi_attach_unicode_*` APIs (binary PSF1/PSF2 table bytes + `psfgettable` text), replace semantics, runtime ids only, 256 KiB text cap; overlong-UTF-8 rejection per RFC 3629 (closes 0.6.0 audit F2); "larger table-walk caps" roadmap item dropped as a non-issue. Accepted. Uses the `cyaudit vet` name (historical). |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-05-27 | 🟡 Stale | Index line for 001 cites `var kashi_font16[1536]` / 12 KiB — the 0.1.0 size; the buffer is `[3584]` since 0.4.0. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-05-27 | 🟡 Stale | The invariant (module-scope `var X[N]` = N×u64, byte-addressed via store8/load8, mirrors agnos `fb_font`) is **still true and still load-bearing**. The numbers are 0.1.0: `kashi_font16[1536]` / `kashi_font8[768]` / 96 glyphs; actual is `3584` / `1792` / 224 glyphs, plus a third buffer `kashi_font9_16[7168]` (2-byte stride, 0.5.1) the note doesn't mention. "Affects" list omits `kashi_font9_16` and `kashi_glyph_row_byte`. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-09-21 | ✅ Fresh | Closed at 1.0.0: "No Open Work"; committed vs unbooked out-of-scope lists; what would reopen it. Appended 2026-09-21: the *Moving the cyrius pin to 6.6.6* recipe + 1.0.9 result (byte-identical glyph sheet, dist regen, kernel-side pin caveat). Header still reads "Last Updated: 2026-05-28" — not bumped when the 6.6.6 section landed. |
| `development/state.md` | 2026-09-21 | 🟡 Stale | Header + Version section current at 1.0.9 (pin 6.6.6, 393 + 49 assertions, library 256,592 B / demo 137,936 B). Interior lags: **Build & size** still "~84 KB DCE" (contradicts the 1.0.9 entry above it); **Cleanliness** uses `cyaudit vet`; **Dependencies** ends "No manifest change for 0.7.1"; **Consumers** lists only agnos 1.38.0 — 1.0.6 names dhancha/crab/puka/aethersafha taking the core, 1.0.8 notes agnos 1.57.4; **What's implemented** has no `dist/kashi.cyr` / `[lib]` line. Version-history block (1.0.4–1.0.7) skipped. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-05-27 | 🟡 Stale | **Unrevised since the 0.1.0 scaffold** — the widest gap in the tree. Says `src/lib.cyr` "books the runtime surface" and "Consumers who can link the stdlib include this" (superseded by 1.0.6 `dist/kashi.cyr`); "Runtime font loading … see the roadmap M1" (M1 shipped 0.2.0; three loading guides exist); Layout omits the parser modules and `docs/api/`; "Adding a font" checklist omits `kashi_font_stride` (0.5.0) and the ≥64 KB-literal toolchain caveat the roadmap's 6.6.6 section asks for. Build block itself is still correct. |
| `guides/loading-psf-fonts.md` | 2026-05-28 | ✅ Fresh | 0.7.2: load, font ids (0/1/2 built-in, ≥3 runtime), codepoint vs raw-index, wide-glyph rows, ligature lookup, sidecar tables (binary + text), limits incl. UTF-8 strictness. One stale code comment: `# always 8 in M1` on `kashi_rt_font_width` (wide fonts since 0.5.0). |
| `guides/loading-bdf-fonts.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.0: load, read, accepted subset (strict BBX, `ENCODING -1`, 4 MiB cap), minimal example BDF, posture, PSF-vs-BDF. `cyaudit vet` name only. |
| `guides/loading-pcf-fonts.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.7.1: load, read, accepted subset (strict metrics, both layouts, four byte×bit combos, geometry + 4 MiB caps), posture, PSF/BDF/PCF choice. `cyaudit vet` name only. |
| `examples/.gitkeep` | 2026-05-27 | ❓ Open question | Placeholder for the whole 0.1.0 → 1.0.9 arc. Every public symbol now carries a code snippet in `docs/api/`, and the demo lives in `src/main.cyr`. Decide: backfill runnable `docs/examples/*.cyr` (freestanding render loop; userland PSF load + sidecar attach) or drop the directory and the CLAUDE.md pointer. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*` + this ledger

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-05-27 | ✅ Fresh | First P(-1) pass on the freestanding-core ASCII surface at 0.1.0 (SECURITY.md / state.md label it "0.2.0 P(-1)" — same pass, opening the post-0.1.0 cycle). 3 fixed (F1 fset guard, F2 ready flag, F3 comment), F4 no-change, F5 mitigated. Historical. |
| `audit/2026-05-28-audit.md` | 2026-05-28 | ✅ Fresh | 0.6.0 P(-1): PSF parser + library face + 0.4.0–0.5.2 core extensions. F1 `glyph_count` overflow guard, F2 strict UTF-8 cp range, F3 stale comments, F4 perf note. Historical. |
| `audit/2026-05-28-audit-0.8.0.md` | 2026-05-28 | ✅ Fresh | *New since last sweep.* 0.8.0 hardening: CVE-research-driven 17-point checklist over PSF/BDF/PCF + attach paths. F1–F9 all fixed (format-byte unknown bits, BDF int truncation, PCF duplicate TOC, F9 attach atomic-on-failure per CVE-2015-1803); +13 regression assertions. Followups (dry-run attach API; strictness relaxation if a benign undocumented bit appears) remain unbooked. Most recent audit in the trail; no source change since. |
| `benchmarks.md` | 2026-05-28 | 🟡 Stale | Methodology, "Current — 1.0.0" table, 0.x trend, structural-shift notes, hot-path pattern — all accurate for the frozen hot path. Lags its own "Updating" rule: 1.0.9 recorded a 6.6.4→6.6.6 comparison in CHANGELOG (`scan_vga_8x16` 57.8→56.2 µs, `font_row_runtime_cp` 57→58 ns, others flat) that is not reflected here. |
| `benchmarks/history.csv` | 2026-05-28 | 🟡 Stale | Per-`(version,benchmark)` rows 0.1.0 → 1.0.0. No 1.0.x row; the 1.0.9 numbers exist only in CHANGELOG. |
| `doc-health.md` | 2026-09-21 | ✅ Fresh | This ledger. Previous sweep 2026-05-28 (0.6.0). |

## Tier 7 — `docs/api/` (new since last sweep — frozen 0.9.0, sealed 1.0.0)

Public surface unchanged 1.0.0 → 1.0.9. This sweep cross-checked every
`fn kashi_*` in `src/` against the reference: 45 declared = 43 public (all
documented, `_file` variants and `kashi_active_font` under shared
headings) + 2 init-time `kashi_fset*` helpers the README declares internal.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-05-28 | ✅ Fresh | Audience guide, two faces, per-file symbol table, stability promise (1.x additive-only; `_kashi_` internal), return-value conventions, cross-refs. Says "44 functions" vs root README's 45 — neither is the 43-public figure. Two-faces section names `src/lib.cyr` as the library face with no mention of the vendorable `dist/kashi.cyr` (1.0.6). |
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

Open at 1.0.9, roughly by value:

- **`guides/getting-started.md`** — rewrite for the 1.0 surface (parser
  modules, `docs/api/`, `dist/kashi.cyr` consumption, three loading
  guides, `kashi_font_stride` + the ≥64 KB-literal caveat in "Adding a
  font"). The only doc still describing 0.1.0 as the present.
- **1.0.6 `dist/kashi.cyr` propagation** — the vendorable library-face
  bundle is documented only in CHANGELOG, the `cyrius.cyml` comment, and
  the ADR 0001 amendment. Surface it in README (userland quick start),
  CONTRIBUTING (regenerate after touching `[lib]` modules), CLAUDE.md
  Quick Start / Work Loop, `api/README.md` two-faces, `state.md`.
- **1.0.1 `cyrius vet` naming** — living docs still say `cyaudit vet`:
  README (one paragraph), CONTRIBUTING, SECURITY, `state.md` Cleanliness,
  BDF/PCF guides. Historical docs (ADR 0010, 0.8.0 audit) stay as written.
  `cyrius vet` dispatches to `cyaudit vet`, so nothing is wrong, only
  non-canonical.
- **`architecture/001` + index** — refresh sizes to `3584` / `1792` /
  `7168`, 224 glyphs; add `kashi_font9_16` (2-byte stride) to the
  invariant's "Affects" list.
- **`state.md` interior** — reconcile Build & size (~84 KB vs 256,592 B /
  137,936 B), Dependencies, Consumers (desktop stack + agnos 1.57.4) with
  the 1.0.9 header.
- **Benchmarks** — append the 1.0.9 (6.6.6) row to `history.csv` from the
  CHANGELOG numbers and refresh `benchmarks.md` "Current"; the doc's own
  rule is one row per release.
- **`cyrius.cyml` prose** — `description` names two built-ins (three since
  0.5.1); `[deps]` comment still future-tenses `io` / `fs`.
- **Function-count drift** — README 45 / `api/README.md` 44 / actual 43
  public. Pick one number and its definition.
- **`docs/examples/`** — backfill or drop (❓ above).
- **ADR forward pointers** — 0002 (`KASHI_RT_FONT_BASE = 2`) and 0004
  (CGA high half blank) carry no "see 0006 / 0007" line; the index covers
  it. Nice-to-have.
- **`roadmap.md` header date** — "Last Updated: 2026-05-28" not bumped
  when the 6.6.6 section landed 2026-09-21.
- **Ledger shape** — `docs/` crossed the ~30-file threshold (34). Per-file
  rows kept while churn is toolchain-only; move to tier roll-ups if a 1.x
  feature cut reopens the tree.
