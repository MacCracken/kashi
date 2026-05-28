# kashi — Roadmap

> **Last Updated**: 2026-05-28
>
> **Status: N/A — No Open Work.**
>
> kashi reached 1.0.0 with every v1 criterion met. The release
> narrative lives in [`CHANGELOG.md`](../../CHANGELOG.md); this file
> tracks only **forward-facing** work.
>
> Future work will be added here when a concrete need surfaces —
> not before. The shape below is "what could reopen this file,"
> not "what's planned."

## Out of scope — committed (will not be added in 1.x)

These are explicit non-goals. Bringing them in would change kashi
from a bitmap-font data provider into something else.

- **Outline / vector fonts** — TrueType / OpenType is a different
  subsystem entirely. kashi is bitmap-only.
- **Text shaping / BiDi / complex scripts** — kashi hands back glyph
  bitmaps; layout / shaping belongs in a separate library (one that
  could consume kashi as its glyph-data provider).
- **Anti-aliasing / subpixel rendering** — monochrome 1-bit-per-pixel
  only.
- **Rendering policy** (color, scaling, scrolling) — owned by the
  consumer (agnos `fb_console.cyr` does this).

## Out of scope — unbooked (no plans, but additive if asked)

These would be additive on top of the 1.0 surface and don't break
the API freeze. Each is named so that *if* a concrete consumer
need shows up, the scope is already roughly understood.

- **BDF lenient-BBX with cell padding** — accept per-glyph BBX
  variation by padding into the FONTBOUNDINGBOX. Currently strict
  (ADR 0008). Would bring display BDFs with italic overhangs /
  descender variation into scope.
- **PCF per-glyph metric variation** — analog of the above for PCF.
  Currently strict-uniform metrics (ADR 0009).
- **Streaming load APIs** (`kashi_load_*_stream`) — for consumers
  that can't fit the whole file in a single buffer. No concrete
  need yet.
- **Additional built-in fonts** — adding a font id 3 (e.g., a
  higher-density cell or a different character set) is an additive
  change kashi-side; would bump `KASHI_RT_FONT_BASE` from 3 to 4.
  Semver-compatible because the constant is referenced symbolically.

## What would legitimately reopen this file

- A real consumer hits per-glyph metric variation in a PCF / BDF
  they need to render → lenient-BBX cut.
- A kernel finds an actual bounds bug → patch + audit entry.
- agnos picks up a fourth built-in cell size → freestanding-core
  additive.
- A CVE drops against a font format kashi parses → audit + targeted
  fix.

Anything else is either shaping-layer work (separate library) or
out-of-scope feature creep.

## Where to look for…

- **Release history** → [`../../CHANGELOG.md`](../../CHANGELOG.md).
- **Live state** → [`state.md`](state.md).
- **Public API** → [`../api/`](../api/).
- **Decisions** → [`../adr/`](../adr/).
- **Security posture** → [`../../SECURITY.md`](../../SECURITY.md) +
  [`../audit/`](../audit/).
