# Python binding for the Compositor C ABI

Research note for ticket #4. Input to #21 (binding technology and
packaging) and #20 (seam async and loop-integration contract).
Evidence only; no decisions here.

## Summary

Recommendation: bind with **cffi in API mode** (out-of-line
`set_source`, compiled at build time), behind a small hand-written
Python wrapper module. Do not use ABI mode for production, and do not
adopt Cython or pybind11 for the seam. Drive the loop with the
**fd-and-deadline contract**: the Compositor never blocks, never calls
back, and never owns the selector; it publishes the file descriptors it
wants watched and the next absolute deadline, and the host's `asyncio`
loop registers
`add_reader`/`add_writer` and `call_later` accordingly.

The binding choice is a smaller lever than the ABI shape. A per-cell
call costs 0.12 µs (best per-call FFI) to 0.62 µs (ctypes); a 2,000-cell
frame through per-cell calls therefore costs 0.25-1.24 ms before any
Python loop overhead. One bulk span call moves the same 2,000 cells in
0.2-1.1 µs. If the seam exposes span- and buffer-oriented drawing, the
binding is nearly free; if it exposes `set_cell`, no binding saves it.

## What the binding has to do

From the project context, the Compositor is:

- C++ with a C ABI, POSIX terminals only.
- Passive and loop-agnostic: no threads, no callbacks into the host.
- A retained surface tree addressed through handles.
- Semantic input events, with a language-agnostic seam.

So the Python binding's job is narrow: pass opaque handles and small
structs, move cell runs and event batches, and read back readiness
information. It does not marshal Python objects, does not expose C++
classes, and does not need callbacks (which is where FFI layers differ
most). That makes the comparison mostly about call cost, ABI-drift
checking, and packaging.

## Measured per-call overhead

Measured on 2026-11-20, x86-64 Linux, CPython 3.14.10, GCC 16.2.1,
against a 15-line C stub exposing `comp_set_cell(comp_t *, int32_t,
int32_t, uint32_t, uint32_t)` and `comp_set_span(comp_t *, int32_t,
int32_t, int32_t, const uint32_t *)`. Each figure is the best of three
runs of 200,000 calls in a straight-line Python loop:

| Binding path | ns/call | 2,000 cells via per-cell calls | one 2,000-cell span call |
| --- | ---: | ---: | ---: |
| ctypes `CDLL` | 619 | 1.24 ms | 1.1 µs |
| cffi ABI (`dlopen`) | 195 | 0.39 ms | 0.6 µs |
| cffi API (compiled) | 124 | 0.25 ms | 0.2 µs |
| Cython `cdef class` | 50 | 0.10 ms | 0.2 µs |
| pybind11 class method | 151 | 0.30 ms | not measured |

Context numbers from the same run:

- A pure-Python loop that only touches 2,000 four-tuples (no FFI at
  all) costs about 91 µs per frame. That is a third of the cffi-API
  per-cell frame cost and dwarfs one span call; Python-side iteration
  and object handling are the real per-cell tax.
- At 60 Hz the whole frame budget is 16.7 ms. Per-cell ctypes on a
  200x60 screen (12,000 cells) would cost about 7.4 ms, cffi API about
  1.5 ms, Cython about 0.6 ms. Small screens hide the problem; large
  ones do not.

Caveats: single machine, shell loops rather than closures, warm caches,
and a C stub whose body is one addition, so the numbers isolate call
boundary cost, not real drawing work. They are good to a factor of two,
not a benchmark suite. The span rows also show the stub's C loop running
at memory bandwidth, so treat them as an upper bound on what a bulk call
can cost, not a measurement of real cell drawing.

### Why the numbers look like this

