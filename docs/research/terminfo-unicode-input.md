# Terminfo, Unicode width and input parsing: building blocks

Research note for
[ticket #3](https://github.com/rocky-star/orchidee/issues/3). It records
facts and costs; it does not make the decision.

- Date: 2026-11-20
- Status: input to an ADR; no code changed.
- Question: which concrete building blocks should the Compositor use for
  terminfo access, Unicode width, and input sequence parsing, given an
  MPL-2.0 project, a passive loop-agnostic C++ core with a C ABI, POSIX
  terminals, and terminfo as primary capability source with a built-in
  ANSI fallback subset?

## Method

Claims below come from primary sources: the upstream headers and licence
files, the ncurses manual pages, the Unicode Standard Annexes, and the
MPL-2.0 text. Where a number was measured, the measuring command or
method is stated. Measurements were taken on an Arch Linux x86-64
machine with ncurses 6.6.20251230 (`infocmp -V`), 2,900 compiled
terminfo entries and a 7.3 MB `/usr/share/terminfo`; utf8proc 2.12.0 was
installed locally (`/usr/include/utf8proc.h`). Candidate sources were
shallow-cloned at their latest release to measure size.

Terminology used below:

- **passive**: the library performs no blocking read, owns no event
  loop, writes nothing to the terminal unless asked, and keeps no
  process-global terminal state that the caller cannot reset.
- **vendoring**: copying the dependency's source into the Orchidée tree
  and building it as part of the project, as opposed to linking a system
  library.

## 1. Terminfo access

### What this layer actually needs

1. Resolve a terminal description from `$TERM` (and overrides) and from
   the compiled database search path.
2. Fetch a small capability set: cursor addressing (`cup`), SGR and
   colour (`sgr`, `setaf`, `setab`, `colors`, `op`), screen switching
   (`smcup`, `rmcup`), cursor visibility (`civis`, `cnorm`), scroll
   region (`csr`), insert and delete (`ich`, `dch`, `il`, `dl`), `ech`,
   `rep`, `bce`, `am`, `xn`, `it`, `lines`, `cols`; input capabilities
   (`kcuu1` and friends, `kbs`, `kf1`...); and the extended `RGB`/`Tc`
   true-colour markers.
3. Expand parameterised strings (`cup`, `sgr`, `setaf`) with the
   terminfo `%`-format language.
4. Degrade to a built-in ANSI subset when no description is available.

All four are CPU-only work on data; none requires the library to own the
terminal. The interesting differences between candidates are licence,
global state, and whether a database must exist at run time.

### libtinfo / ncurses

Licence: MIT-style X11 grant in the ncurses `README`, not the GPL
("Permission is hereby granted, free of charge ..."). ncurses is
actively maintained by Thomas E. Dickey; 6.6 is current.

The interface is the classic terminfo one, documented in
[curs_terminfo(3X)](https://invisible-island.net/ncurses/man/curs_terminfo.3x.html):

- `int setupterm(const char *term, int fd, int *errret);` returns
  `OK`/`ERR` and reports `errret` 1 (hard-copy terminal), 0 (terminal
  not found or generic), -1 (terminfo database not found). If `errret`
  is `NULL`, `setupterm` prints an error and *exits*, which is unusable
  in a library.
- `char *tigetstr(const char *cap);` returns `(char *)-1` when the name
  is not a string capability and `NULL` when the capability is cancelled
  or absent: the caller must distinguish these two failures.
- `tigetflag`, `tigetnum`, `tparm` (also `tiparm`), `del_curterm`,
  `set_curterm`, `restartterm`.
- A description may be supplied inline: if `$TERMINFO` starts with
  `hex:` or `b64:`, ncurses decodes the entry from that value and uses
  it if the name matches, which is useful for overrides and tests.

Database resolution, from
[terminfo(5)](https://invisible-island.net/ncurses/man/terminfo.5.html):
`$TERMINFO`, then `$HOME/.terminfo`, then `$TERMINFO_DIRS` (an empty
element means `/usr/share/terminfo`), then the compiled-in list
(`/usr/share/terminfo`). ncurses reads the first description that passes
its consistency checks. The format is ncurses-compatible; the string
table has a 4,096-byte limit, and extended capabilities are stored after
the classic sections
([`use_extended_names(3X)`](https://invisible-island.net/ncurses/man/use_extended_names.3x.html),
[terminfo(5) EXTENDED CAPABILITIES](https://invisible-island.net/ncurses/man/terminfo.5.html)).
ncurses treats user-defined string capabilities whose names begin with
`k` as function keys, so a data-driven key table must accept those.

Passivity problems: the API is built around one mutable global
`TERMINAL`/`cur_term`; `set_curterm` and `del_curterm` swap it, and
[curs_threads(3X)](https://invisible-island.net/ncurses/man/curs_threads.3x.html)
states that even its reentrant functions are "not enough to make an
application using curses thread-safe". The real cost, though, is the
`setupterm` side effect set: it resolves the description, may consult
the tty for size, and leaves the global state alive until `del_curterm`.

Footprint: `/usr/lib/libncursesw.so.6.6` is 544 KB here, and
`libtinfo.so.6` is a symlink to it on Arch, so linking `-ltinfo` pulls
the whole wide curses object on this distro; some others ship a
separate, smaller `libtinfo`. The database itself is 7.3 MB for 2,900
entries. Distribution coverage is excellent: terminfo is the POSIX-era
mechanism and ncurses is the reference implementation shipped by
essentially every Linux and BSD distribution.

Recommendation shape: excellent *database access* primitive; the fix for
passivity is to read once at start-up into an Orchidée-owned snapshot
and immediately `del_curterm`, then never call the library from the
render path.

### unibilium

[unibilium](https://github.com/neovim/unibilium) (`neovim/unibilium`
fork; the original project "was abandoned", per
[neovim/neovim#10302](https://github.com/neovim/neovim/issues/10302)).

Licence: LGPL-3.0-or-later. The `LICENSE` file says so explicitly and
invites the reader to contact the author "to negotiate other licensing
terms".

API (from `unibilium.h`): `unibi_from_env()`, `unibi_from_file()`,
`unibi_from_mem()`, `unibi_get_bool/num/str()`, extended accessors,
`unibi_format()` for parameter expansion, `unibi_dump()` to
re-serialise, `unibi_del()`. No file descriptor, no curses, no terminal
writes, and the README states it has no global variables and is
thread-safe. That is almost exactly the shape a passive core wants, and
it reads and writes ncurses-style entries including the extended
section. The caller still locates the database: `unibi_from_env()`
honours `$TERM` and `$TERMINFO`-style paths, but there is no bundled
database.

Maintenance is the weak point: it is a small fork with a small team;
latest tag is v2.1.4 and Neovim now marks it deprecated and optional.
Neovim's
[BUILD.md](https://github.com/neovim/neovim/blob/master/BUILD.md#build-without-unibilium)
says: "Unibilium is the only dependency which is licensed under LGPLv3
... used for loading the terminfo database at runtime, and can be
disabled if the internal definitions for common terminals are good
enough." That sentence is also a design precedent for Orchidée's
fallback tier.

Footprint: `unibilium.c` is 38 KB plus a 20 KB generated names table;
the three core sources total about 2,250 lines.

Verdict: technically the best-fitting API for a passive core, but the
LGPL-3.0 licence is a real problem for vendoring into an MPL-2.0 project
(see section 4), and the project is in maintenance rather than active
development.

### Own terminfo reader

The binary format is documented and stable
([terminfo(5)](https://invisible-island.net/ncurses/man/terminfo.5.html)):
a header with magic `0x11 0x11` (16-bit) or `0x21 0x12` (32-bit ncurses
extension), followed by the names section, booleans, numbers, string
offsets and the string table, then an optional extended section. Reading
it is perhaps 300-400 lines; the real work is the `%`-format interpreter
(`%p1%d`, `%i`, `%+`, `%?`...`%;`, `%l`, padding) used by `tparm`, which
needs its own conformance tests against ncurses' `tparm`. The reader
also inherits the "database must exist" problem: on a minimal container
or a static musl build there may be no `/usr/share/terminfo` at all, so
a compiled-in table is still required. Cost: roughly 600-1,000 lines of
C++ plus fixtures, and a permanent second implementation to keep honest.

### Compiled-in capability table

Precedent: Neovim ships internal definitions for common terminals and
can be built with `-DENABLE_UNIBILIUM=0`; terminfo lookups then fail
over to those definitions (`has('terminfo') == 0`). The upstream
rationale is quoted above. For Orchidée this is not a competitor to a
terminfo reader but the *second tier* of the fallback: a curated table
(say 20-60 common `$TERM` values) generated from `terminfo.src`, plus a
synthetic ANSI entry. The cost is a generator, a review burden when
`terminfo.src` changes, and a bounded but real maintenance tax; the
benefit is that unknown terminals and database-less systems still get
correct behaviour.

### Fallback structure

Three tiers, tried in this order:

0. **Compiled-in curated table**, keyed by `$TERM`, generated from the
   ncurses `terminfo.src`. Handles minimal containers and database-less
   builds.
1. **Built-in ANSI subset** (ECMA-48 and DEC sequences) as a synthetic
   entry, used when `$TERM` is unknown or absent. This tier must be the
   default answer, not an error path: it is what makes the Compositor
   usable on an unknown terminal.
2. **Runtime terminfo database** via libtinfo (or an own reader later).

An override layer sits on top: explicit `ORCHIDEE_*` environment
variables (terminal name, database paths, individual capability
overrides) applied after normal resolution, plus test hooks that inject
a synthetic description (ncurses' `hex:`/`b64:` `$TERMINFO` gives one
such hook for free).

Detection facts:

- `$TERM` is the primary selector and is set by the terminal emulator
  (ncurses itself relies on it: terminfo(5)).
- tmux overrides `$TERM` for its panes. Current tmux picks
  `default-terminal` from the first of `tmux-256color`, `tmux`,
  `screen-256color`, `screen` available at build time
  ([tmux CHANGES](https://github.com/tmux/tmux/blob/master/CHANGES)). On
  this machine the database has `tmux-256color` (no `RGB`) and
  `tmux-direct` (with `RGB`), verified with `infocmp -x -1`. So inside
  tmux, terminfo truthfully reports no direct colour unless the outer
  terminal is described and tmux is configured to pass it through;
  colour depth detection must therefore combine terminfo with
  `COLORTERM`.
- `COLORTERM` is the de-facto 24-bit signal: VTE, Konsole and iTerm2 set
  `COLORTERM=truecolor`, S-Lang accepts `truecolor` or `24bit`
  ([termstandard/colors](https://github.com/termstandard/colors)).
  Terminfo has carried the `RGB` capability since ncurses 6.0-20180121
  (same source). The same source notes `COLORTERM` is not forwarded by
  default through ssh or sudo, so it is a hint, not a guarantee. Policy:
  prefer terminfo `RGB` when the description has it, else `COLORTERM`,
  else assume 256 colours if `colors#256`, else 16 or 8.
- User overrides belong in the same resolution chain: `$TERM` override,
  `$TERMINFO`/`$TERMINFO_DIRS` (honoured automatically by libtinfo), and
  Orchidée-specific per-capability overrides for debugging.

## 2. Unicode width and clusters

### What cursor positioning needs

The Compositor must know, for every cell it draws, how many columns the
glyph occupies and how many bytes or code points form it. That is two
different questions: **segmentation** (where a user-perceived character
ends) and **width** (how many columns it covers). Cursor positioning is
a column count, so width alone is not enough: a terminal moves by cells,
but the library must know the cell count of a *cluster*, then advance.

### wcwidth(3)

POSIX Issue 8
([wcwidth](https://pubs.opengroup.org/onlinepubs/9799919799/functions/wcwidth.html))
specifies a single `wchar_t`: returns 0 for a null wide character, the
number of column positions occupied, or -1 if the argument is not
printable; the return value for non-printable wide characters was left
unspecified in ISO C. Properties:

- Locale-dependent, so the result depends on the current `LC_CTYPE`.
- Operates on one code point; there is no cluster notion, so combining
  marks, variation selectors, ZWJ and regional indicators are separate
  calls. `wcswidth` sums them, which is exactly wrong for emoji ZWJ
  sequences and flags.
- The Unicode data version is whatever the C library shipped with, which
  the application cannot query; implementations differ on East Asian
  Ambiguous width.

The reference table by Markus Kuhn
([wcwidth.c](https://www.cl.cam.ac.uk/~mgk25/ucs/wcwidth.c)) is the base
of many implementations, and magiblot/tvision uses the libc function on
Unix and a copy of Kuhn's ranges for the Linux console, which does not
support zero-width characters
([platfchr.cpp](https://github.com/magiblot/tvision/blob/master/source/platform/platfchr.cpp)).

Verdict: usable as a fallback for single code points, not as the primary
width source, because it cannot be fixed or versioned by Orchidée.

### utf8proc

[utf8proc](https://juliastrings.github.io/utf8proc/) is MIT ("expat")
plus the permissive Unicode data licence, version 2.12.0, Unicode 18.0.
[`utf8proc.h`](https://github.com/JuliaStrings/utf8proc/blob/master/utf8proc.h)
provides:

- `utf8proc_iterate()` and `utf8proc_encode_char()`: UTF-8 decode and
  encode.
- `utf8proc_charwidth()`: "a character width analogous to
  `wcwidth(codepoint)`, except that a width of 0 is returned for
  non-printable codepoints instead of -1";
  `utf8proc_charwidth_ambiguous()` reports East Asian Ambiguous.
- `utf8proc_grapheme_break_stateful()` with the `UTF8PROC_CHARBOUND`
  option, which carries the state needed for UAX #29 rules including
  emoji ZWJ sequences and Indic conjunct breaks (`utf8proc.h` keeps an
  `indic_conjunct_break` component in its boundary state).

Footprint: `utf8proc_data.c` is 2.4 MB of source tables; the installed
`libutf8proc.so.3.3.0` is 366 KB. It is used for Unicode support in
Julia and is a Neovim dependency, so its maintenance and Unicode-update
cadence are credible.

### libgrapheme

[libgrapheme](https://github.com/FRIGN/libgrapheme) (upstream:
[git.suckless.org](https://git.suckless.org/libgrapheme/)), by Laslo
Hunhold, ISC licence, Unicode 17.0.0. It implements UAX #29 grapheme
cluster, word, sentence and line-break segmentation plus case detection
and conversion, is freestanding C99 (no libc at run time), and ships
generated compressed tables with conformance tests. Its `README` claims
it is "way smaller and much faster than the other established Unicode
string libraries (ICU, GNU's libunistring, libutf8proc)".

The decisive fact: it has **no width API**. `grep width grapheme.h`
returns nothing; the API is `grapheme_next_character_break_utf8()` and
similar. It solves segmentation only, so it is a complement to a width
source, not a replacement; using it means also owning a width table or
keeping libc `wcwidth`.

Footprint: about 144 KB of C sources in `src/`; the 23 MB `data/`
directory is Unicode data used only to regenerate tables.

### Own width table

Deriving widths from `EastAsianWidth.txt` plus general categories is
mechanical but needs a generator, a per-Unicode-release refresh, and
decisions on Ambiguous width. It is the only option that lets Orchidée
pin a Unicode version independently of the OS, and the only one that can
be shipped without an external dependency. Budget: a generator of a few
hundred lines plus generated tables of order 100 KB.

### Clusters, emoji and ZWJ

Cursor positioning must use UAX #29 *extended grapheme clusters*
([UAX #29](https://www.unicode.org/reports/tr29/)); the relevant rules
are GB9/GB9a (combining marks, spacing marks, ZWJ), GB9c (Indic
conjunct), GB11 (emoji ZWJ sequence) and GB12/GB13 (regional indicator
pairs, that is, flags). Emoji sequences are defined in
[UTS #51](https://www.unicode.org/reports/tr51/).

Practical rule for a terminal library: compute width over the cluster,
not over each code point. A cluster containing U+FE0F (variation
selector-16) or a ZWJ sequence is normally two columns even though its
parts sum to something else; a leading combining mark has width 0 but
still needs a cell; regional-indicator pairs render as one two-column
flag. Terminals disagree with each other here, so the policy should be
explicit and overridable rather than buried.

### Assessment

`utf8proc` is the only single dependency that covers decode, cluster
boundaries and width in one MIT-licensed package, which is why the
minimal set leans on it. `libgrapheme` plus an own width table is a
viable strictly-smaller-footprint alternative, at the cost of writing
and maintaining the width table. libc `wcwidth` is the fallback path for
single code points when nothing else is available, and it must not be
the primary source because it cannot be versioned or fixed.

## 3. Input sequence parsing

### What input parsing needs

- Incremental, resumable parsing of a byte stream: a sequence may be
  split arbitrarily across reads.
- No blocking: the caller owns the loop, so the parser must accept bytes
  and return events, with the ability to say "need more bytes".
- UTF-8 decoding, plain text, ESC, SS3, CSI (with numeric parameters,
  `;`, `:` separators and intermediate bytes), DCS, OSC, APC.
- Legacy function keys defined by terminfo (`kcuu1`, `khome`, `kf1`...),
  which are terminal-specific strings.
- Modifiers: xterm `modifyOtherKeys` and the kitty keyboard protocol's
  CSI-u form
  (`CSI unicode-key-code:alternate-key-codes ; modifiers ; text u`),
  including its progressive-enhancement flags pushed with
  `CSI = flags ; mode u`
  ([kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)).
- Mouse: X10 and SGR encodings, plus bracketed paste, focus reports, and
  device or CPR replies that must not be mistaken for keys.

### libtermkey

[libtermkey](https://github.com/neovim/libtermkey) (Paul Evans; MIT
licence). Latest release v0.22; the repository is **archived** (last
commit 2023-12-13: "Add note before archiving") and the README says
"Libtermkey is now vendored in Neovim".

API shape, from
[termkey(7)](https://github.com/neovim/libtermkey/blob/master/man/termkey.7)
and
[termkey_new(3)](https://github.com/neovim/libtermkey/blob/master/man/termkey_new.3):

- `termkey_new(fd, flags)` for a file handle;
  `termkey_new_abstract(term, flags)` for no handle at all;
  `termkey_push_bytes()` feeds bytes directly; `termkey_getkey()` reads
  a buffered event without blocking; `termkey_advisereadable()`
  integrates with an external loop. So the library *can* be driven
  passively.
- Event types include Unicode codepoint, function key, keysym, mouse,
  position report, mode report and DCS, with `termkey_interpret_*`
  helpers.
- Two drivers are carried in-tree: `driver-csi.c` (generic CSI) and
  `driver-ti.c` (terminfo-based). The ti driver needs a terminfo
  description, which duplicates Orchidée's terminfo concern.
- `TERMKEY_CHECK_VERSION` exists because the ABI has moved between
  versions; the API allocates an opaque `TermKey` object and hides its
  buffer.

Costs: the library predates the kitty keyboard protocol and current
extended-modifier conventions, so a modern terminal UI would need
extensions anyway; it is archived; and its terminfo driver couples it to
the same dependency question. magiblot/tvision solves the same problem
with its own parser instead: `source/platform/termio.cpp` contains
`parseEscapeSeq`, `parseCSIKey`, `parseX10Mouse`, `parseSGRMouse`,
`parseDCS`, `parseOSC` and `parseCPR`, and enables kitty's `ESC[>5u`
enhancement
([termio.cpp](https://github.com/magiblot/tvision/blob/master/source/platform/termio.cpp)).
Neovim took the middle path: it vendored libtermkey and wrapped it in
its own `src/nvim/tui/input.c` layer.

### Own state machine, or hybrid

A hand-written parser needs roughly 800-1,200 lines plus a table of test
vectors (real terminal capture is the only reliable oracle). The hybrid
variant is the realistic one: parse ESC, SS3 and CSI generically in
Orchidée, and resolve *named* keys (`khome`, `kbs`, `kf1`...) through a
table built from the terminfo capabilities plus the compiled-in fallback
table, with longest-match against the current prefix. That removes
libtermkey's terminfo-driver duplication and keeps the parser total (a
parser that must return "incomplete" rather than guess is much easier to
test), and it is what both reference implementations effectively do.

### Assessment

Own incremental state machine with a data-driven key table; use
libtermkey only as a reference and as a source of test cases (MIT
permits that with attribution). Budget a dedicated conformance suite for
ESC prefix ambiguity (ESC alone versus Alt+key versus CSI), because that
is where hand-written parsers break.

## 4. Licence compatibility with MPL-2.0

MPL-2.0 is file-level copyleft: modified MPL-covered files must be made
available, but the project may combine them into a "Larger Work" under
other terms
([MPL 2.0 §3.3](https://www.mozilla.org/media/MPL/2.0/index.815ca599c9df.txt)).
Section 3.3's Secondary Licences (GPLv2+, LGPLv2.1+, AGPLv3+) exist so
MPL code can be *distributed* inside a GPL work; the
[FSF licence list](https://www.gnu.org/licenses/license-list.html#MPL-2.0)
describes it as "indirect compatibility". That clause is not a
permission to absorb LGPL-3.0 code into MPL-covered files.

| Dependency | Licence | Usable in an MPL-2.0 project? | Notes |
| --- | --- | --- | --- |
| ncurses / libtinfo | MIT-style (X11) | Yes, link or vendor | Keep the notice. |
| unibilium | LGPL-3.0-or-later | Dynamic linking only, or negotiate | Static linking and vendoring trigger LGPL relink and notice duties; the author explicitly invites negotiation. |
| libtermkey | MIT | Yes | Any use must keep the MIT notice; archived, so vendored code becomes ours to maintain. |
| utf8proc | MIT plus Unicode data licence | Yes | Data licence is permissive; keep `LICENSE.md`. |
| libgrapheme | ISC | Yes | Keep the notice. |

Practical policy that follows: permissive dependencies (MIT, ISC, X11)
may be vendored inside the tree under their own files and notices; LGPL
dependencies may only be an optional, dynamically linked build option,
as Neovim does with `-DENABLE_UNIBILIUM=0`, and should default to off.
If unibilium is ever wanted, ask its author for a permissive licence
first; the `LICENSE` file explicitly offers that route.

## 5. Maintenance health

| Dependency | Latest release | Activity | Risk |
| --- | --- | --- | --- |
| ncurses / libtinfo | 6.6 (20251230 snapshot) | Active, Thomas Dickey; shipped by every major distro | Low |
| unibilium | v2.1.4 | Small fork, original abandoned; Neovim marks it deprecated and optional | Medium |
| libtermkey | v0.22 | Archived 2023-12-13; vendored by Neovim | High as a dependency |
| utf8proc | 2.12.0, Unicode 18.0 | Active (JuliaStrings) | Low |
| libgrapheme | 3.0.0, Unicode 17.0 | Active (suckless project) | Low |

## 6. Minimal set

Recommended default: **two external dependencies**, both permissive.

1. **terminfo**: libtinfo (system, MIT-style), behind an Orchidée-owned
   capability struct. Read once at start-up into the snapshot, then
   `del_curterm`. Build the fallback tiers (compiled-in curated table,
   then ANSI subset) as first-class code, and keep `$TERMINFO`, `hex:`
   and `b64:` support as the test and override hook.
2. **Unicode width**: utf8proc (MIT), for UTF-8 decode, UAX #29 grapheme
   boundaries and `utf8proc_charwidth()`, with a thin Orchidée width
   policy on top (cluster width, ambiguous-width policy, overridable
   table).
3. **Input parsing**: own incremental state machine, data-driven key
   table from terminfo plus fallback; no libtermkey dependency.

Deferred or optional:

- **unibilium**: rejected for the default build on licence grounds;
  revisit only if a permissive relicence is negotiated.
- **libtermkey**: not a dependency; use as reference and test oracle.
- **libgrapheme**: alternative to utf8proc if the width table is
  implemented in-tree; would then be a third dependency (or replace
  utf8proc's segmentation, with width still needed).
- **Own terminfo reader**: a later zero-dependency build option; it
  removes libtinfo but adds ~600-1,000 lines plus fixtures.
- **Compiled-in table only**: viable for a minimal build, but a curated
  table cannot be the primary terminfo source without accepting coverage
  loss.

Cost to vendor each candidate, measured:

| Candidate | Source size | Compiled size (this machine) | Vendor cost |
| --- | --- | --- | --- |
| libtinfo | Part of ncurses tree | 544 KB `libncursesw.so.6.6`; 7.3 MB database | Do not vendor; system dependency |
| unibilium | ~2,250 lines, 58 KB | not built here | Small, but LGPL blocks it |
| libtermkey | 3,275 lines of `.c` | not built here | Small, but archived and stale protocols |
| utf8proc | 2.4 MB data plus core | 366 KB `.so` | Small; MIT; generated data can be regenerated per Unicode release |
| libgrapheme | 144 KB `src/`, 23 MB `data/` (generation only) | not built here | Small; ISC; tables regenerated from `data/` |

The minimal set costs roughly 600 KB of runtime library plus the
terminfo database at run time, and leaves the option of a
zero-dependency build later without changing the core architecture.

## 7. What this changes for Orchidée

- The dependency question is nearly closed by licences: the only strong
  candidate that is not permissive is unibilium (LGPL-3.0), and the
  permissive alternatives cover every need. Record a dependency policy
  (permissive only, LGPL at most optional and dynamically linked) before
  writing code.
- The Compositor must snapshot capabilities rather than call terminfo
  from the render path: libtinfo's global `cur_term` and non-reentrancy
  are the one place where "passive" is threatened. The C ABI should
  expose an opaque capabilities handle with no global mutable state, and
  `del_curterm` immediately after snapshotting.
- The built-in ANSI fallback is not a corner case: it is the default for
  unknown `$TERM` and database-less systems, so it needs its own curated
  table and the same test coverage as the terminfo path, plus a
  documented resolution order for `$TERM`, `$TERMINFO*`, tmux and
  `COLORTERM`.
- Unicode width must be cluster-based and Unicode-versioned. libc
  `wcwidth` and `wcswidth` cannot express emoji ZWJ or flag clusters, so
  the width policy (including the Ambiguous class) needs to be an
  explicit, overridable Orchidée component even if it is backed by
  utf8proc.
- Input parsing should be an Orchidée-owned incremental state machine
  from day one. libtermkey is archived, predates kitty CSI-u and
  extended modifiers, and would duplicate the terminfo dependency; the
  parser must return "incomplete" and be tested against captured
  sequences.
- Consequence for ticket #3: adopt the minimal set (libtinfo plus
  utf8proc plus an own input parser, with a compiled-in fallback), note
  the deferred options, and open a follow-up ADR for the
  capability-snapshot ABI and the width policy.

## References

- [Ticket #3](https://github.com/rocky-star/orchidee/issues/3)
- ncurses: [project page](https://invisible-island.net/ncurses/),
  [README licence](https://github.com/ThomasDickey/ncurses-snapshots/blob/master/README),
  [terminfo(5)](https://invisible-island.net/ncurses/man/terminfo.5.html),
  [curs_terminfo(3X)](https://invisible-island.net/ncurses/man/curs_terminfo.3x.html),
  [curs_threads(3X)](https://invisible-island.net/ncurses/man/curs_threads.3x.html)
- [unibilium](https://github.com/neovim/unibilium)
- [neovim/neovim#10302 (abandoned upstream)](https://github.com/neovim/neovim/issues/10302),
  [Neovim BUILD.md](https://github.com/neovim/neovim/blob/master/BUILD.md#build-without-unibilium),
  [Neovim MAINTAIN.md](https://github.com/neovim/neovim/blob/master/MAINTAIN.md)
- [libtermkey (archived)](https://github.com/neovim/libtermkey),
  [termkey(7)](https://github.com/neovim/libtermkey/blob/master/man/termkey.7),
  [termkey_new(3)](https://github.com/neovim/libtermkey/blob/master/man/termkey_new.3)
- [utf8proc](https://juliastrings.github.io/utf8proc/),
  [utf8proc.h](https://github.com/JuliaStrings/utf8proc/blob/master/utf8proc.h)
- [libgrapheme](https://github.com/FRIGN/libgrapheme),
  [upstream git](https://git.suckless.org/libgrapheme/)
- POSIX:
  [wcwidth](https://pubs.opengroup.org/onlinepubs/9799919799/functions/wcwidth.html)
- Unicode: [UAX #29](https://www.unicode.org/reports/tr29/),
  [UTS #51](https://www.unicode.org/reports/tr51/)
- [MPL 2.0](https://www.mozilla.org/en-US/MPL/2.0/),
  [MPL 2.0 section 3.3 text](https://www.mozilla.org/media/MPL/2.0/index.815ca599c9df.txt),
  [MPL FAQ](https://www.mozilla.org/en-US/MPL/2.0/FAQ/),
  [FSF licence list](https://www.gnu.org/licenses/license-list.html#MPL-2.0)
- [termstandard/colors (COLORTERM)](https://github.com/termstandard/colors)
- [tmux CHANGES](https://github.com/tmux/tmux/blob/master/CHANGES)
- [kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)
- magiblot/tvision (reference copy under `tvision/`):
  [platfchr.cpp](https://github.com/magiblot/tvision/blob/master/source/platform/platfchr.cpp),
  [termio.cpp](https://github.com/magiblot/tvision/blob/master/source/platform/termio.cpp),
  [README](https://github.com/magiblot/tvision/blob/master/README.md)
  (`-lncursesw`, `wcwidth` note)
- Markus Kuhn's reference width table:
  [wcwidth.c](https://www.cl.cam.ac.uk/~mgk25/ucs/wcwidth.c)
