# The Compositor installs no signal handlers, and crash restore is not attempted

The Compositor is passive by constitution — no threads, no callbacks
into the host, no blocking, no signals — so `SIGWINCH`, `SIGTSTP`, and
`SIGCONT` are the host's business, and the terminal's lifecycle is
driven by calls, not events. The restore guarantee therefore stops at
failures the host can see: an uncaught exception, `SIGTERM`/`SIGHUP`/
`SIGINT` from `kill`, and a suspend. Fatal signals and `SIGKILL` leave
the terminal wedged, and recovery is `stty sane` plus `tput rmcup`, or
`reset`.

## Considered options

- **Install handlers inside the core.** It would catch `SIGWINCH` and
  the suspend dance directly, but it contradicts the constitution the
  dependency policy is built on, introduces async-signal-safety
  obligations into a C++ library, and makes the core's behaviour
  depend on process-global state a host cannot reset.
- **Install a fatal-signal handler in the host**, restoring from a
  precomposed byte string. `faulthandler` proves the mechanism works,
  but the write runs while the interpreter that owns the terminal is
  already broken; a partial or reordered restore is worse than a known
  wedged state with a two-command fix.
- **No handler, documented recovery** (chosen). Fewer moving parts at
  exactly the moment nothing can be trusted, and the restore bytes are
  still exposed as a query, so the option stays open.

## Consequences

The Python layer installs `SIGWINCH`, `SIGTSTP`, `SIGCONT`, `SIGTERM`,
`SIGHUP`, and `SIGINT` handlers, plus an `atexit` backstop, and must
call `leave` on every exit path. User-facing documentation must name
the recovery commands for a crashed session. The backend's restore
script is a seam query rather than a private detail, which also gives
the test stack an exact expectation for what `leave` writes. Normative
detail is in
[terminal-backend.md](../design/terminal-backend.md).
