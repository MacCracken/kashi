---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-10-29 (**1.2.1** — the full-tree re-read owed at the
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
| `README.md` | 2026-10-29 | ✅ Fresh | Sweep: `tag = "1.2.1"`; status line names 1.2.1; the agnos path-only snippet carries the 6.7.6 caveat. Rest re-read: current. |
| `CHANGELOG.md` | 2026-10-29 | ✅ Fresh | `[0.1.0]`–`[1.2.1]`. History; the 1.2.1 entry records the bump, the sweep and its corrections. |
| `CLAUDE.md` | 2026-09-21 | 🟡 Stale | Operator-owned, so it was not edited. Five statements are stale: (1) :39-40 "`src/lib.cyr` + future modules … PSF1/PSF2 import" — the parser modules exist and import covers BDF / PCF; (2) :76 "renders 'A' in both fonts" — three built-ins; the demo draws 'A' in the two 8-wide ones, then '…' / '�'; (3) :109 "dep symlinks" — `lib/` holds copies (a symlinked `lib/` is refused); (4) :156 roadmap "Milestones through v1.0" — it is now the open-issue status, the out-of-scope lists and the pin procedure; (5) :53-54 "complete and frozen since 1.0.0" — additive since (1.1.0). |
| `CONTRIBUTING.md` | 2026-10-29 | ✅ Fresh | Sweep: the pin is 6.7.18. Rest current. |
| `SECURITY.md` | 2026-10-29 | ✅ Fresh | Sweep: fuzz rounds corrected (~8,500: PSF 4,000 / BDF 2,000 / PCF 1,500 / text-tab 1,000; it said ~7,500 across "four parsers"). The audit paragraph now says the P(-1) pass is at minor / v1.0 cuts and that 1.1.0 / 1.2.0 have no audit file (open item below). |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference + conduct@ address. Re-read: current. |
| `LICENSE` | 2026-05-27 | 🔵 Evergreen | GPL-3.0-only short notice pointing at gnu.org. Re-read: current. |
| `VERSION` | 2026-10-29 | ✅ Fresh | `1.2.1`; sole source of truth (`cyrius.cyml` resolves `${file:VERSION}`). |
| `cyrius.cyml` | 2026-10-29 | ✅ Fresh | Pin `6.7.18`. Sweep: the `[lib]` comment said agnos "cannot link the stdlib the other three modules use" — only `src/lib.cyr` uses it; corrected. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-10-29 | ✅ Fresh | Sweep: index entries for 0001 / 0002 / 0003 / 0004 / 0008 / 0010 carry their later pointers (0005 width, 0006 ids, 0007 CGA, the BDF API note, `attach_unicode_*` naming). |
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

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-10-13 | ✅ Fresh | Re-read: current. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-10-29 | ✅ Fresh | Sweep: the header date names 1.2.0's array-size paragraph. Every technical claim was re-checked (layout, alignment, test names, arithmetic); `[3616 / 8]` was re-measured on 6.7.18 at 452 slots. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-10-29 | ✅ Fresh | 1.2.1: the 6.7.9 → 6.7.18 precedent; step 4's scratch-consumer recipe updated for 6.7.6 (`git` + `tag` + `path`, `CYRIUS_LOCAL=1`). |
| `development/state.md` | 2026-10-29 | ✅ Fresh | 1.2.1 entry; pin 6.7.18; sizes (unchanged) and hashes; bench line; `vet src/lib.cyr` description corrected (4 deps = the four included files). |
| `development/issues/2026-10-13-font-arrays-declared-u64-for-byte-tables.md` | 2026-10-13 | ✅ Fresh | Resolved in 1.1.1; re-read — its 454 + 145 and `5544d63f…` still match. |
| `development/issues/2026-10-29-pcf-cp-to-idx-wraps-above-ffff.md` | 2026-10-29 | ✅ Fresh | New, OPEN: `kashi_pcf_cp_to_idx` masks `cp >> 8` to a byte, so a codepoint above U+FFFF wraps. Found by this sweep; the fix is a source change, so it is not in 1.2.1. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-10-29 | ✅ Fresh | Sweep: the pin is 6.7.18; `tag = "1.2.1"`; the CI fmt/lint walk includes `docs/examples/`; the consumer `path` description matches the stack (only crab keeps a live `path`). |
| `guides/loading-psf-fonts.md` | 2026-10-29 | ✅ Fresh | Sweep: success is `font_id ≥ KASHI_RT_FONT_BASE` (3), not ≥ 2; `kashi_rt_font_stride` is runtime-only (0 for built-ins); attach on a built-in returns `0 - KASHI_EINVAL`; the snippet includes `lib/kashi.cyr` (the consumer path). |
| `guides/loading-bdf-fonts.md` | 2026-10-29 | ✅ Fresh | Sweep: the consumer include (`lib/kashi.cyr`). The minimal-BDF example's rows were re-checked by a probe. |
| `guides/loading-pcf-fonts.md` | 2026-10-29 | ✅ Fresh | Sweep: the consumer include. |
| `examples/glyph_sheet.cyr` + `.sha256` | 2026-10-09 | ✅ Fresh | Re-read: 678 lines, `5544d63f…`; `head -672` hashes to `b68ca4d9…`, as the header says. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*` + this ledger

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-10-29 | ✅ Fresh | Sweep: forward notes on the 8× reservation (removed in 1.1.1; the guard test now checksums every core byte) and the `cyaudit` workaround (retired at 1.0.1). |
| `audit/2026-05-28-audit.md` | 2026-05-28 | ✅ Fresh | 0.6.0 P(-1). Re-read: current as a record. |
| `audit/2026-05-28-audit-0.8.0.md` | 2026-10-29 | ✅ Fresh | Sweep: "422 assertions total" corrected to 442 (393 + 49, as its own lines and the CHANGELOG say). |
| `benchmarks.md` | 2026-10-29 | ✅ Fresh | "Current" at 1.2.1 (1.2.0 / 6.7.9 vs 1.2.1 / 6.7.18, flat within ±0.9 %); 1.2.0 Previous, 1.1.1 and 1.1.0 Earlier; Trend rows; a stale "see Current" pointer fixed. |
| `benchmarks/history.csv` | 2026-10-29 | ✅ Fresh | Rows 0.1.0 → 1.2.1; 2026-10-29 has two interleaved sessions (`1.1.1-on-6.6.20` / `1.2.0`, `1.2.0-on-6.7.9` / `1.2.1`). |
| `doc-health.md` | 2026-10-29 | ✅ Fresh | This ledger: the full-tree re-read at 1.2.1. |

## Tier 7 — `docs/api/` (frozen 0.9.0, sealed 1.0.0)

Public surface unchanged since 1.1.0, which added one function and one
enum group. Re-counted at 1.2.1: 46 `fn kashi_*` declared in `src/` = 44
public (all documented; `_file` variants and `kashi_active_font` under
shared headings) + 2 init-time `kashi_fset*` packers the README declares
internal. All 44 signatures and the 21 enum groups match `src/`.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `api/README.md` | 2026-10-29 | ✅ Fresh | Sweep: "changing none" → changed no signature (and names the two result changes); the return-shape section now says "most", gives each parser's own codes and lists the exceptions (attach / `set_active_font`, index-or-`0 - 1`, `uni_token`). 44 public functions re-counted. |
| `api/core.md` | 2026-10-29 | ✅ Fresh | Sweep: `kashi_glyph_row`'s font id is 0..2 only (runtime ids → 0); 226 cells per buffer; costs at 1.2.1 (~18 ns row, ~21 µs init, ~25 ns high-half map); the hot path is ~7 ns per glyph + ~1 ns per row. |
| `api/loading.md` | 2026-10-29 | ✅ Fresh | Sweep: PCF rows are unpadded `ceil(width / 8)` bytes, not "1-byte-stride". |
| `api/accessors.md` | 2026-10-29 | ✅ Fresh | Sweep: the perf block was labelled 0.8.0; now 1.2.1 figures, and the 7 ns is attributed to `kashi_glyph_ptr`. `kashi_font_seq_glyph` can hit on a BDF / PCF font with an attached binary table. |
| `api/attach.md` | 2026-10-29 | ✅ Fresh | Sweep: an oversized file is truncated to 256 KiB, not an error; the text-table example's `tab_len` is 39, not 35. |
| `api/parsers.md` | 2026-10-29 | ✅ Fresh | Sweep: `cyaudit vet` → `cyrius vet`; parsers use load and store intrinsics; overlong UTF-8 credited to ADR 0010; PCF row form; ⚠ the `kashi_pcf_cp_to_idx` caveat (issue 2026-10-29). |
| `api/codes.md` | 2026-10-29 | ✅ Fresh | Sweep: the header notes the 1.1.0 `KashiExtRange` and that `KASHI_EXT_LAST` / `_COUNT` may rise. |

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

Open at 1.2.1:

- **`CLAUDE.md`** — five stale statements (its row). Operator-owned: for
  the maintainer to edit.
- **Stale comments in `src/`** — out of a no-source release's reach. A
  comment edit changes `src/` and `dist/kashi.cyr`; fold these into the
  next source change (the PCF fix is the natural one):
  - `src/lib.cyr`:17 lists only `font_psf`, and :22 says `cyaudit vet`;
  - `src/lib.cyr`:33, :38, :389, :422 and :1101 give built-ins as "0,1" and
    runtime ids as "≥ 2" (they are 0..2 and ≥ 3);
  - `src/lib.cyr`:421 says a PSF file over 256 KiB "is rejected". The read
    truncates at the cap, and the parser rejects the result only when the
    cut lands inside the declared glyph data;
  - `src/lib.cyr`:180 says `w <= count <= 512` (PSF2 allows 65,536);
  - `src/font_pcf.cyr`:25, :112, :131 say the ctx is 128 bytes (it is 144),
    and :5 says `cyaudit vet`;
  - `src/font_psf.cyr`:33 says runtime ids are "≥ 2";
  - `src/font_data.cyr`:141-143 say an array size cannot be an expression
    (true below 6.7.2; the reason now is the consumers' pins), and :600 says
    the CGA high half came in 0.5.1 (it was 0.5.2).
- **Security audit record** — no `docs/audit/` file since 0.8.0. The P(-1)
  pass is due at minor cuts and 1.1.0 / 1.2.0 have none (`SECURITY.md` now
  says so).
- **`kashi_pcf_cp_to_idx` above U+FFFF** —
  `development/issues/2026-10-29-pcf-cp-to-idx-wraps-above-ffff.md`.

Open at 1.1.0:

- **Ledger shape** — `docs/` crossed the ~30-file threshold (36 tracked with
  ADR 0011). Per-file rows were kept "while churn is toolchain-only; move to
  tier roll-ups if a 1.x feature cut reopens the tree" — 1.1.0 is that
  feature cut. The roll-up is still owed (the full-tree re-read was done at
  1.2.1).
