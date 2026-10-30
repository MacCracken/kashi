---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-10-30 (**1.3.0** — the audit-driven hardening
> release: ADR 0012 and the 2026-10-30 audit added; the API reference,
> guides, SECURITY, state, roadmap and benchmarks re-graded for it).
> Previous: 2026-10-29 (roadmap cleanup after 1.2.2 — the roadmap
> rewritten as 1.3.x – 1.x.x milestones; state, SECURITY and benchmarks
> point at the bookings).
> Previous: 2026-10-29 (**1.2.2** — touched rows for the PCF fix:
> the issue resolved, `api/parsers.md`'s caveat retired, state, roadmap,
> benchmarks, tags; the `src/` comment items closed).
> Previous: 2026-10-29 (**1.2.1** — the full-tree re-read owed at the
> 1.2 minor: every file in scope re-checked against the tree, about 45
> corrections, one code bug found and filed).
> Previous: 2026-10-29 (**1.2.0** — touched-rows refresh for the pin move to
> cyrius 6.7.9).
> Previous: 2026-10-13 (**1.1.1** — the pin move to cyrius 6.6.20 and
> the glyph-storage fix).
> Previous: 2026-10-09 (**1.1.0** — the Unicode → CP437 map and the
> extension glyphs, touched rows).
> Previous: 2026-09-21 (1.0.10 — the 1.0.9 sweep's backlog cleared; the
> last full sweep before this one).
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
`docs/` is 38 files (2026-10-29) — past the ~30-file threshold where the
agnosticos pattern switches to tier roll-ups. Per-file rows are kept,
because the API is frozen and churn since 1.1.0 has been toolchain-only;
the roll-up stays an open item.

**Status key**: ✅ Fresh = describes the 1.2.1 tree (every row was re-read
on 2026-10-29; "Last touched" is the last edit). 🟡 Stale = at least one
statement no longer describes the tree (the note says which). 🔵
Evergreen = standard text, re-read annually. ❓ Open question = a decision
is owed, not an edit. Historical records (ADRs, audits) are ✅ when they
accurately record the decision or pass as made and their status lines point
at what changed later.

---

## At a glance — 2026-10-29 inventory (1.2.1, full sweep)

47 files in scope (46 rows: the glyph-sheet pair shares one), +1 since 1.2.0
(the PCF issue). Three read-only reviewers re-checked every file against
`src/`, the tests, the manifest and the consumer repos, and each finding
was verified before it was fixed. **What changed:**
- 37 files edited or added;
- two long-standing factual errors corrected: ADR 0008's BDF API and ADR
  0009's ctx size;
- one code bug found and filed rather than fixed: `kashi_pcf_cp_to_idx`
  above U+FFFF.

