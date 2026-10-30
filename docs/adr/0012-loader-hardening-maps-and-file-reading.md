# 0012 — Loader hardening: how codepoint maps are built, and how font files are read

**Status**: Accepted
**Date**: 2026-10-30
**Refines**: [ADR 0003](0003-codepoint-addressing-runtime-fonts.md) (the codepoint→glyph map), [ADR 0005](0005-wide-glyph-and-ligatures.md) (sequence lookup), [ADR 0008](0008-bdf-import.md) / [ADR 0009](0009-pcf-import.md) / [ADR 0010](0010-psf-u-variant.md) (the file caps)
**Found by**: the 1.3.0 P(-1) audit, [`../audit/2026-10-30-audit.md`](../audit/2026-10-30-audit.md) (L1 – L10, L13, L14)

## Context

The 2026-10-30 audit measured the library face's loaders against hostile
but in-cap input and found that how they build maps and read files was
never decided. Each loader had grown its own copy of the same code.

1. **Map construction was quadratic.** Every codepoint map (PSF inline
   table, BDF loader, PCF loader, text-table attach) was insertion-sorted.
   On a table in reverse order that is O(n²), and the caps admit large n.
   Measured on 1.2.2:
   - a 256 KiB PSF2 took 204 s to load;
   - a 256 KiB text table took 22.6 s to attach;
   - 65,536 BDF glyphs in descending ENCODING order took 13.3 s.

   Real console fonts load in under a millisecond, so this was pure
   exposure.
2. **Duplicate codepoints had no rule.** 85 of the 136 kbd console fonts
   the audit surveyed list one codepoint for several glyphs. Which glyph
   the binary search found depended on the map's size: kashi picked the
   first listed 81 times and the last 108 times.
3. **"Has a table" and "maps nothing" were one state.** A font whose table
   mapped no single codepoint (only sequences, or nothing usable) stored
   no map. It therefore fell back to identity addressing, and U+00E7 'ç'
   drew glyph 0xE7 'τ'. Attaching a sequence-only table to a mapped font
   did the same to it.
4. **Maps were attached after registration.** A loader registered the
   font first and built its map afterwards, ignoring allocation failures,
   so a load could succeed with a half-built record. Attach cleared the
   live map before parsing the new table.
5. **Sequences were a linear scan.** A hostile font with ~130k sequences
   made every `kashi_font_seq_glyph` call cost ~0.8 ms.
6. **File reading was unsafe in four ways.** The `*_file` paths:
   - allocated a buffer of their full cap (up to 4 MiB) before opening the
     path, and kept it even when the path was missing;
   - cut a larger file at the cap and loaded what fit;
   - blocked forever on a FIFO or a tty;
   - made a tty the caller's controlling terminal (the CVE-2017-16611
     pattern).

## Decision

**One map builder, O(n log n), with a fixed rule for duplicates.**
- **Sorting.** Every map's {codepoint, glyph} pairs go through
  `_kashi_pairs_finish`, which skips sorting when the pairs are already in
  order and otherwise heapsorts them. Heapsort runs in place, allocates
  nothing, and is O(n log n) on any input. The already-sorted check keeps
  the common case at its old cost: most tables, and every PCF walk, arrive
  in order.
- **Duplicates.** Of several pairs with one codepoint, the **highest glyph
  index** stays. This is the Linux console's rule. The kernel's
  `con_set_unimap` applies the pairs in the order it receives them, and
  `con_insert_unipair` overwrites the slot unconditionally
  (`drivers/tty/vt/consolemap.c`), so a table applied in glyph order keeps
  its last, highest glyph. Table order is glyph order for PSF and BDF; the
  text table follows the same rule by glyph index, whatever its line order.
- **Sequences.** Records are sorted lexicographically by codepoints, then
  shorter first, and `kashi_font_seq_glyph` binary-searches them. A
  sequence listed twice also keeps its highest glyph.

**A table always addresses the font.** A font that carries or is given a
Unicode table is codepoint-addressed by that table, even when the table
maps no single codepoint: it gets an empty map, so every codepoint misses.
Identity addressing (the codepoint as a raw glyph index) remains only for
fonts without any table, as ADR 0003 intended.

