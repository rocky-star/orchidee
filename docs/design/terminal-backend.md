# Terminal backend lifecycle and modes

Normative design for
[Terminal backend lifecycle and modes](https://github.com/rocky-star/orchidee/issues/10).
It fixes how the Compositor takes a terminal, which modes it owns,
what a session restores and who restores it, and which half of that
work belongs to the host. The descriptions it consumes come from
[capabilities.md](capabilities.md); the loop that drives it belongs to
[#20](https://github.com/rocky-star/orchidee/issues/20), and the
synchronized-output block it must never cut in half belongs to
[#13](https://github.com/rocky-star/orchidee/issues/13).

Why none of this lives in a signal handler is recorded in
[ADR-0003](../adr/0003-no-signal-handlers-in-the-compositor.md).

## Passive means the host owns the process, not the terminal

The Compositor installs no signal handlers, starts no threads, blocks
on nothing, and never calls back into the host. Everything it needs to
know about the outside world arrives as a call or as input bytes.

That one constraint produces the whole split:

- The **backend** owns the terminal's *state*: the file descriptors it
  is given, the saved termios, the mode set, and every byte written to
  the screen.
- The **host** owns the process's *signals*: `SIGWINCH`, `SIGTSTP`,
  `SIGCONT`, `SIGTERM`, `SIGHUP`, `SIGINT`, `atexit`, and the choice of
  when to suspend.

Neither side may do the other's half. A backend that installs a
`SIGWINCH` handler breaks the constitution; a host that calls
`tcsetattr` itself desynchronises the saved termios.

## The two file descriptors

The backend is handed descriptors. It never opens, closes, duplicates,
or `fcntl`-modifies them.

| Descriptor | Purpose | Constraint |
| --- | --- | --- |
| **control fd** | `tcgetattr`, `tcsetattr`, `TIOCGWINSZ` | must be a terminal, else start-up fails |
| **output fd** | every byte the backend writes | defaults to the control fd; `O_NONBLOCK` is a precondition |

- `O_NONBLOCK` on the output fd is **checked** at enter with
  `fcntl(F_GETFL)` and never *set* or *cleared* by the backend. A
  blocking output fd is a start-up error: the backend may not block.
- The control and output descriptors may be the same. When they
  differ, both must resolve to the same terminal — compared with
  `fstat` on device and inode — because termios and size are
  per-terminal, not per-descriptor.
- The backend never opens `/dev/tty`. A host that wants the keyboard
  while stdin is redirected opens it and passes it in; v1's default
  driver simply requires stdin and stdout to be the terminal.
- Only **one backend may be active per process**. The mode set is
  process-wide terminal state, so a second active backend is refused
  with a diagnostic.
- The backend does **not** read. The host reads the input fd and hands
  the bytes to the input decoder; the backend only writes.
- Calls are not thread-safe. The host serialises them; the seam's
  single-thread rule (#20) depends on that.
- The core **writes** its output rather than handing bytes back.
  Short writes are accounted for and exposed, so the host knows when to
  wait for writability; the exact flush API is #20's.

### Headless

`control_fd = -1`, with an explicit `cols`/`rows` and an explicit
capability profile, is a supported configuration rather than a test
backdoor: no termios, no ioctl, no tty validation, but the enter and
leave scripts are still written to the output fd. It is what the test
stack's injected-fd design needs (see
[#5](https://github.com/rocky-star/orchidee/issues/5)) and what the
Python driver's headless mode is built on (#22). The real-terminal
path stays covered by the `openpty` end-to-end job.

## States

Two states and two verbs:

```
inactive --enter--> active --leave--> inactive
```

- `enter` from inactive: save the current termios, set raw mode, write
  the mode set, query size, invalidate the frame, and ask for a full
  repaint (#11).
- `leave` from active: write the restore script, restore termios.
- `enter` while active is refused with a diagnostic. `leave` while
  inactive is a no-op. Both verbs are idempotent in that sense, so a
  `finally` block, an `atexit` hook, and a `SIGTERM` handler can all
  call `leave` without coordinating.
- The capability snapshot is resolved **once**, at the first enter,
  and stays fixed for the backend's lifetime. A `$TERM` change while
  inactive is not noticed; a new backend is how it is picked up.
- The saved termios is re-read on **every** enter, because the shell
  may have changed it while the terminal was not ours. Restoring a
  copy captured an hour ago would clobber that.

**Suspended** is the host's name for *inactive* while the job is
stopped, or while a child owns the terminal. The backend cannot tell
those apart and does not need to: every enter re-reads termios and
re-emits the mode set, so it does not matter why it left or how long
it was gone.

## Enter

Fixed order; each step is a precondition for the next.

1. **Validate**: the control fd is a terminal, `$TERM` is not `dumb`
   (the capability page's rule), and the output fd is non-blocking —
   in headless mode too, because a full queue is still not a reason to
   block.
2. **Resolve**: read terminfo through libtinfo, build the capability
   snapshot, resolve the sequences the mode set needs into the
   backend's private table, then `del_curterm`.
3. **Save** the current termios.
4. **Raw mode** with `TCSAFLUSH`: output drained, type-ahead discarded.
5. **Write the enter buffer**, composed once: alternate screen, mode
   set, cursor hidden, title.
6. **Query size** with `TIOCGWINSZ`.

Nothing is written before step 4: the enter buffer's clear-screen and
mode changes must not interleave with the echo of bytes typed at the
shell prompt.

The enter buffer is a single buffer, not a series of calls per mode.
A failure anywhere in steps 1–6 unwinds by calling `leave`, and
`leave` does not inspect how far enter got — the restore script is
unconditional, so a half-entered terminal is still returned to a sane
state.

## Raw mode

Not `cfmakeraw`: its `VMIN=1` is wrong for a core that never blocks.

| Flag word | After enter |
| --- | --- |
| `c_iflag` | clear `IGNBRK`, `BRKINT`, `PARMRK`, `ISTRIP`, `INLCR`, `IGNCR`, `ICRNL`, `IXON`, `IXOFF` |
| `c_oflag` | clear `OPOST` |
| `c_lflag` | clear `ECHO`, `ECHONL`, `ICANON`, `ISIG`, `IEXTEN` |
| `c_cflag` | `CS8` set, `CREAD` kept, baud untouched |
| `c_cc` | `VMIN = 0`, `VTIME = 0` |

`OPOST` is cleared deliberately: with `ONLCR` left on, the kernel
rewrites `\n` to `\r\n` inside a pty, so PTY byte goldens would diverge
from direct-fd goldens — exactly the line-discipline trap #5 warns
about. The consequence is that the backend never emits a bare `\n`
meaning "new line"; it either positions the cursor or writes `\r\n`.
Crash diagnostics do not care, because termios is restored before any
diagnostic is printed.

`ISIG` off is why Ctrl-C, Ctrl-Z, and Ctrl-\ arrive as bytes, not
signals. That is intentional: the decoder turns them into key events,
and the application decides what they mean.

The saved copy is restored **exactly**, never a synthesised "cooked"
mode: the shell's `stty` settings, `VINTR` bindings, and baud are the
shell's business.

Timing on the two `tcsetattr` calls:

- **enter**: `TCSAFLUSH` — drain pending output, discard type-ahead
  that was aimed at the shell.
- **leave**: `TCSADRAIN` — drain the restore script we just wrote,
  then apply. The terminal never sees the termios flip mid-script.

## The mode set

Every sequence is capability-gated and comes from the backend's
private table — resolved from terminfo at step 2, or from the curated
fallback tier — never from a runtime terminfo call and never hardcoded
where the snapshot has a string.

| Mode | Sequence | When | Default |
| --- | --- | --- | --- |
| Alternate screen | `smcup` / `rmcup` (else `?1049`) | enter / leave | on when available |
| Mouse tracking | `?1002` | enter | on when available |
| Mouse encoding | `?1006` | enter | on with mouse |
| Bracketed paste | `?2004` (`BD`/`BE`) | enter | on when available |
| Focus events | `?1004` | enter | on when available |
| Kitty keyboard | `CSI > 1 u` (push) | enter | on when capable, level 1 only |
| Kitty keyboard | `CSI < u` (pop) | leave | always |
| Cursor visibility | `civis` / `cnorm` | enter / leave | always |
| Cursor shape | `DECSCUSR` | — | **never sent** |
| Application cursor keys | `smkx` / `DECCKM` | — | **never sent** |
| Autowrap | `?7h` | enter / leave | ensured on |
| Scroll region | `CSI r` | leave | reset |
| Synchronized output | `?2026` | present (#13) | closed on leave if open |
| Window title | `tsl`+text+`fsl`, else OSC `0;`/`2;` | enter | set, not restored |
| OSC 52 clipboard | `Ms` | on demand | write-only, 64 KiB cap |

Notes:

- **Mouse levels** are the application's choice through one call:
  off, clicks (`?1000`), drag (`?1002`), motion (`?1003`). The default
  is drag, because scrollbar and list-selection dragging need
  button-motion events, and `?1003` would flood the decoder with
  motion nobody asked for. The level change is emitted on the ordered
  output path, so it can never land inside a frame.
- **Kitty keyboard** is enabled at level 1 only (disambiguate escape
  codes), pushed so a nested program can pop back. The decoder must
  still accept the legacy forms; that half is
  [#14](https://github.com/rocky-star/orchidee/issues/14)'s.
- **Cursor shape** is the user's preference, not the application's.
  Only visibility is touched.
- **`smkx` / `DECCKM` are never sent.** Application cursor keys would
  change the arrow-key sequences for no gain; the decoder accepts both
  CSI and SS3 forms instead.
- **Title** is sanitised before it becomes an OSC: C0/C1 control bytes
  stripped (BEL and ESC terminate an OSC early), tab and newline
  replaced with spaces, capped at 200 bytes, truncated on a UTF-8
  boundary. The host supplies the title at enter; without one, or
  without a title capability, nothing is sent — never a speculative
  OSC. The title is not restored on leave; the shell rewrites it at
  the next prompt.
- **OSC 52** is write-only in v1. One sequence, base64, refused above
  64 KiB with a distinct "too large" result rather than chunked.
  Reading the clipboard needs response correlation inside the input
  decoder and stays deferred with #14.

## Resize

The core never installs a `SIGWINCH` handler and never polls size.

- The host installs the handler — `signal.signal` in the sync driver,
  `loop.add_signal_handler` under asyncio — and on the signal calls
  `refresh_size()`, which does `TIOCGWINSZ` on the control fd.
- The resulting size enters the core as an **event**, not as a mode
  change. How the surface tree responds is #11's.
- At enter and after resume the same call runs. Before the first
  successful ioctl the size is 80x24 with `fallback` provenance, per
  the capability page; after one success, a failing ioctl keeps the
  last known size rather than reverting.
- Resize storms coalesce: the host reports at most one size event per
  loop turn, and the current size is authoritative.

## Suspend, resume, and lending the terminal

One primitive pair, `leave` then `enter`, serves three callers. Turbo
Vision has the same shape (`TScreen::suspend`/`resume` around
`system()`).

Raw mode means Ctrl-Z is byte `0x1A`, not `SIGTSTP`, so:

1. **Suspend by key.** The application binds Ctrl-Z; the driver calls
   `leave`, then re-raises `SIGTSTP` with the default disposition, so
   the shell takes the terminal back and the job really stops.
2. **Suspend by signal.** A `SIGTSTP` from `kill` hits the host's
   handler, which runs the same path: `leave`, then default action.
3. **Lending to a child.** An external command — the file manager will
   run them — is `leave`, spawn, wait, `enter`. The child must never
   inherit the mode set.

On `SIGCONT` the host's handler only **sets a flag**. The loop
re-enters at the top of the next turn: re-read termios, re-emit the
mode set, `TCSAFLUSH` (dropping whatever the shell left in the input
queue), re-query size, request a full repaint. Re-entering inside the
handler would run the core at an arbitrary point in the host's frame
cycle.

`leave` before stopping is not optional: a job that stops in raw mode
leaves the shell prompt with no echo and no line discipline.

## Leave and the restore script

`leave` writes one **restore script**: a precomposed, unconditional
byte sequence that returns the terminal to how the backend found it.
Unconditional is the property that makes a partial enter safe —
disabling a mode that was never enabled is harmless — and it is also
what makes the script byte-comparable in tests.

Order within the script:

1. Close an open synchronized-output block (`?2026l`).
2. Mouse encoding and tracking off (`?1006l`, level off).
3. Bracketed paste off (`?2004l`).
4. Focus events off (`?1004l`).
5. Kitty keyboard pop (`CSI < u`).
6. `sgr0`, then autowrap on (`?7h`), then scroll region reset
   (`CSI r`).
7. Cursor shown (`cnorm`).
8. Alternate screen exited (`rmcup`), or, where there is none, park
   the cursor on the line below the frame with `\r\n` so the prompt
   does not overwrite the UI.
9. Restore the saved termios with `TCSADRAIN`.

The first eight steps are the script; step 9 is the syscall that
follows it. The script is written once and is at most a few hundred
bytes.

### Partial writes

The output fd is non-blocking, so a write may be short or refused.

- A short write means the buffer filled: continue writing the
  remainder.
- `EAGAIN` means the terminal's output queue is full and the backend
  may not wait. Record a **restore incomplete** status, return it, and
  let the driver report it — never sleep, poll, or spin.
- `EBADF` or `EIO` means the terminal is gone: ignore the write error
  and still restore termios.
- The enter buffer follows the same short-write rule. If it cannot be
  completed, `enter` reports failure and unwinds by `leave`.
- Exposure is small by construction: the script is tiny, `leave` runs
  when the queue is idle, and undrained frame bytes are discarded
  before the script is composed, because a frame nobody will see is
  not worth writing.

`destroy()` on an active backend performs `leave` first, so a host
that forgets does not leak the terminal. It never emits a restore from
an inactive backend.

## The restore guarantee

| Failure class | Who acts | Guarantee |
| --- | --- | --- |
| Normal exit, context manager | driver `finally` | full restore |
| Uncaught Python exception | `finally` plus `atexit` backstop | full restore |
| `SIGTERM`, `SIGHUP`, `SIGINT` from `kill` | host handler: `leave`, then default exit | full restore |
| `SIGTSTP`, `SIGCONT` | the suspend protocol | restore before stop, re-enter on continue |
| Fatal: `SIGSEGV`, `SIGBUS`, `SIGABRT` | nobody | **none** |
| `SIGKILL` | nobody, possible | none |

"Full restore" always means, and only means:

- the saved termios copy, restored exactly;
- every mode set above disabled, kitty flags popped, an open
  synchronized-output block closed;
- alternate screen exited, or the cursor parked below the frame when
  there is none;
- cursor visible, autowrap on, scroll region reset, `sgr0`;
- `O_NONBLOCK` untouched, because it was never set;
- descriptors never closed, so the host decides what happens to them;
- title left as set.

No fatal-signal handler is installed in v1, by design (ADR-0003). A
wedged terminal is recoverable in two commands — `stty sane` and
`tput rmcup`, or simply `reset` — while a handler running on a
half-broken interpreter is not something to trust with the terminal.
The backend does expose the restore script as a query, so a host that
later wants a fatal-signal path has the exact bytes, and so tests can
assert that `leave` writes precisely them.

## Diagnostics

Nothing is written to stderr or any other descriptor while the backend
is active: it would corrupt the screen and break the frame's
byte-level expectations.

- Start-up failures unwind first, then report.
- The `leave` path is the only place a diagnostic may follow terminal
  output.
- Capability provenance (the capability page's `terminfo | env |
  override | fallback | unknown`) is available to that diagnostic, so
  a report can say which terminal it believed it was talking to.

## Verification

The ticket's test hooks, per the stack in
[#5](https://github.com/rocky-star/orchidee/issues/5):

- **Byte goldens** for the enter and restore scripts per pinned
  `TERM`, asserted against the headless path's output fd.
- **Termios round-trip**: a pty whose settings are captured, entered,
  and left, comparing the restored copy to the original; and after a
  simulated suspend and resume.
- **Script equality**: `leave`'s bytes equal the queried restore
  script, including after a failed enter.
- **Partial-write behaviour**: a socketpair with a small buffer that
  forces a short write, and an `EAGAIN` that sets "restore
  incomplete".
- **Suspend/resume**: a pty test that raises `SIGTSTP` and `SIGCONT`
  and checks that termios and modes are restored while stopped and
  re-entered afterwards.

## Owned elsewhere

- Synchronized-output block content and pacing: #13.
- CSI-u and legacy key decoding, focus-event delivery: #14.
- Flush API, fd readiness, and the write queue: #20.
- Full repaint and invalidated regions after resume: #11.
- Clipboard read, active capability probing, title push/pop stacks:
  deferred, and tracked as fog on the map.