Left open on purpose:
- `CLAUDE.md`, which is operator-owned;
- six stale comments in `src/`, since a comment edit changes `src/` and
  the bundle and this release is no-source-change (see below).

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 43 | Everything in scope but `CLAUDE.md` and the two evergreen files. |
| 🟡 **Stale** | 1 | `CLAUDE.md` — five statements (its row); operator-owned. |
| 🟠 **Read-through outstanding** | 0 | — (the 1.2.0 debt is paid). |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-10-30 | ✅ Fresh | 1.3.0: status line names the hardening pass; `tag = "1.3.0"`; the bundle's five sidecar leaves (`syscalls` joined). |
| `CHANGELOG.md` | 2026-10-30 | ✅ Fresh | `[0.1.0]`–`[1.3.0]`. 1.3.0 = the audit-driven hardening release: Security / Fixed / Changed (behaviour) / Added, bench table with the load paths. |
| `CLAUDE.md` | 2026-09-21 | 🟡 Stale | Operator-owned, so it was not edited. Five statements are stale: (1) :39-40 "`src/lib.cyr` + future modules … PSF1/PSF2 import" — the parser modules exist and import covers BDF / PCF; (2) :76 "renders 'A' in both fonts" — three built-ins; the demo draws 'A' in the two 8-wide ones, then '…' / '�'; (3) :109 "dep symlinks" — `lib/` holds copies (a symlinked `lib/` is refused); (4) :156 roadmap "Milestones through v1.0" — it now holds the 1.3.x – 1.x.x milestones, the non-goals and the pin procedure; (5) :53-54 "complete and frozen since 1.0.0" — additive since (1.1.0). |
| `CONTRIBUTING.md` | 2026-10-30 | ✅ Fresh | 1.3.0: step 4 names the poison runs CI added. |
| `SECURITY.md` | 2026-10-30 | ✅ Fresh | 1.3.0: accessor fuzz claim now true (unified accessors fuzzed); C1 clamp; load-cost, font-file and thread-safety bullets; the text tab's range rule; the 2026-10-30 audit in the trail (the 'no audit since 0.8.0' warning retired). |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. Re-read: current. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only short notice pointing at gnu.org. Re-read: current. |
| `VERSION` | 2026-10-30 | ✅ Fresh | `1.3.0`; sole source of truth (`cyrius.cyml` resolves `${file:VERSION}`). |
| `cyrius.cyml` | 2026-10-29 | ✅ Fresh | Pin `6.7.18`. Sweep: the `[lib]` comment said agnos "cannot link the stdlib the other three modules use" — only `src/lib.cyr` uses it; corrected. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-10-30 | ✅ Fresh | Index covers 0001–0012; 0012 added for 1.3.0. |
| `adr/template.md` | 2026-05-27 | ✅ Fresh | Re-read: current. |
| `adr/0001-freestanding-font-data-core.md` | 2026-10-29 | ✅ Fresh | Sweep: the status line names the amendments, the built-in set's later ADRs and the 1.1.1 sizing; body as decided. |
| `adr/0002-runtime-font-registry.md` | 2026-10-29 | ✅ Fresh | Sweep: status adds the 0005 width cap (8 → 32) and the BDF / PCF include chain. |
| `adr/0003-codepoint-addressing-runtime-fonts.md` | 2026-10-29 | ✅ Fresh | Sweep: status adds sequences (0005) and sidecar attach (0010). |
| `adr/0004-cp437-glyph-range.md` | 2026-10-29 | ✅ Fresh | Sweep: header gains the 1.1.1 storage re-size pointer. |
| `adr/0005-wide-glyph-and-ligatures.md` | 2026-10-29 | ✅ Fresh | Sweep: status points at 0006 (the 9×16 has stride 2). |
| `adr/0006-vga-9x16-derived-builtin.md` | 2026-10-29 | ✅ Fresh | Sweep: status points at 0011 (extension cells) and the 904-slot buffer. |
| `adr/0007-cga-high-half-from-linux-pd.md` | 2026-10-29 | ✅ Fresh | Sweep: status points at 0011 and the 226-slot buffer. |
| `adr/0008-bdf-import.md` | 2026-10-29 | ✅ Fresh | Sweep: the sketched API never matched the shipped one. The status now lists the differences (signatures, `BDF_PH_*` / `KASHI_BDF_*`, `BG_KIND`, a missing `ENCODING` fails the parse) and points at `api/parsers.md`. |
| `adr/0009-pcf-import.md` | 2026-10-29 | ✅ Fresh | Sweep: ctx is 144 bytes (136 used), not 128 — wrong since 0.7.1. |
| `adr/0010-psf-u-variant.md` | 2026-10-29 | ✅ Fresh | Sweep: replace semantics note the 0.8.0 F9 rollback on a failed attach. |
| `adr/0011-unicode-to-cp437-and-extension-glyphs.md` | 2026-10-29 | ✅ Fresh | Sweep: the reverse index is no longer "the one buffer not sized in bytes" (since 1.1.1); the pre-tag "decide before tagging" note is past tense. |
| `adr/0012-loader-hardening-maps-and-file-reading.md` | 2026-10-30 | ✅ Fresh | New in 1.3.0. Map construction (O(n log n), highest-glyph duplicates per the kernel's con_insert_unipair, a table always addresses its font, complete records), file reading (regular files only, non-blocking + fstat on Linux, sized to the file, over-cap refused). Claims checked: the kernel rule against drivers/tty/vt/consolemap.c; the numbers against the final 1.3.0 measurements. |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-10-13 | ✅ Fresh | Re-read: current. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-10-29 | ✅ Fresh | Sweep: the header date names 1.2.0's array-size paragraph. Every technical claim was re-checked (layout, alignment, test names, arithmetic); `[3616 / 8]` was re-measured on 6.7.18 at 452 slots. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-10-30 | ✅ Fresh | After 1.3.0: the 1.3.x hardening milestone is done; three follow-ups remain (aarch64 CI leg, ledger roll-up, CLAUDE.md). New: a thread-safe library face (1.x.x, gated) and a 2.0-candidates section (prefixing the parser struct constants). The pin procedure's gate adds the poison runs and the aarch64 check. |
| `development/state.md` | 2026-10-30 | ✅ Fresh | 1.3.0 entry; sizes 204,088 / 56,840 B; six test files, 1,389 assertions; the bundle's five leaves. |
| `development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md` | 2026-10-13 | ✅ Fresh | Resolved in 1.1.1; re-read — its 454 + 145 and `5544d63f…` still match. |
| `development/issues/2026-10-29-pcf-cp-to-idx-wraps-above-ffff.md` | 2026-10-29 | ✅ Fresh | RESOLVED in 1.2.2, with a Resolution section: the guard, the unit group, the fuzz probe built from the encoded range, and the mutation result. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-10-30 | ✅ Fresh | 1.3.0: `tag = "1.3.0"`; the suites paragraph names the hardening files and the poison runs; the bundle's five leaves. |
| `guides/loading-psf-fonts.md` | 2026-10-30 | ✅ Fresh | 1.3.0: the PSF1 surrogate and text-tab range rules, duplicates and empty tables, file refusals (ADR 0010 credited for overlong UTF-8, not 'audit F3'). |
| `guides/loading-bdf-fonts.md` | 2026-10-30 | ✅ Fresh | 1.3.0: the file cap refuses rather than truncates; a *Strictness* section (ENDFONT, after-BITMAP, numbers, repeated header keys, duplicate ENCODING rule). |
| `guides/loading-pcf-fonts.md` | 2026-10-30 | ✅ Fresh | 1.3.0: the file cap refuses rather than truncates; non-regular paths refused. |
| `examples/glyph_sheet.cyr` + `.sha256` | 2026-10-09 | ✅ Fresh | Re-read: 678 lines, `5544d63f…`; `head -672` hashes to `b68ca4d9…`, as the header says. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*` + this ledger

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-10-29 | ✅ Fresh | Sweep: forward notes on the 8× reservation (removed in 1.1.1; the guard test now checksums every core byte) and the `cyaudit` workaround (retired at 1.0.1). |
| `audit/2026-05-28-audit.md` | 2026-05-28 | ✅ Fresh | 0.6.0 P(-1). Re-read: current as a record. |
| `audit/2026-05-28-audit-0.8.0.md` | 2026-10-29 | ✅ Fresh | Sweep: "422 assertions total" corrected to 442 (393 + 49, as its own lines and the CHANGELOG say). |
| `audit/2026-10-30-audit.md` | 2026-10-30 | ✅ Fresh | New in 1.3.0: the P(-1) pass over the 1.2.2 surface. 34 findings with fixes and regression tests, *Found clean*, *Open after 1.3.0*, *Verification* (suites, mutation, bench). |
| `benchmarks.md` | 2026-10-30 | ✅ Fresh | "Current" at 1.3.0 with the four new load-path benches (1.2.2 vs 1.3.0, nine interleaved rounds); 1.2.2 Previous; Trend rows; the load-path note updated. |
| `benchmarks/history.csv` | 2026-10-30 | ✅ Fresh | Rows 0.1.0 → 1.3.0; 2026-10-30 adds `1.2.2-rerun` / `1.3.0` with the `load_*` benches. |
| `doc-health.md` | 2026-10-30 | ✅ Fresh | This ledger: a 1.3.0 touched-rows refresh on top of the 1.2.1 full sweep. |

## Tier 7 — `docs/api/` (frozen 0.9.0, sealed 1.0.0)

Public surface unchanged since 1.1.0, which added one function and one
enum group. Re-counted at 1.2.1: 46 `fn kashi_*` declared in `src/` = 44
public (all documented; `_file` variants and `kashi_active_font` under
shared headings) + 2 init-time `kashi_fset*` packers the README declares
internal. All 44 signatures and the 21 enum groups match `src/`.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-10-30 | ✅ Fresh | 1.3.0: a *Thread safety* section (audit L12); `_KashiRtRec` internal naming. |
| `api/core.md` | 2026-10-30 | ✅ Fresh | 1.3.0: `kashi_font_init` warm vs first-call cost; the high-half lookup's per-depth cost; the C1 clamp. |
| `api/loading.md` | 2026-10-30 | ✅ Fresh | 1.3.0: *Reading font files* (the reader's contract table) and *Codepoint maps* (cost, duplicates, empty maps, allocation failure); BDF strictness; caps refuse. |
| `api/accessors.md` | 2026-10-30 | ✅ Fresh | 1.3.0: runtime addressing (a table always addresses its font, the duplicate rule); `kashi_font_seq_glyph` is a binary search. |
| `api/attach.md` | 2026-10-30 | ✅ Fresh | 1.3.0: attach builds aside; file companions validate first, take regular files only and refuse over-cap. |
| `api/parsers.md` | 2026-10-30 | ✅ Fresh | 1.3.0: `kashi_psf_uni_token` nulls and PSF1 surrogates; BDF header uniqueness and number boundaries, after-BITMAP and ctx validation, the loader's ENDFONT rule; PCF's inert ctx after a failed parse. |
| `api/codes.md` | 2026-10-30 | ✅ Fresh | 1.3.0: `_KashiRtRec` / `_KASHI_RT_*` (renamed from `KashiRtRec` / `RT_*`), the empty-map meaning of `_KASHI_RT_UMAP`. |

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

Closed in 1.2.1:

- ~~The minor-closeout full-tree re-read~~ ✅ — owed since 1.1.0 (and
  logged at 1.2.0); done 2026-10-29.

Closed in 1.2.2:

- ~~Stale comments in `src/`~~ ✅ — every one the 1.2.1 sweep listed, plus
  `src/font_bdf.cyr`'s `cyaudit vet` and its "load8 only" line, and two
  test labels.
- ~~`kashi_pcf_cp_to_idx` above U+FFFF~~ ✅ — fixed, with a unit group and a
  fuzz contract (issue resolved).

Open at 1.2.2:

- **`CLAUDE.md`** — five stale statements (its row). Operator-owned: for
  the maintainer to edit.
- ~~**Security audit record**~~ ✅ — `audit/2026-10-30-audit.md` (1.3.0).

Open at 1.1.0:

- **Ledger shape** — `docs/` crossed the ~30-file threshold (36 tracked with
  ADR 0011). Per-file rows were kept "while churn is toolchain-only; move to
  tier roll-ups if a 1.x feature cut reopens the tree" — 1.1.0 is that
  feature cut. The roll-up is still owed (the full-tree re-read was done at
  1.2.1); booked as roadmap 1.3.x.