**A record is complete before anyone can see it.**
- Loaders build every map first and register the finished record.
- An allocation failure while building fails the load with
  `0 - KASHI_EFORMAT`.
- Attach builds the new maps aside and stores them only once the table has
  parsed. A failed attach leaves the record exactly as it was, so the 0.8.0
  F9 rollback is never needed.

**Files are read whole, sized to the file, and only if regular.**
- **Linux (x86_64 and aarch64).** `_kashi_read_file`:
  - opens the path `O_NONBLOCK | O_NOCTTY | O_CLOEXEC`;
  - fstats the open descriptor, so there is no stat/open race;
  - accepts only a regular file of 1 – cap bytes;
  - allocates exactly its size, reads to EOF, retries `-EINTR`, and
    refuses a file that grew while being read.
- **Other targets** (agnos, Windows, macOS). The stdlib reads up to cap + 1
  bytes, so a larger file is refused rather than cut.
- **Outcomes.** A file over the cap, a FIFO / tty / device / directory, or
  an unreadable path is `0 - KASHI_EINVAL`; an empty file is
  `0 - KASHI_EFORMAT`.
- **Attach.** The `attach_*_file` calls validate the font id (and kind)
  before they touch the path.

## Consequences

- **Positive.**
  - **Worst cases bounded.** The audit's worst cases load in 69 ms
    (PSF / binary table, 256 KiB), 26 ms (text table) and 51 ms (65,536
    BDF glyphs, either order).
  - **Typical loads close to unchanged.** A 512-glyph PSF2 loads in 63 µs,
    against 61 µs in 1.2.2 (nine interleaved rounds); BDF +7 % from its
    stricter parsing.
  - **Deterministic duplicates,** matching what the Linux console shows for
    the same font.
  - **Sequence lookups are O(log n):** 153 ns on 53,248 sequences, against
    ~0.8 ms.
  - **No half-built or cleared state** for a single-threaded caller.
  - **File loaders cannot hang** on a special file, cannot take a
    controlling terminal, and cost nothing for a missing path.
- **Negative.**
  - **Over-cap files now fail.** A file over the cap used to load
    (truncated); it now fails. Every documented cap is far above any real
    console font (the largest kbd font in the audit's survey had 669 table
    entries), so no real font is affected.
  - **Different glyphs for some duplicates.** A consumer that depended on
    the old, size-dependent duplicate winner sees the other glyph for that
    codepoint.
  - **No identity fallback for table-carrying fonts.** A font whose table
    maps nothing now draws nothing by codepoint, where it used to draw by
    glyph index. Its glyphs stay reachable through `kashi_rt_glyph_*`.
  - **A target-specific branch in the library face.** The Linux reader
    uses `sys_fstat` and Linux open flags inside `#ifdef`s; the other
    targets keep the plain bounded read and its blocking behaviour on
    special files.
- **Neutral.**
  - **Not thread-safe.** The library face still is not thread-safe
    (audit L12; documented in [`../api/README.md`](../api/README.md)).
    Building aside narrows, but does not close, the window in which a
    concurrent reader can see a torn {map, count} pair.

## Alternatives considered

- **Merge sort.** Also O(n log n) and stable, but it needs an n-sized
  scratch buffer from a bump allocator that never frees. Heapsort needs
  none, and the explicit tie-break on glyph index makes stability
  unnecessary.
- **"First listed wins" for duplicates.** Equally deterministic, but it
  disagrees with the Linux console. Linux is what a PSF font's author
  tested against.
- **Rejecting tables that map nothing.** That would make a sequence-only
  table unusable, and would turn a malformed-but-loadable font into a load
  failure. The empty map keeps the font loadable and honest.
- **stat() before open().** This leaves a race (the path can be swapped for
  a FIFO in between) and still blocks if the swap wins. Opening
  non-blocking and fstat-ing the descriptor closes the race.
- **Raising the caps instead of refusing.** The caps exist to bound memory
  for untrusted input; truncating silently was the bug, not the size.
