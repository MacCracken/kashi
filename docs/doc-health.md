---
name: kashi Documentation Health
description: Living state of doc currency in the kashi repo — fresh / stale / archive / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — kashi

> **Last refresh**: 2026-10-30, after 1.3.0. The ledger was rolled up into
> tiers (open since 1.1.0), and `CLAUDE.md`'s stale statements were fixed.
> **Last full-tree re-read**: 2026-10-29 (1.2.1), with every file checked
> against the tree. 1.3.0's own changes were graded as they were made.
> **Next full sweep**: at the 1.4 minor closeout.
>
> **Refresh cadence**: when a doc is touched, re-grade its tier below. Run a
> full-tree sweep at each minor-version closeout. The release history of
> this ledger is in `CHANGELOG.md` and git; it is not repeated here.
>
> **Scope**: this repo only (`kashi`) — the `docs/` tree plus root-level
> files (README, CLAUDE.md, CHANGELOG, CONTRIBUTING, SECURITY,
> CODE_OF_CONDUCT, LICENSE, VERSION, cyrius.cyml). Sibling-repo docs are not
> audited here. Cross-repo version drift lives in [`development/state.md`](development/state.md).
>
> **Location**: `docs/doc-health.md` (whole-tree scope), **not** under
> `docs/development/` — the ledger sweeps the whole tree and the location
> matches the scope.

This is a **ledger**, not a one-time audit. Rewrite it in place as docs
change. With 49 files in scope (40 under `docs/`), it is kept as **tier
roll-ups**, the agnosticos pattern past ~30 files. It has one row per
tier, and per-file notes only where a file needs its own attention.

**Status key**:
- ✅ **Fresh** = describes the current tree.
- 🟡 **Stale** = at least one statement no longer describes it (the note
  says which).
- 🔵 **Evergreen** = standard text, re-read annually.
- 📦 **Archive** = kept for the record, not maintained.
- ❓ **Open question** = a decision is owed, not an edit.

Historical records (ADRs, audits, resolved issues) are ✅ when they record
the decision or pass as made and their status lines point at what changed
later.

---

## At a glance — 2026-10-30 (1.3.0)

| Bucket | Files | Which |
|---|---|---|
| ✅ **Fresh** | 47 | Everything in scope but the two evergreen files |
| 🟡 **Stale** | 0 | — |
| 🔵 **Evergreen** | 2 | `CODE_OF_CONDUCT.md`, `LICENSE` |
| 📦 **Archive** | 0 | — |
| ❓ **Open question** | 0 | — |

## Tiers

