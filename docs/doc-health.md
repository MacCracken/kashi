---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-05-27 (**0.4.0** cut — rest of M2: registry
> niceties + VGA extended to full CP437). **Refresh cadence**: when a doc
> is touched, update its row. Full-tree sweep at minor-version closeouts.
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
Small repo (well under the ~30-file threshold), so the ledger stays narrow;
if `docs/` grows past ~30 files, switch to tier roll-ups like the agnosticos
pattern.

---

## At a glance — 2026-05-27 inventory (0.1.0 baseline)

All docs authored fresh at the 0.1.0 cut alongside the code. Nothing stale,
nothing archived, no open strategic questions.

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 21 | 0.1.0 inventory + hardening docs + M1 docs (ADR 0002, PSF guide) + M2 docs (ADR 0003, ADR 0004). |
| 🟡 **Stale** | 0 | — |
| 🟠 **Read-through outstanding** | 0 | — |
| 🔵 **Probably evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` — standard; re-read annually. |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — |

---

## Tier 1 — Root files

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-05-27 | ✅ Fresh | Two-faces architecture, built-in font table, freestanding-core API, agnos consumption contract, build/test. |
| `CHANGELOG.md` | 2026-05-27 | ✅ Fresh | `[0.1.0]`, `[0.2.0]`, `[0.3.0]`, `[0.4.0]` (rest of M2: niceties + CP437 in VGA). SemVer + Keep a Changelog. |
| `CLAUDE.md` | 2026-05-27 | ✅ Fresh | Durable-only; freestanding-vs-full-lib split, one-owner/two-faces boundary discipline, standard constraints. State deferred to state.md. |
| `CONTRIBUTING.md` | 2026-05-27 | ✅ Fresh | The two faces, accessor-bounds rule, glyph-fidelity rule, Cyrius enum/style notes. |
| `SECURITY.md` | 2026-05-27 | ✅ Fresh | Accessor-bounds = kernel trust boundary; buffer sizing; future PSF-parse validation. |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference. |
| `LICENSE` | (scaffold) | 🔵 Evergreen | GPL-3.0-only verbatim. |
| `VERSION` | 2026-05-27 | ✅ Fresh | `0.1.0`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-05-27 | ✅ Fresh | Real description; `version=${file:VERSION}`; `repository`; trimmed deps. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-05-27 | ✅ Fresh | Index updated with ADR 0001 + 0002. |
| `adr/template.md` | (scaffold) | ✅ Fresh | Standard ADR template. |
| `adr/0001-freestanding-font-data-core.md` | 2026-05-27 | ✅ Fresh | The load-bearing split decision. Accepted. |
| `adr/0002-runtime-font-registry.md` | 2026-05-27 | ✅ Fresh | M1: runtime registry, unified dispatch, width≤8, negative-error convention. Accepted (addressing refined by 0003). |
| `adr/0003-codepoint-addressing-runtime-fonts.md` | 2026-05-27 | ✅ Fresh | M2: PSF Unicode table → codepoint→glyph map; codepoint-addressed accessors + raw-index `kashi_rt_glyph_*`. Accepted. |
| `adr/0004-cp437-glyph-range.md` | 2026-05-27 | ✅ Fresh | 0.4.0: widen freestanding range to `0x20..0xFF` (full CP437) in VGA 8×16 from Linux PD source; CGA high-half blank. Accepted. |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-05-27 | ✅ Fresh | Index updated with note 001. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-05-27 | ✅ Fresh | The u64-unit BSS quirk the core relies on. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-05-27 | ✅ Fresh | M0 ✅, M1 ✅, M2 ✅ (Unicode + niceties + CP437) → M3 (agnos 1.38.0) / 0.5.0 wide-glyph. |
| `development/state.md` | 2026-05-27 | ✅ Fresh | Live state through 0.4.0: range widened to CP437, active-font knob, tests (212 assertions), deps. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-05-27 | ✅ Fresh | Build/test/add-a-font, reflecting the two-faces layout. |
| `guides/loading-psf-fonts.md` | 2026-05-27 | ✅ Fresh | How to load PSF1/PSF2; codepoint vs raw-index addressing (M2); hot-path ptr pattern; limits. |
| `examples/.gitkeep` | (scaffold) | ✅ Fresh | Placeholder; the demo lives in `src/main.cyr` for now. Backfill `docs/examples/*.cyr` as the API surface grows. |

## Tier 6 — `docs/audit/` + `docs/benchmarks*`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `audit/2026-05-27-audit.md` | 2026-05-27 | ✅ Fresh | First P(-1) security/hardening pass. 3 findings fixed, 2 noted; accessor surface proven `i64`-safe. |
| `benchmarks.md` | 2026-05-27 | ✅ Fresh | Methodology + 0.1.0 baseline + dispatch/codepoint-lookup overhead (M2); points at the CSV. |
| `benchmarks/history.csv` | 2026-05-27 | ✅ Fresh | Per-`(version,benchmark)` rows; append each release. |

---

## Carry-forward / open items

- ~~**`docs/audit/`**~~ — ✅ created 2026-05-27 (`2026-05-27-audit.md`, first
  P(-1) pass). Re-audit when M2 fonts / PSF import land.
- ~~**`docs/benchmarks.md`**~~ — ✅ created 2026-05-27, CSV-backed
  (`benchmarks/history.csv`). 0.1.0 is the first row; append per release to
  build the version-over-version trend.
- **`docs/api/`** — defer until the library face (PSF import, registry) lands
  and the surface is large enough that grepping beats reading.
