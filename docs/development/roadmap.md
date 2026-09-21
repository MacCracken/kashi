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

---

## Moving the cyrius pin to 6.6.6

**Done — 1.0.9 (2026-09-21) pins `cyrius = "6.6.6"`** (was `6.6.4`). Nothing had to change
first — pin bump and rebuild, as the check below predicted.

**What was checked** (7 `.cyr` under `src/`; vendored `lib/` excluded):

- **Windows write corruption: not a kashi exposure.** `O_APPEND` / `O_TRUNC` appear **52
  times, all inside the vendored `lib/` fold, zero in `src/`**. kashi never opens a file for
  writing. Its only file I/O is five read-only `file_read_all(path, buf, cap)` calls in
  `src/lib.cyr` (lines 420, 543, 660, 961, 984) — the runtime font-load path.
- **The read-side half of the PE fix is the interesting one, and it is still not kashi's.**
  6.6.6 stops `file_exists` / `file_read_all` requesting write access, so they now succeed on
  read-only files and volumes. That is exactly kashi's access pattern — fonts are read-only
  data — but the change is in the **PE** access-mode decode, and kashi has no Windows target:
  CI is `ubuntu-latest` only and `src/` carries no `CYRIUS_TARGET_*` branch. No behaviour
  change on Linux or on the agnos build.
- **No shape the new refusals catch.** Zero `struct` declarations in `src/`, so no
  struct/vector copy, no by-value struct parameter, no struct-valued return, no top-level
  struct call. No `async fn`, no `operator` fn, no SIMD-returning fn, no fn mixing pair and
  scalar returns, no `var` inside a top-level block (zero top-level `{` / `if (` / `while (`
  at column 0), no raw `SYS_STATFS`, no `lib/regression.cyr` consumer, no `vec_*` of kashi's
  own, no duplicate top-level global, no symlink in `lib/`.
- **The literal-size question, measured.** `src/font_data.cyr` is the glyph byte-table wall
  and carries kashi's longest string literal at **17,121 bytes** — the largest in this slice
  of the ecosystem, and still well under the **64 KB** threshold of the 6.6.4 read-back
  defect. Not exposed today. ⚠ But it is the closest any repo here gets, and font tables only
  grow: if a future font pushes a single literal past 64 KB, the pin **must** be ≥6.6.4 or
  the table silently reads back from its second byte with `rc=0`. Worth a line in the
  "adding a font" checklist.

**What it gains:** 6.6.5's aggregate-layout fix (silently wrong since 5.8.17) and the
corrected ENTRY stack bases, plus 6.6.6's nine new refusals.

**Verify after bumping:** `cyrius deps` → `cyrius build` → `cyrius test` → **regenerate
`dist/kashi.cyr`**, then render a glyph sheet and compare it byte-for-byte against the
pre-bump render. kashi emits fixed data; a pixel difference after a toolchain bump means
codegen changed what the font says, which is worth stopping for rather than tagging through.

**Result (1.0.9):** every step ran as written. The glyph sheet — 3 fonts × 224 glyphs, every row
byte via `kashi_glyph_row_byte`, cross-checked against `kashi_glyph_row` — is **byte-identical**
between the 6.6.4 and 6.6.6 builds (sha256 `b68ca4d9…`); `dist/kashi.cyr` differs only in its
version stamp; 393 + 49 assertions pass; `cyrius vet` still dependency-free; bench flat within
noise. Full numbers in `CHANGELOG.md` *1.0.9*.

⚠ **`src/font_data.cyr` is the freestanding entry point included directly by the agnos
kernel** (`cyrius.cyml` records why it stays that way). The agnos kernel compiles it with
*its* toolchain, not kashi's, so the bump is only complete when the kernel side has a pin
that accepts whatever the regenerated table looks like. Confirm that before tagging rather
than after.

⚠ The refreshed `lib/` will carry the 6.6.6 `sigil.cyr` and `mabda.cyr`, whose shipped dist
does not pass `cyrfmt --check`. That is harmless here — kashi's format gate never walks
`lib/`. Do not "fix" it in the fold; per the ecosystem rule the repair belongs in the sigil
and mabda source repos, then re-vendors.
