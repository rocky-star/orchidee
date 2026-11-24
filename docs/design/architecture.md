# Architecture: layers, boundaries, and dependencies

Normative design for
[Foundation and dependency policy](https://github.com/rocky-star/orchidee/issues/7).
It fixes the dependency posture the rest of the design builds on;
later tickets amend it as they refine a layer. The reasoning behind
system-first acquisition is in
[ADR-0001](../adr/0001-dependency-policy.md); the build-versus-adopt
evidence is in
[#2](https://github.com/rocky-star/orchidee/issues/2) and
[#3](https://github.com/rocky-star/orchidee/issues/3).

## Layers

Orchidée has two layers and one seam.

- **Compositor** (the core layer): C++ internals behind a C ABI. It
  owns the terminal backend, the capability snapshot, input decoding,
  the surface tree, damage, composition, and the present pipeline. It
  is passive and loop-agnostic: no threads, no callbacks into the
  host, no blocking reads, no signals.
- **Seam**: the language-agnostic contract per-language layers code
  against — the C ABI plus its conventions. It is the only interface
  the Compositor exposes.
- **Python layer**: widgets, event dispatch, window management, and
  async integration, built on the seam only. It keeps the host loop.

Dependencies point one way. The Compositor knows nothing above it,
and a per-language layer never reaches into Compositor internals.

## Adopted dependencies

| Dependency | Role | Acquired as | Floor |
| --- | --- | --- | --- |
| libtinfo (ncurses) | terminfo access for the capability snapshot | system library, required | ncurses 6.1 |
| utf8proc | UTF-8 decode, UAX #29 grapheme boundaries, width | system library when found; pinned download otherwise | 2.10.0 |

Nothing else is adopted as a library. notcurses, PDCurses, ncurses's
screen and window layers, libvterm, libtermkey, unibilium, and
libgrapheme are not dependencies; where they informed the design it
was as prior art, not as code. termbox2 is not a dependency either;
selected fragments may be copied under
[Copying from termbox2](#copying-from-termbox2).

## The line: below and above the Compositor

*Below* the Compositor is anything compiled, linked, or required at
build time for the native library. *Above* it are the Python layer
and the repository's development tooling.

A dependency below the line must pass both tests:

1. **Licence**: it is on the allowlist below.
2. **Constitution**: it is passive — no threads, no signal handlers,
   no event loop, no blocking I/O, and no process-global state the
   caller cannot reset. It writes to the terminal only through the
   Compositor's sink.

The seam is guarded too. Public headers include only `stdint.h`,
`stddef.h`, and `stdbool.h`. No dependency type, header, or symbol
crosses the C ABI, and nothing from a dependency is installed.

Banned below the line: ncurses's screen and window layers, ICU,
pthreads, `dlopen`-style runtime optional dependencies, and libc
`wcwidth` as the *primary* width source (it stays a single-code-point
fallback).

Above the line, Python dependencies such as cffi and development
tooling are allowed. They must never be required to build or link the
core, and must never appear in the C ABI.

## Licence policy

MPL-2.0 is file-level copyleft: Orchidée's own files stay MPL-2.0,
and combined dependencies keep their own terms.

Allowed for anything shipped or built in:

- MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, X11, Zlib, 0BSD,
  and public-domain or CC0 works.

Not allowed in the default build: LGPL, GPL, AGPL, or any other
copyleft licence. An LGPL dependency may return only as an optional,
dynamically linked, off-by-default build flag, and only by decision
(the unibilium precedent in #3).

Development and test tooling may additionally use Apache-2.0 WITH
LLVM-exception (libFuzzer). That exception list is closed and changes
by decision, not by convenience.

Vendored or copied third-party files keep their own licence text and
are never relicensed to MPL-2.0. Every third-party component and
every copied fragment appears in a root `NOTICE` file with its
copyright and permission notice.

## Acquisition and version discovery

Both dependencies are discovered at configure time — pkg-config first
(`tinfo`, `ncursesw`, `libutf8proc`), then a `find_library` plus
header probe — and checked against the floor above. A missing or
too-old library fails configuration with a message naming the install
package and the alternative acquisition path. There is no runtime
version probing.

### libtinfo

Required in the default build; v1 has no terminfo-less build. The
Compositor reads the description once at start-up into its own
capability snapshot and releases the library's global state
immediately. terminfo is never consulted from the buffer or present
paths. The built-in fallback tiers handle *runtime* absence — an
unknown `$TERM` or a missing database — not build-time absence.

### utf8proc

System first, download as the fallback:

- `ORCHIDEE_UTF8PROC=Auto|System|Fetch`, `Auto` by default: the system
  library when it meets the floor, otherwise `Fetch`.
- `Fetch` pins one release with `URL_HASH`; the network is touched
  only on that path.
- `FETCHCONTENT_SOURCE_DIR_UTF8PROC` pre-seeds an offline build.
- There is no in-tree copy and no git submodule. A GitHub source
  tarball does not carry a submodule's contents, so neither is
  self-contained at release time.

The found or downloaded copy is built privately: no installed
headers, no exported symbols.

## Copying from termbox2

termbox2 (MIT) is a donor, not a dependency. Fragments may be copied,
adapted, or reworded, including:

- raw and cooked mode entry and restoration;
- alternate-screen and cursor setup and teardown;
- terminal detection heuristics;
- input parsing as a starting point, reworked to the passive contract;
- captured escape sequences used as test vectors.

Copied code must satisfy the constitution: no `tb_global`-style
singleton, no `SIGWINCH` handler, no blocking event call. Each
fragment carries the MIT copyright and permission notice, a
provenance line naming the upstream commit or release and marking it
adapted, and an entry in `NOTICE`. Copied fragments are Orchidée code
to maintain; upstream rebases happen deliberately, never
automatically.

Capability tables are generated from `terminfo.src`, never copied
from termbox2.

## Testing under version skew

System-first means the linked utf8proc differs across machines, so
the suite is split:

- **Invariants** run in every configuration: cluster width sums,
  zero-width handling, overflow bounds, decode and encode round-trips.
- **Exact goldens** for width and grapheme boundaries run only against
  the pinned download (2.12.0). CI forces `Fetch`, so the canonical
  golden configuration is the pinned one.

The build records `UTF8PROC_VERSION_MAJOR`, `MINOR`, and `PATCH`, and
tests report it, so a failing golden names the Unicode version it ran
against.

## Updates

System dependencies move by raising a floor in a decision ticket; the
pinned download moves by bumping its version and hash in a decision
ticket. Nothing bumps automatically. A dependency update is a
reviewed change with its tests.

## Deferred and out of scope

- Windows and ConPTY: out of scope for v1, which is POSIX only.
- Artifact packaging — wheels, sdists, pkg-config, static versus
  shared builds, and how a released artifact obtains utf8proc without
  a system copy — is not settled here. It stays fog on the map and
  depends on the C ABI page.
- A zero-dependency build (own terminfo reader, in-tree width tables)
  is a possible later option, not a v1 configuration.