- **ctypes** uses libffi for every call and, for `CDLL`, releases and
  reacquires the GIL around it
  ([ctypes docs](https://docs.python.org/3/library/ctypes.html)).
  `PyDLL` skips the GIL release and is faster for tiny calls, at the
  cost of assuming the C function may use the Python C API; there is
  also no build-time check of the signature, and `argtypes`/`restype`
  errors corrupt memory.
- **cffi ABI mode** also goes through libffi, and the cffi docs are
  blunt that this is slower and less safe: ABI calls "need to go
  through the very general libffi library, which is slow", ABI mode
  "will crash if you call some function or access some fields of a
  structure that was slightly misdeclared", and API mode "can be
  massively faster"
  ([cffi overview, ABI versus API](https://cffi.readthedocs.io/en/latest/overview.html#abi-versus-api)).
- **cffi API mode** generates a C extension that calls the target
  function directly, so the per-call cost is argument conversion plus
  the C call. It still releases the GIL before entering C
  ([cffi overview, thread safety](https://cffi.readthedocs.io/en/latest/overview.html#thread-safety)),
  which is a small fixed cost per call and a reason to batch.
- **Cython** compiles the wrapper and the call site; at 50 ns the FFI
  boundary is almost gone, but you now maintain a `.pyx` layer.
- **pybind11** is fast for what it does (151 ns here), but it is a C++
  binding framework. Its own benchmark page measures compile time and
  module size, not call cost
  ([pybind11 benchmark docs](https://pybind11.readthedocs.io/en/stable/benchmark.html)).

## Candidate by candidate

### ctypes

Standard library, no build step, no dependency. Ergonomics for a
handle-based API are acceptable: `c_void_p` for handles, `Structure`
subclasses for event structs, `POINTER(...)` for buffers. But every
prototype is set by hand at runtime; a wrong `restype` (the default is
`c_int`) truncates a 64-bit pointer to 32 bits and crashes or corrupts
memory. There is no build-time link check against the real header, so an
ABI change shows up as undefined behaviour rather than a compile error.
At 619 ns/call it is the slowest credible option. Verdict: fine for a
throwaway prototype (and useful as a zero-dependency fallback for
tests), wrong for production.

### cffi ABI mode

No compiler at install time, so `pip install` works anywhere; the cdef
is checked only against the programmer's transcription. At 195 ns it is
still 1.6x slower than API mode here. The real objection is safety:
runtime signature drift against a versioned C ABI is exactly the failure
mode an ABI compatibility policy is supposed to prevent. Verdict: a
useful escape hatch for source checkouts or system-installed libraries,
not the shipped path.

### cffi API mode

Out-of-line `set_source()` compiles a CPython extension that `#include`s
the real header, so signature mismatches fail the build, struct layouts
come from the compiler (cffi allows `...` for fields it should not
transcribe), and the wrapper is generated, not hand-written. Call cost
is 124 ns, and bulk buffer calls are cheap because cffi passes
`uint32_t *` from a Python buffer without copying
([cffi overview](https://cffi.readthedocs.io/en/latest/overview.html)).
Ergonomics: the cdef is a declarative copy of the public header, which
is a maintenance cost but a small, reviewable one; cffi also supports
`pkgconfig=[...]` for locating the library at build time
([cffi cdef docs](https://cffi.readthedocs.io/en/latest/cdef.html)).
Verdict: best fit.

### Cython

Fastest measured path (50 ns), and it can host hot Python-side loops in
C, which is a real option if a per-cell loop ever survives the ABI
redesign. Costs: a `.pyx` facade is another language in the Python
layer, harder to type-stub and debug; build integration is
Cython-specific; the Python layer's public surface ends up generated.
Also, Cython's free-threading support is explicitly experimental
([Cython free threading docs](https://cython.readthedocs.io/en/latest/src/userguide/freethreading.html)).
Verdict: keep as a targeted optimisation behind the same Python facade
if profiling ever justifies it; do not make it the seam.

### pybind11

Excellent for binding C++ classes, which is not the seam here: the seam
is deliberately a C ABI so that non-Python hosts can use it. Using
pybind11 for the Python layer means the Python binding is written
against C++ internals rather than the published C ABI, and the
Python-layer package picks up a C++ compiler and a C++ ABI. Call cost
(151 ns) is not better than cffi API mode. Verdict: out of scope.

### Cost of keeping the binding in Python

There are three separable costs:

1. Per-call overhead, measured above. cffi API mode removes most of it;
   the rest is better removed by moving work across the boundary in
   bulk than by moving the wrapper into C.
2. Wrapper maintenance. A generated cffi extension is regenerated from
   the cdef and the header at build time; a Cython or C++ wrapper is a
   second implementation of the header's types that must be edited when
   the ABI changes. cffi wins here.
3. Debuggability and typing. All options are poor at type stubs; cffi's
   cdef at least names every signature in one file, which can be
   checked in and diffed, and a build-time compile against the real
   header is a strong ABI test.

If profiling later shows the wrapper itself is hot (not the C side),
the right move is to push the operation into the C ABI, not to rewrite
the binding in Cython; a facade makes that a local change.

## Free-threaded Python

Facts, from primary sources:

- Free threading exists since CPython 3.13; the build is identified by
  the `Py_GIL_DISABLED` macro. Extension modules must declare support
  (a `Py_mod_gil` module slot, or `PyUnstable_Module_SetGIL`), or
  importing them disables free threading by re-enabling the GIL
  ([CPython free-threading C API howto](https://docs.python.org/3/howto/free-threading-extensions.html)).
- The free-threaded build does not support the Limited C API or the
  stable ABI, so abi3 wheels do not cover `cp314t`; separate `t`-tagged
  wheels are needed, and manylinux/cibuildwheel support them
  (same howto; also [PEP 703](https://peps.python.org/pep-0703/)).
- cffi added free-threaded support in 2.0, with the caveat that
  `py_limited_api=False` must be used for free-threaded builds, and
  2.1 added `abi3t` targeting
  ([cffi changelog](https://cffi.readthedocs.io/en/latest/whatsnew.html)).
  cffi releases the GIL before calling into C on GIL builds and
  documents that this makes it usable from both build types.
- ctypes works on the free-threaded build; its docs have a "Thread
  safety without the GIL" section describing which concurrent accesses
  are safe ([ctypes docs](https://docs.python.org/3/library/ctypes.html)).
  Verified locally: `python3.14t -c "import ctypes, sys;
  print(sys._is_gil_enabled())"` prints `False` on a CPython 3.14.10t
  install, i.e. importing ctypes does not re-enable the GIL.
- pybind11 added free-threaded support in 2.13.0 with the
  `py::mod_gil_not_used()` tag, and 3.0 added Python 3.14/3.14t support
  ([pybind11 changelog](https://pybind11.readthedocs.io/en/latest/changelog.html)).
- Cython 3.1+ has "basic support" for free-threaded builds, marked
  experimental, with a `freethreading_compatible` directive
  ([Cython free threading docs](https://cython.readthedocs.io/en/latest/src/userguide/freethreading.html)).

What this means for Orchidée: the seam contract already says one thread
and no callbacks, so nothing in the design needs free threading, and
free-threaded builds buy the library nothing. But the binding must not
*punish* users who run one: a cffi API-mode extension can be built for
`cp314t` (without `py_limited_api`) and should declare GIL-not-used, as
should any Cython alternative. The practical cost is a wider wheel
matrix: per-arch, per-minor-version, optionally plus `t` variants.

## Packaging on POSIX

Shape that follows from choosing cffi API mode:

- One build script (e.g. `python/orchidee/_compositor_build.py`)
  holding `ffi.cdef()` and `ffi.set_source()`, invoked by
  `setup.py`/`pyproject.toml` through `cffi_modules` or the
  `cffi-gen-src` tool. Generated C can be committed (reproducible
  builds without cffi at build time) or generated in the build hook
  (better checked against the header). The cffi changelog added
  `cffi-gen-src` in 2.1 for build backends such as meson-python
  ([cffi changelog](https://cffi.readthedocs.io/en/latest/whatsnew.html)).
- Either vendor the C++ sources into the extension (`sources=[...]`, one
  self-contained wheel) or link a system library found via
  `pkgconfig=["orchidee-compositor"]`. Pick one as the default and keep
  the other as an opt-in; mixing both in one wheel is where "works on
  my machine" bugs come from.
- POSIX-only simplifies the matrix: no Windows DLL search rules. Wheels
  still need to be split by platform and by CPython minor version
  because there is no stable-ABI story under free threading; do not
  promise abi3.
- A `ctypes` or cffi-ABI fallback is cheap to offer for source
  checkouts and CI without a compiler, but it should not be the
  documented install path.
- Version coupling: pin the ABI major/minor as constants in the cdef,
  expose `comp_abi_version()`, and have the Python wrapper refuse to
  import against an unexpected major. The cdef is a copy of the public
  header, so keep a test that builds it against the header on every
  change.

## Loop integration: what other Python TUI code does

### asyncio's low-level fd API

`loop.add_reader(fd, callback, *args)` and `loop.add_writer(fd,
callback, *args)` register one callback per fd for readability and
writability; `remove_reader`/`remove_writer` unregister them
([asyncio event loop docs](https://docs.python.org/3/library/asyncio-eventloop.html)).
Callbacks run in the loop thread, so they must not block. The docs'
own example is exactly the shape a passive library needs: register a
reader on an fd, drain it in the callback, remove the reader when done.
`call_later(delay, callback)` and `call_at(when, callback)` cover
deadlines, using the loop's monotonic clock, and return a handle that
can be cancelled and rescheduled.

This API is Unix-oriented; the default Windows proactor loop does not
support fd watching. That is fine for a POSIX-only Compositor, and it
is the reason the seam should publish *fds plus deadline* rather than
asyncio-specific objects: another host can drive the same contract with
`select`, `epoll`, or `trio.lowlevel.wait_readable`.

### prompt_toolkit: add_reader plus a byte parser

`Vt100Input` on POSIX keeps the stdin fileno, registers
`loop.add_reader(fd, callback_wrapper)` when attached, drains bytes in
the callback, feeds them to a `Vt100Parser`, and removes the reader on
detach. Raw mode is set with `termios`/`tty` around the application
([prompt_toolkit vt100.py](https://github.com/prompt-toolkit/python-prompt-toolkit/blob/master/src/prompt_toolkit/input/vt100.py)).
The important structural point: the library owns *decoding*, the loop
owns *waiting* and *reading*. That is the shape the Compositor should
mirror.

### curses: the model to avoid

curses is a blocking-input model: `window.getch()` blocks, `timeout()`
sets a positive delay (or zero for non-blocking), `nodelay()` is the
flag form, and `halfdelay(tenths)` makes `getch` block then return -1
([curses docs](https://docs.python.org/3/library/curses.html)). It also
needs an escape-sequence deadline: `set_escdelay(ms)` sets how long to
wait after an ESC byte before deciding it was the Escape key rather
than the start of a sequence
([curses docs, set_escdelay](https://docs.python.org/3/library/curses.html#curses.set_escdelay)).
If Orchidée copied this, the library would own the wait and the host
loop would be bypassed. The lesson to keep is the *deadline*: an input
parser needs a timeout, so the seam must report "no more input is
processable until time T" to the host, and the host arms
`call_later(T - loop.time())`.

### pyte: parser with no I/O at all

pyte is an in-memory VT emulator, not a terminal driver: callers feed
bytes into a `Stream` and read screen contents from `Screen` objects
([pyte](https://github.com/selectel/pyte)). It has no loop, no fds, no
threads. It is the purest demonstration that decoding and rendering can
be separated from loop ownership, and it suggests the Compositor's
input decoder should be a feedable state machine (`comp_feed(bytes,
len)`) rather than something that reads the terminal itself.

## The loop-integration pattern the seam should anticipate

Contract, in the seam's own terms:

1. **Readiness is data, not callbacks.** The Compositor exposes the fd
   set it wants watched and the events of interest (read the tty for
   input, write the tty for output; possibly a signal-safe wakeup fd).
   `comp_fds(handle, comp_fd_t *out, size_t cap)` is enough; the host
   translates that to `add_reader`/`add_writer` in asyncio, `select` in
   a plain loop, or a `trio` primitive.
2. **Deadlines are absolute and queryable.** `comp_next_timeout(handle,
   struct timespec *out)` returns the parser's next deadline (escape
   ambiguity, repeat timers, cursor blink if it ever exists) or "none".
   The host arms `call_later`/`call_at`; the C side never sleeps.
3. **Input is pushed in, not pulled.** `comp_feed_input(handle, bytes,
   len)` lets the host read whatever the OS gave it and hand it over;
   the host keeps ownership of the read loop and of EAGAIN. This is the
   prompt_toolkit/pyte split, and it keeps the parser testable without
   a terminal.
4. **Output is pulled out with partial-write accounting.**
   `comp_flush(handle, void **buf, size_t *len)` (or fill/write pairs)
   returns a byte window; the host writes it and reports how much the
   kernel took, exactly like `loop.add_writer` expects. The Compositor
   must tolerate partial writes and never assume it can finish a frame
   in one syscall.
5. **Mutations are loop-thread-only.** Host calls
   `comp_surface_*`/`comp_draw_*`/`comp_present()` from the same thread
   that runs the loop. No wakeup fd is needed for the "host changed a
   surface" case, because the host is the one that will flush next.
   Document this as part of the single-thread rule. If a later feature
   needs cross-thread mutation, that is a new ABI decision, not
   something to smuggle in now.
6. **Resize and signals stay in the host.** SIGWINCH and SIGINT are
   signals, not fd readiness; asyncio's `add_signal_handler` is the
   host's tool, and the seam should offer `comp_resize(cols, rows)` and
   a quit path instead of installing signal handlers itself.
7. **Shutdown is ordered and explicit.** `comp_restore(handle)` must be
   callable after partial writes and after a signal-driven abort, and
   the host should call it from `finally`. The contract should state
   that no fd is closed by the Compositor, so the host never races its
   own reader removal against the library.

For asyncio specifically, this collapses to: one task (or no task at
all), readers on the input fd, a writer on the output fd only while
`comp_flush` reports pending bytes, and one `call_later` rescheduled
after every input event and every timer fire.

## Recommendation

1. **Binding:** cffi API mode, out-of-line, compiled against the
   Compositor's public header; a thin hand-written wrapper
   (`compositor.py`) owns lifetime, error translation, and loop
   integration; the generated `_compositor` extension is an
   implementation detail. Ship as platform/version-specific wheels;
   keep a ctypes-based fallback for tests and compiler-less source
   checkouts, not for production.
2. **ABI shape:** expose span/cell-run drawing and a `present` call that
   can be batched, plus a cell buffer the host can fill through
   `ffi.from_buffer`/memoryview. A `set_cell`-only ABI makes every
   binding look bad and caps the achievable frame rate.
3. **Loop pattern:** fd set plus absolute deadline in the seam; asyncio
   `add_reader`/`add_writer` and `call_later` in the Python driver;
   input pushed in, output pulled out with partial-write accounting; no
   callbacks, no sleeps, no signal handlers in C.
4. **Free threading:** not a seam requirement; build the extension for
   `cp314t` (without `py_limited_api`) only when a free-threaded wheel
   is actually requested, and declare GIL-not-used so a `3.14t`
   interpreter is not silently forced back to the GIL.

## What this changes for Orchidée

- The ABI must be **batch-first**: span/cell-buffer drawing and one
  present call, not one call per cell. This is an ABI-level requirement
  driven by measured cost, not a Python convenience.
- The seam must publish **fds plus an absolute deadline** (and accept
  pushed input), so that asyncio, `select`, and other runtimes can all
  drive it without the Compositor owning the loop. That is the concrete
  contract #20 should write down, and it should be runtime-neutral.
- The Python layer should be **cffi API mode plus a thin wrapper**, with
  ctypes only as a test/bootstrap path. This settles the "compiled
  extension vs pure Python" question in #21 in favour of compiled, with
  an explicit fallback story.
- Plan for a **wheel matrix without abi3** (per platform and per CPython
  minor version, optional `t` variants), and a build-time header check
  as the ABI-drift test.
- Performance work should target the **Python-side per-cell loop** and
  the ABI's batching, not the binding: at 50-150 ns per call the FFI
  boundary is already small compared with the cost of touching 2,000
  Python objects.
