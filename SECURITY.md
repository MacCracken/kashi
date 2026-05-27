# Security Policy

## Reporting Vulnerabilities

Report security issues to: security@agnosticos.org

Do **not** open public issues for security vulnerabilities.

## Scope

kashi is a console-font library: bitmap glyph data plus the machinery to
load and query fonts. Its **freestanding core** (`src/font_data.cyr`) is
consumed directly by the agnos kernel's framebuffer console, so a bug there
runs in kernel context. Security-relevant areas:

- **Accessor bounds** — `kashi_glyph_row` / `kashi_glyph_ptr` must never
  return data outside the glyph buffer or wild-deref for any
  `(font_id, ch, row)` input. Out-of-range inputs return the safe sentinels
  (row `0`, ptr `0`/null). This is the kernel's trust boundary; it is fuzzed
  (`tests/kashi.fcyr`) over the full integer range of each argument.
- **Buffer sizing** — the module-scope `var kashi_font16[1536]` /
  `kashi_font8[768]` declarations reserve 8× their element count (Cyrius
  module-scope u64 units; see `docs/architecture/001-*`). All byte access is
  bounds-bounded to the used region.
- **Font-file parsing** (future, M1+) — PSF1/PSF2 import parses untrusted
  file bytes. When that lands it must validate magic, header lengths, and
  glyph-count bounds before any indexed read (no `../` is relevant; this is
  in-memory parsing, but length/overflow validation is mandatory).

## Out of scope

- The demo binary (`src/main.cyr`) — illustrative only; not a deployment
  surface.
- Rendering policy (color, scaling, scrolling) — owned by the consumer
  (e.g. agnos `fb_console.cyr`), not by kashi.

## Supported Versions

Pre-1.0. The current `VERSION` is the only supported line; report against
it. The full support matrix lands at 1.0.

## Hardening

Per [AGNOS first-party standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/first-party/first-party-standards.md#security-hardening-required-before-every-release),
a security audit pass runs before every release; findings are recorded in
`docs/audit/YYYY-MM-DD-audit.md`.