| Tier | Files | Status | Last touched | What keeps it fresh |
|---|---|---|---|---|
| **Root files** | 9: `README.md`, `CHANGELOG.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `LICENSE`, `VERSION`, `cyrius.cyml` | ✅ (2 🔵) | 2026-10-30 | Version, `tag =` lines, the pin, test and CI steps, the security trail. See *Refresh triggers*. |
| **`docs/api/`** | 7: `README`, `core`, `loading`, `accessors`, `attach`, `parsers`, `codes` | ✅ | 2026-10-30 (1.3.0) | Any change to a public function, constant, result or cost. Last count: 44 public functions + 2 internal packers in `src/`, 21 enum groups, every signature matching `src/` (re-checked 2026-10-29). |
| **`docs/adr/`** | 14: `README`, `template`, 0001 – 0012 | ✅ | 2026-10-30 (0012 added) | A new decision gets a new ADR and an index line; a decision changed later gets a status-line pointer, not a rewrite. |
| **`docs/architecture/`** | 2: `README`, `001` (module-scope buffer sizing) | ✅ | 2026-10-29 | A change to glyph storage, globals layout, or what array sizes the oldest consumer pin accepts. |
| **`docs/development/`** | 4: `roadmap`, `state`, 2 issues (both resolved) | ✅ | 2026-10-30 | `state.md` every release. `roadmap.md` when a milestone ships or a trigger arrives (it stays forward-facing). An issue gets a Resolution section when fixed. |
| **`docs/guides/` + `docs/examples/`** | 4 guides + `glyph_sheet.cyr` / `.sha256` | ✅ | 2026-10-30 | A guide when its loader's behaviour or the consumer snippets change. The glyph-sheet pin only with a deliberate glyph change (CHANGELOG says why). |
| **`docs/audit/`** | 4 audit records (2026-05-27, 2026-05-28, 0.8.0, 2026-10-30) | ✅ | 2026-10-30 | A new record per P(-1) pass. Older records get forward notes, never edits to their findings. |
| **Benchmarks** | `benchmarks.md`, `benchmarks/history.csv` | ✅ | 2026-10-30 | Every release: CSV rows from an interleaved A/B, then *Current* and the trend table. |
| **This ledger** | `doc-health.md` | ✅ | 2026-10-30 | Itself. |

### Per-file notes

- **`CLAUDE.md`** (refreshed 2026-10-30). Five statements had been stale
  since 1.1.0 – 1.3.0 and are fixed:
  - the library face's modules and formats;
  - the frozen-API wording (it grows additively);
  - the demo's output;
  - `lib/` as copies rather than symlinks;
  - the roadmap's description.

  Also updated: the Quick Start (every suite, the poison runs, the fuzz
  oracles), the P(-1) cleanliness and audit steps (strict lint, poison,
  measured worst-case load cost), the core's consumer-pin syntax rule, and
  the Documentation list (`docs/api/`, `docs/audit/`, issues, benchmarks).
  Keep it free of volatile state, as its own header says.
- **`CODE_OF_CONDUCT.md`, `LICENSE`** (🔵, 2026-05-27): the Contributor
  Covenant v2.1 reference, and the GPL-3.0-only short notice pointing at
  gnu.org. Last re-read 2026-10-29.
- **`CHANGELOG.md`** is history. It is graded only for its newest entry and
  for the `[Unreleased]` section.
- **`docs/examples/glyph_sheet.cyr`**: 678 lines hashing to `5544d63f…`;
  `head -672` still hashes to the 1.0.x pin `b68ca4d9…`, as its header says.

## Refresh triggers

What to touch when. A change that ships without these is how the ledger
goes stale.

| When this changes… | …update these |
|---|---|
| **A release** | `VERSION`; `CHANGELOG.md`; `docs/development/state.md`; the `tag =` lines in `README.md` and `guides/getting-started.md`; `README.md`'s status line; this ledger |
| **The cyrius pin** | `cyrius.cyml`, `cyrius.lock`; `CONTRIBUTING.md` and `guides/getting-started.md` (the pin they name); `state.md`; benchmarks (A/B against the old pin); the roadmap's pin procedure if a step changed |
| **A public function, constant, result or cost** | `docs/api/` (and its function count); the affected guide; `README.md` if the surface summary moves |
| **A loader's behaviour on some input** | `api/loading.md` (*Reading font files* / *Codepoint maps*), the format's guide, `SECURITY.md` if it is a trust-boundary rule |
| **A new built-in font or cell** | `api/core.md`, `api/codes.md`, `architecture/001` (sizes), `SECURITY.md` (buffer sizes), the glyph-sheet pin, the add-a-font checklist in `getting-started.md` |
| **A P(-1) pass** | a new `docs/audit/` record, `SECURITY.md`'s trail, `state.md`, an ADR for any decision it forced |
| **Tests or CI steps** | `CONTRIBUTING.md`, `CLAUDE.md`'s Quick Start (if a command changes), `getting-started.md`'s suites paragraph, `state.md`'s Tests section |
| **A consumer's pin or dep shape** | `state.md` *Consumers*; `architecture/001` if the oldest pin moves past a syntax limit |

## Open items

- None for the docs themselves. Roadmap work that will touch them
  (`development/roadmap.md`): the aarch64 CI leg (`CONTRIBUTING.md`, the
  CI notes in `state.md`) and the 1.4.x / 1.5.x milestones when their
  triggers arrive.

Closed items are recorded where they were closed: the 1.0.10, 1.2.1, 1.2.2
and 1.3.0 CHANGELOG entries, and the roll-up and `CLAUDE.md` refresh in
`[Unreleased]`.
