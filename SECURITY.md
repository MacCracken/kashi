# Security Policy

## Reporting Vulnerabilities

Report security issues to: security@agnosticos.org

Do **not** open public issues for security vulnerabilities.

## Scope

kashi is a console-font library: bitmap glyph data plus the
machinery to load and query fonts. Its **freestanding core**
(`src/font_data.cyr`) is consumed directly by the agnos kernel's
framebuffer console, so a bug there runs in kernel context.
Security-relevant areas:

- **Accessor bounds** — `kashi_glyph_row` / `kashi_glyph_ptr` /
  the unified `kashi_font_row` / `kashi_font_ptr` must never return
  data outside the glyph buffer or wild-deref for any
  `(font_id, ch, row, byte_idx)` input. Out-of-range inputs return
  safe sentinels (`0` row byte, `0`/null ptr). This is the kernel's
  trust boundary; it is fuzzed (`tests/kashi.fcyr`) over the full
  integer range of each argument.
- **Buffer sizing** — the freestanding core's module-scope BSS
  buffers (`var kashi_font16[3584]`, `var kashi_font8[1792]`,
  `var kashi_font9_16[7168]`) reserve 8× their byte count (Cyrius
  module-scope u64 units; see
  `docs/architecture/001-module-scope-var-array-byte-addressing.md`).
  All byte access is bounded to the used region; the table-builder
  helpers (`kashi_fset16`, `kashi_fset8`) reject out-of-range
  codepoints (audit 2026-05-27 F1).
- **Font-file parsing** — PSF1/PSF2 (`src/font_psf.cyr`), BDF
  (`src/font_bdf.cyr`), and PCF (`src/font_pcf.cyr`) all parse
  untrusted file bytes. Each parser is **heapless and
  dependency-free** (`cyaudit vet` reports "no dependencies"),
  validation-first (magic, header, geometry, glyph count, total
  length all checked before any indexed read past the fixed header),
  and fuzzed in `tests/kashi.fcyr` (~7,500 rounds across the four
  parsers and the text-tab attach path).
- **Untrusted Unicode tables** — the PSF2 UTF-8 decoder rejects
  codepoints > U+10FFFF, the UTF-16 surrogate range (audit F2,
  0.6.0), and overlong sequences per RFC 3629 (audit F3, 0.7.2).
  Sidecar tab attach paths use the same decoder.

## Out of scope

- The demo binary (`src/main.cyr`) — illustrative only; not a
  deployment surface.
- Rendering policy (color, scaling, scrolling) — owned by the
  consumer (e.g. agnos `fb_console.cyr`), not by kashi.

## Supported Versions

1.x — the current major line. Security fixes are issued against
the latest 1.x.y; older 1.x.y are best-effort. Pre-1.0 releases are
not supported.

## Hardening

Per [AGNOS first-party standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-standards.md#security-hardening-required-before-every-release),
a security audit pass runs before every release; findings are
recorded in `docs/audit/YYYY-MM-DD-audit.md`. The audit trail to
date:

- **`docs/audit/2026-05-27-audit.md`** — 0.2.0 P(-1) on the
  freestanding-core accessor surface. Proved memory-safe for the
  full signed-64-bit input domain. 3 findings, all fixed (fset
  guard, ready-flag reader, comment fix).
- **`docs/audit/2026-05-28-audit.md`** — 0.6.0 P(-1) re-walk of the
  post-0.1.0 surface (PSF parser, runtime registry, map/sequence
  build, wide-glyph accessors, VGA 9×16 derivation, CGA dual-source
  high half). 3 findings, all fixed (glyph_count overflow guard,
  strict UTF-8 cp range, stale comments).
- **`docs/audit/2026-05-28-audit-0.8.0.md`** — 0.8.0 hardening
  pass, research-driven against the 2020–2026 font-parser CVE
  corpus. 9 findings, 0 exploitable, all fixed (PSF/PCF format-byte
  unknown bits rejection, PCF duplicate-TOC rejection, BDF integer
  overflow tightened, sidecar attach atomic-on-failure per
  CVE-2015-1803 pattern).
