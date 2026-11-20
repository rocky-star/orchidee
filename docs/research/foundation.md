# Foundation: build from scratch or on a terminal library?

Research note for [ticket #2](https://github.com/rocky-star/orchidee/issues/2).
Evidence gathered 2026-11-20 from primary sources: upstream docs, source
code, man pages and licences. Candidate versions inspected: libtickit
0.4.6, notcurses 3.0.17 (README and man pages), libvterm 0.3.3,
termbox2 master (2026-11-12), PDCurses 3.9 plus PDCursesMod's VT port,
ncurses 6.6 man pages, and the `magiblot/tvision` reference copy in this
repo.

## The question, and how to judge it

The ticket asks whether to start from scratch or build on libtickit,
notcurses, libvterm, termbox2, PDCurses "or similar". It also says to
judge candidates against Orchidée's requirements, not against their own
advertised scope. So the useful unit of comparison is not "what is the
library for" but "how much of the required vertical does it already
supply, and how much of it fights us".

Requirements restated as testable properties:

| # | Requirement | What a passing implementation must show |
|---|-------------|------------------------------------------|
| R1 | Passive, loop-agnostic | No blocking read, sleep, or callback into host. Caller feeds bytes/times; compositor returns events and a flush plan. |
| R2 | Retained surface tree | Content lives in compositor-owned surfaces; overlap is resolved by the compositor, not by the host repainting on demand. |
| R3 | Damage tracking | Per-surface/per-region damage is accumulated and reported or consumed at flush; unchanged cells are not emitted. |
| R4 | terminfo-driven, with downgrade | Capabilities come from the terminfo entry for `TERM`, with a defined fallback for missing/lying entries and for terminals that cannot express a cell. |
| R5 | Semantic input events | Bytes decode to key/text/mouse/resize/paste events with modifiers and text; not raw escape sequences. |
| R6 | Small footprint, C ABI, deps below | C ABI (opaque handles, no C++/host types), dependencies only beneath the compositor, no global process state, no threads required. |

R1 and R2 together are the hard part, and they are the properties no
candidate advertises.

## Prior art: Turbo Vision's platform layer

Turbo Vision is Orchidée's model, so its platform layer is the most
relevant prior art. The `magiblot/tvision` port is the maintained
reference here.

What the layer looks like:

- A shadow screen of cells: `TScreen::screenBuffer` is a
  `TScreenCell` array, where a cell is character + colour attribute
  ([`include/tvision/system.h`](https://github.com/magiblot/tvision/blob/master/include/tvision/system.h),
  [`include/tvision/scrncell.h`](https://github.com/magiblot/tvision/blob/master/include/tvision/scrncell.h)).
- A backend interface: `DisplayAdapter` with `writeCell`,
  `setCaretPosition`, `clearScreen`, `flush`; `Platform` owns the
  display buffer, input sources and an `EventWaiter`
  ([`include/tvision/internal/platform.h`](https://github.com/magiblot/tvision/blob/master/include/tvision/internal/platform.h)).
- Damage in the modern port is a per-row byte range plus a
  previous-frame
  cell buffer: `DisplayBuffer` holds `buffer`, `flushBuffer` and
  `rowDamage`, and `flushScreenAlgorithm` emits only rows/cells whose
  range is dirty and whose cell differs from `flushBuffer`
  ([`include/tvision/internal/dispbuff.h`](https://github.com/magiblot/tvision/blob/master/include/tvision/internal/dispbuff.h),
  [`source/platform/dispbuff.cpp`](https://github.com/magiblot/tvision/blob/master/source/platform/dispbuff.cpp)).
- Line drawing goes through `TDrawBuffer` and
  `TView::writeBuf`/`writeLine`;
  writes are clipped to the view and then to the region not obscured by
  higher siblings, including shadow cells (`TVWrite::L20` walks the
  sibling list from the top)
  ([`include/tvision/drawbuf.h`](https://github.com/magiblot/tvision/blob/master/include/tvision/drawbuf.h),
  [`source/tvision/tvwrite.cpp`](https://github.com/magiblot/tvision/blob/master/source/tvision/tvwrite.cpp)).
  `TGroup::draw` simply visits children in z-order and lets each one
  draw ([`source/tvision/tgroup.cpp`](https://github.com/magiblot/tvision/blob/master/source/tvision/tgroup.cpp)).
- Overlap is therefore solved by clipping at write time, directly into
  one shared screen buffer. There is no per-surface offscreen content
  and no per-surface damage: a view that must repaint draws again over
  the buffer.
- Capability handling in this port does **not** use terminfo. It
  hardcodes an ANSI writer and guesses from environment: `COLORTERM`,
  a tentative colour count (from ncurses when built with it), `TERM ==
  "xterm"` treated as 16-colour, and quirks such as bold-is-bright,
  blink-is-bright, no-italic on the Linux console
  ([`source/platform/ansiwrit.cpp`](https://github.com/magiblot/tvision/blob/master/source/platform/ansiwrit.cpp)).
  This is workable but shows the cost: a growing quirk list and a
  separate ncurses/win32/Linux-console adapter behind `DisplayAdapter`.
- Input decoding is a terminal byte parser producing 16-bit
  BIOS-style key codes plus modifier masks (`TKey`, `kbShift`, ...),
  with terminal-specific heuristics for function keys and mouse
  ([`source/platform/termio.cpp`](https://github.com/magiblot/tvision/blob/master/source/platform/termio.cpp),
  [`source/platform/ncursinp.cpp`](https://github.com/magiblot/tvision/blob/master/source/platform/ncursinp.cpp)).
- The loop is owned above this layer: `TEventQueue::waitForEvents`
  uses `select`/`poll`, but `TProgram::getEvent` blocks on it and
  `TProgram::run` drives the application
  ([`source/platform/events.cpp`](https://github.com/magiblot/tvision/blob/master/source/platform/events.cpp),
  [`source/tvision/tprogram.cpp`](https://github.com/magiblot/tvision/blob/master/source/tvision/tprogram.cpp)).

Lessons for Orchidée:

1. The seam (`DisplayAdapter` + shadow cell buffer + dirty ranges) is
   right, but it is a C++ virtual interface and assumes a host loop.
   Orchidée needs the same seam expressed as a C ABI with an explicit
   "here are bytes, here is what to write" contract.
2. Clipping at write time is not retained composition. It works, but it
   makes repaint the host's problem and cannot answer "what changed?"
   without diffing whole frames. Orchidée's R2/R3 ask for the opposite.
3. Skipping terminfo is a known maintenance tax. Orchidée's R4 is the
   correction, not an over-engineering.

## Candidates

### Scorecard

Legend: +++ strong fit, ++ usable / adapt, + partial, - absent or
counterproductive. "Fit" judges the required vertical only.

| Candidate | POSIX tty | R1 passivity | R2 retained tree | R3 damage | R4 terminfo + downgrade | R5 semantic input | Footprint / deps | Licence (MPL-2.0 ok?) | Maintenance | Forkability | Fit |
|-----------|-----------|--------------|------------------|-----------|--------------------------|-------------------|------------------|----------------------|-------------|-------------|-----|
| libtickit 0.4.6 | yes | + (term layer passive; Tickit object owns a loop) | ++ (window tree, stacking) | ++ (expose rectangles) | ++ (terminfo driver + xterm fallback) | ++ (key/text/mouse with mods) | small; libtermkey dep | MIT, yes | active (0.4.6, 2026-02) | easy | closest, but not passive at the tree layer |
| notcurses 3.0.17 | yes | - (blocking exclusive render, init interrogates terminal, signal handlers) | ++ (plane pile, z-order, alpha) | ++ (internal lastframe damage map) | ++ (terminfo + terminal queries) | +++ (rich, kitty protocol) | large; terminfo 6.1+, libunistring, pthreads | Apache-2.0, yes | very active | impractical | no: owns the process terminal and the loop model |
| libvterm 0.3.3 | n/a (no I/O) | +++ (byte-buffer in, callbacks out) | - (grid, not a surface tree) | ++ (merged damage callbacks) | - (no output generation) | - (parses terminal *output*, not keys) | tiny; no deps | MIT, yes | slow; Neovim mirror archived | easy | component/prior art only |
| termbox2 | yes | + (peek/poll blocks; fds exported) | - (one flat buffer) | + (front/back cell diff, no regions) | ++ (terminfo if present, built-in caps otherwise) | + (key/ch/mod/mouse/resize) | tiny; libc only, single header | MIT, yes | active | easy | excellent donor code, not an architecture |
| PDCurses 3.9 | no (mainline: DOS/OS2/Windows/X11/SDL) | - (blocking `getch`) | + (curses windows) | + (line-touch + `doupdate`) | - (no terminfo; VT port hardcodes sequences) | + (curses keys) | medium | public domain, yes | maintained | easy | no: wrong POSIX story |
| ncurses 6.6 + panels | yes | - (global SCREEN, blocking input) | + (windows; overlap needs panels layer) | + (touched lines, virtual screen) | +++ (terminfo is its database) | + (curses keys, `get_wch`) | large | X11-style, yes | active | impractical | prior art and terminfo donor, not a foundation |

### libtickit

The closest architectural fit. Its own description is a window tree
subdividing the terminal, and it explicitly supports overlapping
windows, synchronously or asynchronously via file descriptors, or
abstractly via byte buffers
([project page](https://www.leonerd.org.uk/code/libtickit/),
[`tickit(7)`](https://www.leonerd.org.uk/code/libtickit/doc/tickit.7.html)).

- Windows are a retained tree with stacking order, visibility, focus,
  and clipping to ancestors; "drawing operations on a window only take
  effect if there are no higher siblings that would obscure it"
  ([`tickit_window(7)`](https://www.leonerd.org.uk/code/libtickit/doc/tickit_window.7.html)).
- Damage is exposure tracking: "newly-exposed areas of windows are
  tracked by the root window, ready to be rendered by expose events",
  with non-overlapping `TICKIT_WINDOW_ON_EXPOSE` segments and a render
  buffer pre-configured with clip and hole regions. That is *app
  repaints on damage*, not *composition of retained content*. The gap to
  R2/R3 is real but small: the tree, stacking and region bookkeeping are
  already there.
- `TickitRenderBuffer` has skip cells, clipping, masks, translation and
  a save stack, designed "to easily support applications that divide the
  screen area into several possibly-overlapping regions"
  ([`tickit_renderbuffer(7)`](https://www.leonerd.org.uk/code/libtickit/doc/tickit_renderbuffer.7.html)).
- Terminal layer is genuinely low-level and loop-friendly:
  `TickitTerm` has synchronous `tickit_term_input_wait_msec`, or
  asynchronous `tickit_term_input_readable` for an external loop; output
  buffers are explicitly flushed by the caller
  ([`tickit_term(7)`](https://www.leonerd.org.uk/code/libtickit/doc/tickit_term.7.html)).
- Capabilities: a terminfo driver plus an xterm fallback driver
  (`src/termdriver-ti.c`, `src/termdriver-xterm.c`), with documented
  fallbacks where terminfo is incomplete
  ([source tarball](https://www.leonerd.org.uk/code/libtickit/libtickit-0.4.6.tar.gz),
  [termdriver-ti.c](https://github.com/leonerd/libtickit/blob/master/src/termdriver-ti.c)).
- Input: `TICKIT_TERM_ON_KEY` distinguishes `TICKIT_KEYEV_KEY` (named
  keys, modifiers) from `TICKIT_KEYEV_TEXT` (UTF-8 text); mouse events
  include press, drag, release and wheel; events are relative to a
  window for window events. Input parsing is delegated to the external
  libtermkey (`Makefile` uses `pkg-config termkey`).
- Loop model: a `Tickit` object "can act as a containing event loop"
  (`tickit_run`), but there is a pluggable `TickitEventHooks` interface.
  That interface is explicitly unstable: its header says the contents
  "should be considered entirely experimental" with "no API or ABI
  stability guarantees" (`include/tickit-evloop.h`). So passivity is
  possible but not a contract.
- Health and fitness: MIT licence (`LICENSE`); active, 0.4.6 released
  2026-02-12 with 0.4.5 in 2025 (`CHANGES`); small C library. Costs:
  a second dependency (libtermkey), a C API whose evloop hook is
  unstable, and a window/expose model that would still need a retained
  content layer and a passive C ABI in front of it.

Verdict: the most credible "build on" option. Not adoptable as-is
because the properties Orchidée treats as invariants (passive, retained,
stable C ABI) are the ones it treats as best-effort or leaves to the
host. Worth borrowing its driver split, its window/damage bookkeeping,
and possibly its input path if accepting libtermkey.

### notcurses

Technically the most capable and the least suitable.

- Renders a pile of planes (flat z-order, not a hierarchical tree) to a
  cell matrix, then rasterises with terminfo-derived sequences
  ([`notcurses_render(3)`](https://notcurses.com/notcurses_render.3.html)).
  It has internal damage: the rasteriser keeps a `lastframe` cell matrix
  and a damage map (`src/lib/render.c` comment "copies that area of the
  lastframe (damage map)").
- But the model is a process-wide terminal owner: init opens and
  interrogates the terminal, installs signal handlers (including fatal
  signals), and rasterisation is "an exclusive blocking call"; callers
  "must not call any other functions modifying the same pile" while it
  runs ([`notcurses_render(3)`](https://notcurses.com/notcurses_render.3.html),
  [README](https://github.com/dankamongmen/notcurses#readme)).
- Input is strong (Unicode codepoints, modifiers, kitty keyboard
  protocol, `nckey` synthesis from terminfo input capabilities) and
  offers `notcurses_inputready_fd` for `poll(2)`, but `notcurses_get`
  still owns the read path, and init can block on terminal replies
  ([`notcurses_input(3)`](https://notcurses.com/notcurses_input.3.html),
  README: "Notcurses queries the terminal on startup").
- Footprint: terminfo 6.1+, libunistring, pthreads, optional FFmpeg /
  OpenImageIO stack; Apache-2.0. Concurrency is a design goal, not an
  absence of threads.

Verdict: fails R1 and R6 structurally; adopting it would mean wrapping
a terminal-owning runtime and hiding its render loop, which is more work
than writing the compositor core. Keep as a reference for capability
probing and input breadth.

### libvterm

An abstract VT220/xterm/ECMA-48 emulator: "it invokes callback function
pointers that its embedding program should provide"; it avoids malloc in
the running state and does no I/O of its own
([project page](https://www.leonerd.org.uk/code/libvterm/)).
`vterm_screen_set_callbacks` provides `damage`, `moverect`, `scrollrect`
and friends, with `vterm_screen_set_damage_merge` and
`vterm_screen_flush_damage` to coalesce regions
([`include/vterm.h`](https://github.com/neovim/libvterm/blob/master/include/vterm.h)).

But it solves the opposite direction: it parses bytes an *application*
writes to a terminal and maintains a screen grid. Orchidée needs to
drive a terminal and parse *user* input. Its damage model (rectangle
callback + merge policy) is good prior art for R3; its parser is not the
input decoder R5 needs. MIT; tiny; no dependencies. Maintenance is slow
(upstream 0.3.3; the Neovim GitHub mirror is archived, with the code
vendored into Neovim).

Verdict: borrow the damage-merge idea if usable; do not build on it.

### termbox2

The best donor code for the POSIX adapter, and the clearest example of
why "a cell buffer plus a diff" is not a compositor.

- One header, ~4,200 lines, libc only, MIT; global singleton state
  (`struct tb_global`), one terminal per process
  ([README](https://github.com/termbox/termbox2#readme),
  [`termbox2.h`](https://github.com/termbox/termbox2/blob/master/termbox2.h)).
- Capabilities: parses a terminfo entry when one is found (`TERMINFO`,
  `~/.terminfo`, then the standard directories), otherwise uses built-in
  tables for popular terminals (xterm, linux, screen, tmux) — a working
  downgrade story, though only for the capabilities termbox2 models.
- Rendering: front/back cell buffers; `tb_present` walks the grid and
  emits only differing cells; `tb_invalidate` forces a full repaint.
  Damage is per-cell, full-screen, with no rectangles and no surfaces.
  Cells carry EGC support only with `TB_OPT_EGC` (opt-in, "very coarse
  approximation").
- Input: `tb_event` with key/ch/mods, mouse and resize; `tb_peek_event`
  with a timeout, `tb_poll_event` blocking; `tb_get_fds` returns tty and
  resize fds for external `poll`, but callers must still call
  `tb_poll_event`/`tb_peek_event` to consume input. SIGWINCH is handled
  with a `sigaction` handler and a self-pipe.
- No window tree, no overlap, no clipping regions, no C ABI boundary
  (the header *is* the library), no retained content.

Verdict: vendor or reimplement its tty setup, capability fallback and
input parser; do not mistake it for a foundation.

### PDCurses

Public domain, and explicitly a portability project: DOS, OS/2,
Windows console, X11 and SDL
([README](https://github.com/wmcbrine/PDCurses#readme)). Mainline has no
native POSIX terminal port. The PDCursesMod VT port exists but is
honest about itself: it uses "a mix of xterm, VT-100, VT-200, and ANSI
escape sequences", tested in a short list of terminals, assumes 256
colours, and its README says the sequences "may, or may not, work on
your terminal"
([PDCursesMod VT README](https://github.com/bill-gray/pdcursesmod/blob/master/vt/README.md)).
No terminfo. The curses window model plus its panel library (PDCurses
has one since 2.2, per
[`panel(3x)`](https://invisible-island.net/ncurses/man/panel.3x.html))
is prior art, not a fit.

Verdict: exclude.

### ncurses + panels

The reference implementation of the terminfo model, and the best prior
art for terminal-side diffing: windows are copied to a virtual screen
with `wnoutrefresh`, and `doupdate` compares virtual and physical
screens, updating only touched lines
([`curs_refresh(3x)`](https://invisible-island.net/ncurses/man/curs_refresh.3x.html)).
Overlapping windows are a separate layer: panels add depth to curses
windows and hide/show the portions beneath them
([`panel(3x)`](https://invisible-island.net/ncurses/man/panel.3x.html)).
Terminfo itself is a compiled capability database with boolean, numeric
and string capabilities, parameterised strings and padding
([`terminfo(5)`](https://man7.org/linux/man-pages/man5/terminfo.5.html)).

It fails Orchidée's structural requirements: global `SCREEN` state,
blocking `getch`, signals, no passive API, no stable embeddable C ABI
boundary, and a window model where overlap needs another library. The
ncurses licence is X11-style, not a blocker
([licence text](https://invisible-island.net/ncurses/ncurses-license.html)).

Verdict: do not adopt as a foundation; do link the terminfo library
(`libtinfo`, the part the rest of the world already depends on) for R4.

## Cross-cutting findings

1. **Nobody supplies R1 + R2 + R3 together.** libtickit is closest on
   the tree; libvterm on merged damage; termbox2 on tty setup and
   input. The passive, retained, damage-reporting compositor with a C
   ABI is the part that must be built.
2. **"Damage" means two different things in the field.** Curses,
   Turbo Vision and notcurses compare a virtual/previous frame at flush
   time (cheap to adopt, O(screen) work per frame). libtickit and X11
   expose damage regions upward (cheap to compute, host must repaint).
   Orchidée wants damage that is accumulated by the tree and consumed by
   the compositor itself; that is a third design, and only the region
   bookkeeping can be borrowed.
3. **termbox2 is the only small candidate with a real downgrade path**
   (terminfo when available, built-in tables otherwise, plus explicit
   "unsupported terminal" errors). notcurses and ncurses rely on
   terminfo being correct; libtickit adds an xterm fallback driver;
   Turbo
   Vision's modern port skips terminfo entirely and pays in quirks.
4. **Input semantics are table stakes at the escape-sequence layer.**
   libtickit, notcurses and termbox2 all expose decoded keys with
   modifiers; only notcurses covers the kitty keyboard protocol. Turbo
   Vision's BIOS-scan-code key model is the historical cost of not
   having Unicode text events.
5. **Each candidate's loop model decides its fate.** libtickit can be
   driven from outside but its stable entry points still want
   `tickit_run`; notcurses renders exclusively and blocks;
   ncurses/PDCurses/termbox2 own the process terminal. Only libvterm is
   fully passive, because it does no I/O at all.
6. **Licences are not the constraint.** MIT (libtickit, libvterm,
   termbox2), Apache-2.0 (notcurses), public domain (PDCurses) and
   X11-style (ncurses) all permit combination with an MPL-2.0 work.
   Forkability is therefore decided by code size and coupling, not by
   law: termbox2 and libvterm are trivially forkable, libtickit
   moderately, notcurses and ncurses not practically.

## What this changes for Orchidée

- **Build** the compositor core: surface tree, overlap and damage
  accumulation, composition into a cell buffer, the flush planner and
  the C ABI. No candidate has this combination, and wrapping one would
  keep its loop or its state model.
- **Adopt** terminfo as the capability source, linking `libtinfo` (or a
  small vendored terminfo reader) beneath the compositor, with an
  explicit fallback table for when the entry is missing or the terminal
  cannot represent a cell. Do not let ncurses' screen/window layers in.
- **Borrow** two things, kept behind a replaceable adapter:
  termbox2's tty setup, input parser and capability-fallback tables
  (MIT, single header, small enough to vendor and patch), and
  libtickit's split between a terminfo driver and an xterm fallback
  driver, plus its window exposing/stacking bookkeeping as the shape of
  the damage API.
- **Do not adopt** notcurses, PDCurses, or ncurses as the foundation.
  They fail R1/R6 before their features matter.
- **Validate** against the requirements as tests: the core API accepts
  bytes and a timestamp, returns events and a flush plan, and never
  blocks; a damage-only repaint produces a byte-identical stream to a
  full repaint on a plain test terminal; a missing terminfo entry
  degrades to the fallback table rather than failing.
- **Revisit** libtickit as a dependency (not a wrapper) if the surface
  tree work grows beyond the plan; its C API is MIT and active, and its
  window/damage model is the closest prior art available.
