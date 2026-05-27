---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-05-27 (0.1.0 baseline — initial full inventory).
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
Small repo (well under the ~30-file threshold), so the ledger stays narrow;
if `docs/` grows past ~30 files, switch to tier roll-ups like the agnosticos
pattern.

---

## At a glance — 2026-05-27 inventory (0.1.0 baseline)

All docs authored fresh at the 0.1.0 cut alongside the code. Nothing stale,
nothing archived, no open strategic questions.

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh** | 14 | Everything below — all written/verified at 0.1.0. |
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
| `CHANGELOG.md` | 2026-05-27 | ✅ Fresh | Real `[0.1.0]` entry; SemVer + Keep a Changelog. |
| `CLAUDE.md` | 2026-05-27 | ✅ Fresh | Durable-only; freestanding-vs-full-lib split, parallel-agent note, standard constraints. State deferred to state.md. |
| `CONTRIBUTING.md` | 2026-05-27 | ✅ Fresh | The two faces, accessor-bounds rule, glyph-fidelity rule, Cyrius enum/style notes. |
| `SECURITY.md` | 2026-05-27 | ✅ Fresh | Accessor-bounds = kernel trust boundary; buffer sizing; future PSF-parse validation. |
| `CODE_OF_CONDUCT.md` | 2026-05-27 | 🔵 Evergreen | Contributor Covenant v2.1 reference. |
| `LICENSE` | (scaffold) | 🔵 Evergreen | GPL-3.0-only verbatim. |
| `VERSION` | 2026-05-27 | ✅ Fresh | `0.1.0`; sole source of truth (cyrius.cyml resolves via `${file:VERSION}`). |
| `cyrius.cyml` | 2026-05-27 | ✅ Fresh | Real description; `version=${file:VERSION}`; `repository`; trimmed deps. |

## Tier 2 — `docs/adr/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `adr/README.md` | 2026-05-27 | ✅ Fresh | Index updated with ADR 0001. |
| `adr/template.md` | (scaffold) | ✅ Fresh | Standard ADR template. |
| `adr/0001-freestanding-font-data-core.md` | 2026-05-27 | ✅ Fresh | The load-bearing split decision. Accepted. |

## Tier 3 — `docs/architecture/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `architecture/README.md` | 2026-05-27 | ✅ Fresh | Index updated with note 001. |
| `architecture/001-module-scope-var-array-byte-addressing.md` | 2026-05-27 | ✅ Fresh | The u64-unit BSS quirk the core relies on. |

## Tier 4 — `docs/development/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `development/roadmap.md` | 2026-05-27 | ✅ Fresh | M0 (done) → M4 (v1.0); books PSF import / runtime registry / agnos-1.38.0 consumption. |
| `development/state.md` | 2026-05-27 | ✅ Fresh | 0.1.0 live state: impl-vs-booked, tests (81 assertions), sizes, fidelity. |

## Tier 5 — `docs/guides/` + `docs/examples/`

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-05-27 | ✅ Fresh | Build/test/add-a-font, reflecting the two-faces layout. |
| `examples/.gitkeep` | (scaffold) | ✅ Fresh | Placeholder; the demo lives in `src/main.cyr` for now. Backfill `docs/examples/*.cyr` as the API surface grows. |

---

## Carry-forward / open items

- **`docs/audit/`** — not yet created. First security-audit pass produces
  `docs/audit/YYYY-MM-DD-audit.md` (P(-1) before a real release).
- **`docs/benchmarks.md`** — not yet created. Bench numbers currently live
  inline in state.md / CHANGELOG; promote to a CSV-backed `benchmarks.md`
  once a version-over-version trend exists.
- **`docs/api/`** — defer until the library face (PSF import, registry) lands
  and the surface is large enough that grepping beats reading.
