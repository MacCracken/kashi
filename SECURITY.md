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
  trust boundary. `tests/kashi.fcyr` fuzzes the core accessors against an
  independent cell model, and since 1.3.0 the unified and runtime accessors
  too, over the full integer range of each argument.
- **Buffer sizing** — the freestanding core's module-scope glyph
  buffers (`kashi_font16` 452 / `kashi_font8` 226 / `kashi_font9_16`
  904 u64 slots = 3,616 / 1,808 / 7,232 bytes — 224 CP437 + 2
  extension glyphs each) hold **exactly** the bytes used since 1.1.1
  (until then they reserved 8× — see
  `docs/architecture/001-module-scope-var-array-byte-addressing.md`).
  With no slack, a write past a buffer's end lands on the next
  variable, so the bounds below are the only thing between a
  table-authoring slip and corruption: all byte access is bounded to
  the used region; the table-builder helpers (`kashi_fset16`,
  `kashi_fset8`) reject cells outside `[0x20, KASHI_EXT_LAST]` (audit
  2026-05-27 F1), and since 1.1.1 a test sees each of those four
  guards fail (`src/test.cyr` "fset guard"), and the "storage" group
  fails if a cell is added without growing the buffers.
- **Unicode → cell mapping** (1.1.0) — `kashi_cp437_from_unicode` is
  total over the `i64` domain (anything unmapped → `0`, which every
  accessor treats as out of range) and only ever answers `0` or an
  encoded cell; its binary search indexes a 128-slot table
  (`kashi_cp437_rev[128]`, one u64 slot per entry), clamped to those 128
  slots whatever the count global holds (1.3.0, audit 2026-10-30 C1), that
  the init-time writer fills slot by slot, bounding slot, codepoint and
  cell (a bad one is skipped, never written past the table). Fuzzed in
  `tests/kashi.fcyr` (`fuzz_unicode`) and swept over all of Unicode in
  `tests/kashi.tcyr`.
- **Font-file parsing** — PSF1/PSF2 (`src/font_psf.cyr`), BDF
  (`src/font_bdf.cyr`), and PCF (`src/font_pcf.cyr`) all parse
  untrusted file bytes. Each parser is **heapless and
  dependency-free** (`cyrius vet` reports "no dependencies"),
  validation-first (magic, header, geometry, glyph count, total
  length all checked before any indexed read past the fixed header),
  and fuzzed in `tests/kashi.fcyr`. Since 1.3.0 that harness mutates
  generated valid fonts and checks every answer of a valid one against
  what was generated (round-trip oracles), not only that nothing crashed.
- **Load cost** (1.3.0, audit 2026-10-30 L1 / L2) — every codepoint map
  and sequence list is built in O(n log n) and searched in O(log n), so no
  input within the caps can make a load or a lookup quadratic. Before
  1.3.0 a 256 KiB PSF stalled a load for ~200 s.
- **Font files** (1.3.0, L3 – L5, L14) — the `*_file` loaders take only a
  regular file of at most their cap. On Linux they open non-blocking and
  without acquiring a controlling terminal, then fstat the open
  descriptor, so a FIFO, tty or device path is refused at once (the
  CVE-2017-16611 pattern). A larger file is refused, not truncated, and the
  buffer is sized to the file. Details:
  [`docs/api/loading.md`](docs/api/loading.md#reading-font-files).
- **Thread safety** — the freestanding core is safe for concurrent
  readers once initialised; the library face is not thread-safe (audit
  L12; contract in [`docs/api/README.md`](docs/api/README.md#thread-safety)).
- **Untrusted Unicode tables** — the PSF2 UTF-8 decoder rejects
  codepoints > U+10FFFF, the UTF-16 surrogate range (audit F2,
  0.6.0), and overlong sequences per RFC 3629 (ADR 0010, 0.7.2).
  The binary sidecar tab paths use the same decoder; PSF1 tables end at a
  surrogate since 1.3.0 (P1), and the text tab refuses surrogates and
  anything past U+10FFFF (L9).

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
a security audit pass belongs before every release; kashi runs one as
its P(-1) step at minor and v1.0 cuts (`CLAUDE.md`), and findings are
recorded in `docs/audit/YYYY-MM-DD-audit.md`. (1.1.0 and 1.2.0 shipped
without one; 1.3.0 is that pass.) The audit trail to date:

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
- **`docs/audit/2026-10-30-audit.md`** — 1.3.0 P(-1) pass over the whole
  1.2.2 surface, multi-agent with every finding reproduced or refuted.
  34 findings, 0 exploitable for code execution. The one high was
  quadratic map construction: a 256 KiB font stalled a load for 204 s. The
  others include file loaders blocking on FIFOs / ttys (CVE-2017-16611
  pattern), truncated over-cap files and BDFs loading as partial fonts,
  and half-built records on allocation failure. All fixed except thread
  safety (documented), plus 47-of-57 surviving test mutants answered with
  780 new assertions and round-trip fuzz oracles.
