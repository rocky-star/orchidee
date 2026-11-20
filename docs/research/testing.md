# Testing the Compositor and its Python layer

Research note for
[ticket #5](https://github.com/rocky-star/orchidee/issues/5) It answers
"how will the Compositor and its Python layer be tested?" with a
recommended architecture, the primary sources behind each choice, and
the maintenance cost of each piece. It records facts and options; it
does not decide product behaviour.

Evidence base: sources cited inline, plus a local probe of this
machine's terminfo database (`infocmp`, `tput`), the test tooling
installed here, and the untracked reference copy of magiblot/tvision in
`tvision/`. Statements that come only from the probe are marked
"observed".

## 1. Testability is an interface property

Most answers below depend on four C ABI decisions. Treat them as
requirements, not implementation details.

1. **The core writes to an injected file descriptor, not to stdout.**
   [openpty(3)](https://man7.org/linux/man-pages/man3/openpty.3.html)
   returns a master/slave pair; if the core takes the slave fd, a test
   reads the master fd and never needs a controlling terminal. If the
   core writes `STDOUT_FILENO` and calls `isatty()`, the same tests
   become process-level fixtures.
2. **Terminal identity is passed in, not discovered.** `TERM`,
   `TERMINFO`, `LINES`/`COLUMNS`, and locale are inputs. Pass a resolved
   profile (capability set plus size) into the core; keep `getenv()` in
   the CLI wrapper.
   [terminfo(5)](https://man7.org/linux/man-pages/man5/terminfo.5.html)
   documents the precedence tests must be able to pin: `TERMINFO` is
   checked first, then `TERMINFO_DIRS`, then `$HOME/.terminfo` and the
   system database.
3. **The writer emits a capability plan, not just bytes.** The reference
   tvision keeps an `AnsiScreenWriter` (buffer plus ANSI output)
   separable from screen state
   (`tvision/include/tvision/internal/ansiwrit.h`). A comparable seam
   lets tests assert "which capabilities were used, with which
   parameters" without parsing escape output.
4. **A frame has an observable boundary.** `present()`/`flush()` should
   report the number of bytes written (or invoke a sink). Without that,
   every capture needs a timeout, and timeouts are the main source of CI
   flakiness in terminal tests.

The Compositor is passive and loop-agnostic, which is already the right
shape: no hidden threads, no internal clock. Add a clock parameter where
the Python layer debounces or coalesces, so those tests need no sleeps.

## 2. Layers

| Layer | Runs | Tool | What it asserts | Churn |
|---|---|---|---|---|
| L0 C++ unit | in-process | GoogleTest or doctest, CTest | surface tree, damage geometry, cell ops, colour mapping | low |
| L1 decoder unit | in-process | same | bytes to semantic events, including incomplete sequences | low |
| L1f fuzz | nightly | libFuzzer + ASan/UBSan | no crash, bounded buffers, invariants from section 7 | low |
| L2 writer plan | in-process | same | capability use subset of profile; op sequence | medium |
| L3 byte golden | in-process sink | custom harness | exact output for a pinned profile | high |
| L4 grid golden | in-process | libvterm oracle | cells, attributes, cursor across the matrix | medium |
| L5 fake transport | in-process | asyncio + socketpair | driver loop, cancellation, backpressure | low |
| L6 pty integration | Linux only | `openpty`/`forkpty` or Python `pty` | end-to-end bytes, termios interaction, SIGWINCH | medium |
| L7 capability matrix | in-process | generated from terminfo | downgrade legality, no unsupported capabilities | medium |
| L8 manual conformance | interactive | tack, vttest, esctest2 | terminal description / emulator correctness | n/a |

The pyramid should be steep. The pull-request suite is L0-L5 and L7,
entirely in-process. L6 is a handful of Linux-only cases in its own job.
L8 never gates a merge.

## 3. Driving the core from a pseudo-terminal

Use `openpty()` when a test needs real terminal semantics: the `winp`
argument sets the window size on the slave, `termp` sets its termios
([openpty(3)](https://man7.org/linux/man-pages/man3/openpty.3.html)).
`forkpty()` combines openpty, fork and `login_tty()` when the core runs
as a child process. Python exposes the same primitives as
`pty.openpty()`, `pty.fork()` and `pty.spawn()`
([pty module](https://docs.python.org/3/library/pty.html)).

The critical determinism point: a pty is not a byte pipe. The line
discipline applies output post-processing (`ONLCR` turns `\n` into
`\r\n`), input translation (`ICRNL`), echo (`ECHO`), and signal
generation for `^C`/`^Z`
([termios(3)](https://man7.org/linux/man-pages/man3/termios.3.html)).
Unless the slave is put in raw mode (`cfmakeraw()` or explicit termios),
the bytes read from the master are not the bytes the core wrote, and
input injected at the master is transformed before the core sees it. A
test that compares bytes without controlling termios is testing the line
discipline, not the Compositor.

Read-side caveats:

- A master read returns whatever is available; there are no frame
  boundaries. "Read until idle" is a timeout in disguise.
- If the core queries the terminal (DSR/CPR, DA, colour queries), the
  test must script the replies. `esctest2` does exactly this class of
  inference with `DECRQCRA` on a real terminal, and its author notes the
  limits: it cannot test interactive features and has gaps (NRCS, C1,
  UTF-8) ([vttest](https://invisible-island.net/vttest/)).
- Resizing is `TIOCSWINSZ` on the slave/master; the kernel then delivers
  `SIGWINCH`
  ([pty(7)](https://man7.org/linux/man-pages/man7/pty.7.html)). A
  passive core should not depend on the signal: the driver should call
  an explicit `resize(cols, rows)`.

Recommendation: the pty is a thin end-to-end layer. Byte-exact and
cell-exact assertions run in-process through an fd pair (`socketpair()`
or a sink callback), where ordering is defined by the test, not the
scheduler. `script(1)` is useful for manual reproduction of a CI
failure, not for assertions
([script(1)](https://man7.org/linux/man-pages/man1/script.1.html)).

## 4. Deterministic capture

Capture at the write boundary. In-process, the cleanest seam is a sink:
the core writes to a callback or fd supplied by the test, and the test
records a sequence of `(frame, bytes)` records. A pty is only needed for
the cases in section 3.

Pin every input that can change output:

- `TERM`, `TERMINFO`, `TERMINFO_DIRS`, `LC_ALL`, `LANG`, `COLUMNS`,
  `LINES`, `COLORTERM`. Tests should set all of them explicitly and fail
  if the core reads others.
- Vendor terminfo. Check in a small `terminfo.src` subset and compile it
  with `tic -x` into the build directory, then set `TERMINFO` to that
  directory
  ([tic(1)](https://man7.org/linux/man-pages/man1/tic.1.html)). The
  system database differs between distributions and ncurses versions;
  compiled entries also change format between ncurses releases.
- Record provenance in the golden: `TERM`, `infocmp -1` checksum, core
  version, profile name. A golden without provenance cannot be reviewed.
- Forbid: hash-order-dependent emission, timestamps, cursor blink,
  animations, terminal queries outside a scripted negotiation step, and
  any output that depends on how many `write(2)` calls the OS accepted.

Store byte goldens as escaped text (`\x1b[31m...`), one frame per line,
so diffs are reviewable in a pull request. Do not store raw binary.

## 5. Golden frames: cell grid versus raw escapes

| Question | Raw escape golden | Cell-grid golden |
|---|---|---|
| Sensitive to TERM/terminfo | yes, strongly | no, if the oracle is correct |
| Sensitive to writer heuristics (cursor moves, SGR ordering) | yes | no |
| Catches damage/overlap bugs | only via sequences | needs multi-frame scenarios |
| Catches writer regressions and inefficiency | yes | poorly |
| Reviewable diff | noisy | readable |
| Churn from terminfo updates | high | low |

Use both, at different layers, never as substitutes:

- **Byte golden (L3), one to three pinned profiles.** Assert the
  complete byte stream for a scripted frame sequence. This is the only
  layer that catches a writer that emits a capability the profile lacks,
  or that regresses to a slower but correct path. Because damage
  rendering makes output history-dependent, scenarios must include:
  first full repaint, incremental damage, overlapping windows, scroll,
  resize, and a no-op frame. Name goldens by scenario and profile.
- **Grid golden (L4), all profiles.** Normalise to cells: characters,
  attributes (as a structured tuple, not a raw SGR string), cursor
  position, cursor visibility. Assert equality across every profile.
  This is the layer that answers "is the same logical frame shown on
  vt100 and xterm-direct?".

Never compare raw bytes across `TERM` values: the bytes are supposed to
differ. Assert instead that the *capability-use log* is identical modulo
capability substitution, and that the grid is identical. Likewise, never
"stabilise" a byte golden by stripping trailing `SGR 0`, trailing
spaces, or clear sequences: those are exactly the assertions.

Updating goldens should be a flag (`--regen`/`--force-regen`), not a
manual edit, with the diff reviewed and committed:
[pytest-regressions](https://pytest-regressions.readthedocs.io/en/latest/overview.html)
documents that workflow for Python; the C++ harness should mirror it.
Per-profile directories keep matrix growth mechanical:
`tests/golden/bytes/<profile>/<scenario>.txt` and
`tests/golden/grid/<scenario>/<case>.txt`.

## 6. A VT emulator as oracle

**libvterm** is the strongest candidate. It is a C99 library
implementing "a VT220 or xterm-like terminal emulator" that calls back
into the embedder and avoids `malloc()` in normal operation
([libvterm](https://www.leonerd.org.uk/code/libvterm/)). Its screen API
exposes cell contents, attributes, cursor and damage, which is what a
grid golden needs. Its own regression harness is a small stdin-driven
protocol: `PUSH <hex-bytes>`, `WANTSCREEN`, `?screen_chars`,
`?screen_cell`, `?screen_text`, `DAMAGEFLUSH`, with expected output
compared as text
([t/harness.c](https://github.com/neovim/libvterm/blob/master/t/harness.c)).
That is the same shape as the L3/L4 harness proposed here, which is
useful evidence that the design scales.

Three ways to use it: link it into a C++ test binary (fastest, best
failure detail); wrap it with `ctypes`/`cffi` for Python tests; or run
the harness binary as a subprocess (slowest, and it introduces a second
parser). Prefer linking. Pin the libvterm version in the dev environment
and record it in grid golden provenance.

**pyte** is a pure-Python "in memory VTXXX-compatible terminal emulator"
([pyte](https://pyte.readthedocs.io/en/latest/)). It is convenient for
Python-level snapshots and has no build dependency, but its own history
is a warning: it "started as a fork of vt102, which is an incomplete
pure Python implementation of VT100". Modern sequences (truecolor, kitty
keyboard, bracketed paste edge cases) and attribute fidelity are weaker
than libvterm's. Use it as a convenience oracle for coarse grid checks,
never as the only oracle, and keep an explicit list of sequences it does
not model.

**esctest2** is an automated VT100-VT525 test suite that infers screen
contents with `DECRQCRA`
([esctest2](https://github.com/ThomasDickey/esctest2), described in the
vttest page). Orchidée is an application, not an emulator, so this tests
the test oracle rather than the product. It is worth running once when
evaluating an oracle, not as a product CI gate.

Oracle risk is symmetric: a correct Compositor can be marked wrong by an
oracle bug. Mitigate by keeping a small set of hand-reviewed goldens
(independent of the oracle) and by upgrading the oracle deliberately, in
a commit that also reviews the golden diff.

## 7. Fuzzing the input decoder

The input decoder is the highest-risk parser in the project: partial
escape sequences, UTF-8, mouse, paste, and terminal responses arrive in
arbitrary fragments.

Build a libFuzzer target with `-fsanitize=fuzzer,address,undefined`; the
LLVM documentation covers the entry point, corpus handling, `-merge=1`
minimisation, and the rule that a fuzz target "must not exit() on any
input" and "must be as deterministic as possible"
([libFuzzer](https://llvm.org/docs/LibFuzzer.html)). Assert the
properties inside the target rather than relying on sanitizers alone:

- all bytes are consumed, or at most a documented maximum is left
  pending;
- every complete sequence produces an event or an explicit ignore, never
  a silent drop;
- the pending buffer never exceeds its documented cap (no unbounded
  growth);
- no invalid rune is emitted; an incomplete UTF-8 tail is retained, not
  replaced by U+FFFD prematurely;
- mouse, paste, focus and kitty-keyboard modes stay in range;
- resetting the decoder produces identical output for identical input
  (no cross-input state).

Seed the corpus with: escape strings harvested from the vendored
terminfo entries, the relevant xterm control sequences
([xterm ctlseqs](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)),
truncations at every byte of each seed, and every historical crash. Pass
a libFuzzer dictionary (`-dict`) containing the capability strings; it
makes structured sequences far easier to reach. In CI, run the corpus
files as a regression test (libFuzzer re-runs inputs passed as files)
and run real fuzzing nightly with `-max_total_time`, `-jobs`, and a
cached corpus. Keep `-rss_limit_mb` and `-timeout` at defaults or
tighter: they are part of the assertion.

Python-side fuzzing: if any input parsing lives in Python (key-name
grammar, mouse SGR decoding, paste framing), use
[Atheris](https://github.com/google/atheris), which instruments Python
bytecode and supports custom mutators for structured input. If all
parsing stays in the C decoder, Atheris adds a dependency without
coverage; skip it.

Hypothesis stateful testing is the better complement at the driver
level: model the driver modes as a state machine (raw/cbreak,
application cursor keys, mouse tracking off/on, bracketed paste) and
assert model-vs-driver agreement after every rule
([Hypothesis stateful](https://hypothesis.readthedocs.io/en/latest/stateful.html)).
This catches mode-transition bugs that byte-level fuzzing rarely
reaches.

## 8. Capability matrix

The matrix is data, generated from terminfo, not a hand-written list. A
local probe of this machine gives the shape (observed with `tput -T` and
`infocmp -x`; `-1` means the capability is absent):

| `TERM` | `colors` | `pairs` | `ncv` | `RGB` | Notes |
|---|---|---|---|---|---|
| `xterm` | 8 | 64 | - | no | ANSI colours, SGR 30-37 |
| `xterm-256color` | 256 | 65536 | - | no | SGR 38;5 |
| `xterm-direct` | 16777216 | 65536 | - | yes | SGR 38;2, extended cap |
| `screen` | 8 | 64 | - | no | multiplexer, `smcup` matters |
| `screen-256color` | 256 | 65536 | - | no | |
| `tmux-256color` | 256 | 65536 | - | no | |
| `alacritty` | 256 | 65536 | - | no | |
| `vt100` | - | - | - | no | no colour at all |
| `ansi` | 8 | 64 | 3 | no | `ncv` forbids attrs with colour |
| `linux` | 8 | 64 | 18 | no | console, `ncv` very restrictive |
| `dumb` | - | - | - | no | no capabilities |

`ncv` is the bitmask of attributes that cannot be combined with colour
and `ma` the maximum number of combined attributes
([terminfo(5)](https://man7.org/linux/man-pages/man5/terminfo.5.html));
both must feed the downgrade logic and its tests. Truecolor has no
X/Open terminfo primitive: ncurses' extended entries add an `RGB`
boolean and `colors#16777216` (`xterm-direct`, observed; see the ncurses
[terminfo.src](https://invisible-island.net/ncurses/ncurses.html#download_database)).
`COLORTERM=truecolor` is a convention, not a capability; do not make
tests depend on it. Pass an explicit profile instead.

What this layer asserts:

- every capability used is present in the profile (check the L2 use log,
  not the bytes);
- downgrade is legal and deterministic: for each requested colour, the
  chosen SGR is representable in the target depth, using a documented
  metric (e.g. nearest in a stated palette space), with table-driven
  cases covering truecolor to 256 to 8 to monochrome;
- attributes respect `ncv`/`ma` (e.g. bold plus colour on `ansi`);
- cursor addressing degrades or fails explicitly when `cup` is absent,
  and `smcup`/`rmcup` are never assumed;
- `dumb` behaves as documented (minimal output or a clear unsupported
  error), deterministically;
- a user override such as "8 colours on a 256-capable terminal" goes
  through the same downgrade path, because that is what happens inside
  `TERM=xterm` on modern emulators.

Manual tools validate the *description*, not our code: `tack` tests
terminfo entries interactively, `vttest` tests emulator conformance
(including colour erase and BCE behaviour), and the vttest author states
that he relies on both when testing descriptions
([vttest](https://invisible-island.net/vttest/),
[tack](https://invisible-island.net/ncurses/tack.html)). Use them when
adding a new `TERM` profile, not as CI gates.

## 9. The Python async driver without a terminal

The driver should be composed from three injected seams: a transport
(`read`, `write`, `close`, optional `fileno`), a clock (monotonic time,
`call_later`, `sleep`), and a resolved capability profile. With those,
no test needs a real terminal.

- **Pure decode/dispatch tests.** Feed recorded byte streams (the L3
  corpus) into the Python event layer and assert exact semantic events.
  No I/O, no loop, no timing.
- **Loop tests over `socketpair()`.** `socket.socketpair()` gives a
  full-duplex fd pair; the driver registers `loop.add_reader(fd)` (Unix
  only) and the test writes to the peer and awaits an event
  ([asyncio loop.add_reader](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.add_reader)).
  Assert callback order, that frames are presented once, that closing
  the peer produces a clean shutdown, and that a partial write is
  completed. Use `asyncio.wait_for` only as a hang detector, never as a
  synchroniser.
- **Fake clock.** Debounce, coalescing and redraw-rate limiting are
  tested by advancing the injected clock and running due callbacks; no
  sleeps.
- **Scripted responder.** If the driver negotiates (CPR, DA, OSC 10/11
  colour queries), a fake peer answers from a script, making the
  negotiation path deterministic. In production the same interface reads
  from the real fd.
- **Fake Compositor object.** Widget, layout and event-dispatch tests
  replace the ctypes/cffi boundary with a Python fake and assert calls
  (present count, damage rectangles). Keep a small set of contract tests
  against the real shared library so the fake cannot drift.
- **One real pty test per feature.** Run the driver against
  `pty.openpty()` with raw termios to catch `isatty`, termios, SIGWINCH
  and EOF/EIO mistakes. Keep this count small.

Failure modes worth explicit unit tests: `EAGAIN`/partial writes on a
non-blocking fd, cancellation while a read is pending, double close,
`EIO` when the peer disappears, and shutdown ordering (stop reader
before closing, or a late callback writes to a closed fd).

## 10. CI in containers

Runner image: Linux, clang (for libFuzzer and sanitizers), cmake/ninja,
python3, ncurses development tools (for `tic`), and a pinned libvterm
built from source or a pinned package.

Jobs:

1. `linux-gcc-unit`: build, L0/L1/L2/L3/L7, CTest, Python unit tests.
2. `linux-clang-sanitizers`: ASan + UBSan build, same suite; any
   sanitizer report fails the job.
3. `pty-integration`: Linux-only, L6, runs after the unit job. Non-root.
4. `python`: pytest with pinned dependencies, L5 plus contract tests.
5. `fuzz-nightly`: libFuzzer with cached corpus, time-boxed, artifacts
   on failure.
6. `golden-refresh`: manual dispatch; regenerates goldens and uploads
   the diff as an artifact for review.

Container constraints that matter:

- Pty allocation needs a mounted devpts; each devpts mount is a distinct
  instance and `/dev/pts/ptmx` has special permission mechanics; the
  global and per-instance pty counts are limited (`kernel.pty.max`,
  default 4096)
  ([devpts](https://www.kernel.org/doc/html/latest/filesystems/devpts.html)).
  Close both ends of every pty pair so leaked fds do not exhaust the
  limit in long test sessions.
- CI steps are not attached to a terminal. Never assert on `isatty()` of
  stdio, and set `TERM=dumb` globally in the environment so any
  accidental dependence on the ambient `TERM` fails immediately; tests
  then set the `TERM` they want.
- Pin the ncurses version used for `tic`; terminfo output and the
  extended-capability set change between releases. Set `TERMINFO` to the
  vendored directory so the host `/usr/share/terminfo` is never
  consulted.
- No network, no sleeps, no wall-clock assertions. `pytest-timeout` as a
  backstop, not as a test mechanism.
- Upload captured byte streams (hex), normalised grids, `TERM`, terminfo
  hash and oracle version on failure. A CI failure that cannot be
  reproduced locally is not debuggable.

## 11. Maintenance cost

| Choice | Setup cost | Ongoing cost | Failure diagnosis | When it pays |
|---|---|---|---|---|
| L0 unit tests | low | low | good | always; the base of the pyramid |
| Decoder libFuzzer | medium (clang, corpus) | low-medium (triage) | good, with reproducer | from the first decoder commit |
| Writer capability plan | low, needs ABI seam | low | excellent | always; makes L7 cheap |
| Byte goldens (1-3 profiles) | low | high churn on writer/terminfo changes | noisy diffs | writer optimisation and capability legality |
| Grid goldens + libvterm | medium (dependency, wrapper) | medium (oracle upgrades) | good, semantic diffs | cross-TERM semantic regression |
| pyte snapshots | low | low if scope is small | fair | convenience in Python-only tests |
| PTY integration | medium | medium; flaky on shared runners | hard (timing) | one or two end-to-end cases |
| Capability matrix (generated) | medium (terminfo parsing) | low, data-driven | good | colour/downgrade correctness |
| Hypothesis stateful | low-medium | low | excellent (shrunk program) | driver mode transitions |
| tack/vttest/esctest2 | low | manual | manual | adding profiles, investigating real terminals |
| Golden refresh job | low | low | n/a | keeping goldens reviewable |

Rules of thumb: every layer that requires a timeout is a flakiness
source; every golden without provenance is a future mystery; every
hand-maintained matrix row is a row that will be wrong.

## 12. Recommended architecture

- **C core:** GoogleTest or doctest through CTest for L0-L2. The decoder
  is an incremental pure state machine with a libFuzzer target and a
  checked-in seed corpus. The writer takes a sink and records a
  capability-use log. Byte goldens for `xterm-256color`, `vt100` and one
  `direct` profile. Grid goldens generated through libvterm for the full
  matrix. A C ABI smoke test compiled as C (`-std=c99`) plus a
  symbol-visibility check keeps the ABI honest.
- **PTY:** a small L6 job with `openpty()` (in-process where possible,
  one `forkpty()` child for the full pipeline), raw termios, explicit
  winsize.
- **Python:** pytest + pytest-asyncio; transport, clock and responder
  fakes; recorded byte corpus shared with the C tests; contract tests
  against the real shared library; one pty smoke test per driver
  feature.
- **CI:** the six jobs above; fuzz nightly, pty Linux-only, goldens
  refreshed by an explicit dispatch.
- **Explicitly not recommended:** byte goldens across different `TERM`
  values; pty-based tests for everything; pyte as the sole oracle;
  hand-edited goldens; terminal queries in the core without a responder
  seam; asserting `isatty()` or ambient `TERM` anywhere in tests.

## 13. What this changes for Orchidée

- The C ABI must take the output fd, the resolved capability profile and
  the size as parameters. Environment discovery belongs in the CLI
  wrapper, so that tests can pin `TERM`/`TERMINFO`/locale and the core
  never reads them.
- `present()`/`flush()` needs a byte-count or sink contract. Without an
  observable frame boundary, deterministic capture is not achievable and
  every end-to-end test needs a timeout.
- The writer should expose a capability-plan/use-log seam. That single
  decision turns the capability matrix and downgrade tests from
  byte-diffing into assertions over a small structured record.
- The decoder must be defined as an incremental, bounded, pure state
  machine from the start, with a fuzz target in the tree and a seed
  corpus derived from terminfo and xterm control sequences.
- Terminfo must be vendored and compiled in CI with a pinned `tic`, and
  byte goldens must record `TERM` plus the `infocmp` hash. Otherwise the
  goldens silently depend on the distro.
- libvterm becomes a dev/test dependency (pinned, linked into the C test
  binary or wrapped for Python); pyte stays optional. The product must
  not depend on either.
- The Python driver needs transport, clock and responder seams, and its
  tests should use `socketpair()` and fakes rather than a terminal. The
  pty is for the handful of end-to-end cases only.
- CI needs a sanitizer job, a nightly fuzz job with corpus caching, a
  Linux-only pty job, and a manual golden-refresh job. Budget for fuzz
  triage, not just the first run.
- Open questions this note does not settle: GoogleTest versus doctest
  (the tvision reference uses GoogleTest); whether to check in terminfo
  source or precompiled entries; the nightly fuzz time budget; whether
  any Python parsing is large enough to justify Atheris in addition to
  the C fuzzer; and whether the grid oracle is linked into the C test
  binary or exposed only through a Python wrapper.
