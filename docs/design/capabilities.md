# Capability model and downgrade policy

Normative design for
[Capability model and downgrade policy](https://github.com/rocky-star/orchidee/issues/9).
It fixes what the Compositor believes about a terminal, how it learns
it, what it guarantees when the terminal is unknown, and how it maps
what the application asks for onto what the terminal can show. The
colour-value model is recorded in
[ADR-0002](../adr/0002-tagged-colour-values.md).

Downstream pages refine this one: `terminal-backend.md` performs the
start-up read, `frame-model.md` stores attributes and colours,
`present-pipeline.md` emits them, `input-decoding.md` owns input
protocol negotiation, and `python-palette.md` builds colour schemes on
top of the seam.

## The capability snapshot

The Compositor reads terminfo once at start-up (see
[architecture.md](architecture.md#libtinfo)) and freezes the result
into a **capability snapshot**. The buffer and present paths never
consult terminfo; they consult the snapshot.

### Axes

The snapshot is a typed, versioned struct, not a terminfo-style bag of
strings. Every field is grouped under one of five axes.

- **Colour**: one ordered depth — `Mono`, `8`, `16`, `256`,
  `TrueColor` — plus a numeric colour-pair limit.
- **Attributes**: a support bit for bold, dim, italic, blink, reverse,
  invisible, and strikethrough; and a supported underline style —
  `none`, `single`, `double`, `curly`, `dotted`, or `dashed`.
- **Input**: mouse protocol and encoding, bracketed paste, focus
  events, and the kitty-keyboard protocol flags.
- **Screen modes**: alternate screen, synchronized output, cursor
  shapes, window title, and OSC 52 clipboard.
- **Metrics**: `cols` and `lines`, taken from `TIOCGWINSZ` rather than
  terminfo, with 80x24 assumed when the ioctl fails.

Every field also records its **provenance**: `terminfo`, `env`,
`override`, `fallback`, or `unknown`. Provenance exists so a bug
report can state *why* a value was believed, and it is exposed in
diagnostics and on the seam. Raw terminfo strings stay private to the
terminal backend; nothing terminfo-shaped crosses the seam.

The snapshot is derived data, not a second source of truth: the
terminal is authoritative, and the snapshot is what we resolved at
start-up.

### The capability matrix

| Capability | Learned from | Unknown `$TERM` fallback |
| --- | --- | --- |
| Colour depth | `colors`, `RGB`/`Tc`, `COLORTERM` | 16 |
| Colour pairs | `pairs` | 0 (no pair table) |
| Bold, underline, reverse | `sgr`, `smso`, `rmso`, family | yes |
| Dim, italic, blink, strikethrough | extended `sgr`, family | no |
| Underline styles | `Smulx`, `Su`, `Setulc` | single only |
| Mouse | `XM`, `kmous` | none |
| Bracketed paste | `BD`, `BE` | no |
| Focus events | family, environment | no |
| Alternate screen | `smcup`, `rmcup` | no |
| Synchronized output | family, environment, override | no |
| Cursor shapes | `Ss`, `Se` | no |
| Window title | `tsl`, `fsl` | no |
| OSC 52 clipboard | `Ms` | off |
| Kitty keyboard | family, environment, override | off |
| Size | `TIOCGWINSZ` | 80x24 |

Absence of a terminfo string is not proof of absence of the feature.
`vt100` has no `colors` and no `sgr` string, yet it has `cup` and
renders bold, underline, and reverse; `linux` has `colors=8` and only
the base-8 `setaf`/`setab`. Discovery therefore combines the terminfo
entry with a curated family table, and never infers colour support
from the presence of `setaf` alone.

## Discovery and precedence

Most authoritative first:

1. **Explicit override**: one environment variable, `ORCHIDEE_CAPS`,
   holding a comma-separated `key=value` list. Keys are the capability
   names in the matrix, values are depth names, protocol names, or
   `on`/`off`, for example:
   `ORCHIDEE_CAPS=colors=truecolor,mouse=sgr,keyboard=kitty-1,sync=on,os52=off`.
   An unparsable entry is ignored and recorded as a diagnostic; it
   never aborts start-up.
2. **Terminfo**: the description libtinfo resolved from `$TERM`, the
   terminfo search path, or an inline `hex:`/`b64:` description.
3. **Environment hints**: `COLORTERM` (`truecolor` or `24bit`) may
   *widen* the colour depth when terminfo has no `RGB`; it may never
   narrow a depth terminfo already reports. `COLORTERM` is not
   forwarded by ssh or sudo by default, so it is a hint, not proof.
4. **Built-in fallback tiers**, below.

There is no user configuration file in v1; a config format remains
unspecified on the map. The override variable is a debugging and
escape-hatch surface, not the theming story.

## Built-in fallback tiers

When the terminfo database has no usable entry — unknown `$TERM`, a
missing database, a corrupt entry — the Compositor falls back in two
tiers:

- **Curated table**: compiled-in capability sets keyed by `$TERM`
  family prefix: `xterm`, `screen`, `tmux`, `linux`, `vt100`, `ansi`,
  `alacritty`, `foot`, `wezterm`, `xterm-kitty`, `ghostty`, `cygwin`.
  These carry the widely-deployed extensions the family is known for,
  at the depth the family reliably supports.
- **Strict ANSI/ECMA-48 subset**: for any other non-`dumb` `$TERM`:
  16 colours, bold, underline, reverse, cursor addressing, cursor
  hide/show, and an assumed 80x24. It promises **no** alternate
  screen, mouse, bracketed paste, focus events, OSC 52, window title,
  synchronized output, or kitty keyboard.

`TERM=dumb`, an unset `$TERM`, or a non-terminal stdout is a failure
at backend start, with a diagnostic naming the reason. It is not a
degraded mode: a "UI" with no cursor addressing is not a UI.

## The floor and graceful degradation

Two levels, deliberately separated.

- **Hard floor** — what must work for the Compositor to start at all:
  a POSIX terminal with cursor addressing and SGR for bold, underline,
  and reverse. Colour is **not** part of the floor. `vt100` passes
  (monochrome) and is the reference exemplar; the DOS-era assumption
  that monochrome is a first-class display still holds.
- **Design target** — what widgets and design pages may assume and
  what degrades gracefully: 16 colours, UTF-8, alternate screen,
  mouse, and bracketed paste. A terminal below the target still runs
  the same UI with the missing features dropped by the downgrade
  rules.

80x24 is an assumed size only until the first successful
`TIOCGWINSZ`; there is no enforced minimum window size. A smaller
window is the application's problem to lay out, not a start-up error.

## Rendering depth

Terminal capability and rendering depth are different things. The
snapshot says what the terminal *can* do; the rendering depth says
what the Compositor is *asked* to do right now.

- The depth enum is shared: `Mono`, `8`, `16`, `256`, `TrueColor`.
- The Python layer selects a preset — **4-bit** (16 colours),
  **8-bit** (256 colours), or **24-bit** (`TrueColor`) — because
  terminal palettes vary and an application may deliberately target a
  palette-addressed display.
- The default is the terminal's own depth.
- A request above the terminal's depth is **clamped downward**, never
  an error: 24-bit on a 256-colour terminal renders as 8-bit; on a
  monochrome terminal, as mono. Clamping may land on `8` where the
  terminal reports eight colours.
- The effective depth is readable back, and it is one setting per
  Compositor, not per surface.
- Attribute support is orthogonal to depth: a 24-bit terminal can
  still lack curly underline.

## Colour values at the seam

A colour crossing the seam is **tagged**:

- `Palette(index)`, where index is 0–255; or
- `RGB(r, g, b)`.

The tag is what makes a chosen depth meaningful. Per depth:

- **Mono**: all colour is dropped. Emphasis comes from bold, reverse,
  and underline only.
- **4-bit**: `Palette(0–15)` renders exactly as the terminal's own
  ANSI colour (`SGR 30–37` for the base eight, `90–97` for the bright
  eight). `Palette(16–255)` and `RGB` are mapped to the nearest of the
  assumed 16.
- **8-bit**: `Palette(0–255)` renders exactly through `setaf`/`setab`.
  `RGB` is mapped into the xterm 6x6x6 cube plus 24 greys.
- **24-bit**: `RGB` passes through. `Palette(0–255)` is *still*
  emitted as a palette index, so a theme written in ANSI terms keeps
  the user's palette instead of being frozen to xterm's RGB values.

The alternative — RGB only — would make 4-bit mode a pure colour
conversion and would discard the application's palette intent. It is
rejected in [ADR-0002](../adr/0002-tagged-colour-values.md).

## Colour downgrade algorithm

When a colour must become a palette entry:

1. **Exact match wins**: an `RGB` equal to a palette entry renders as
   that entry, so repeated round-trips do not drift.
2. Otherwise, the **nearest entry in OKLab** (perceptually uniform,
   cheap to compute, cacheable per requested colour). Euclidean RGB
   distance is rejected: it is visibly wrong on blues and purples.
3. The assumed palettes are **compiled-in xterm defaults**: the 16
   ANSI colours and the 256-colour cube plus greys. Without active
   probing the real palette cannot be known, so the assumption is
   documented rather than hidden, and `ORCHIDEE_CAPS` can override
   depth.
4. **No dithering for cell colour**: ordered dithering turns flat UI
   fields into noise, fights damage-rectangle repainting, and breaks
   byte goldens. Dithering is reserved for a future bitmap path
   (Sixel and other image protocols), where it is a different
   problem.
5. **16 → 8**: a bright background folds to its base colour; reverse
   video is preserved. Foreground brightness is kept as bold where
   bold is supported.
6. **Mono**: hue and brightness are dropped; bold, reverse, and
   underline carry all emphasis.
7. Mappings are **cached** for the session keyed by
   `(requested colour, depth, palette)`, so present-time cost is a
   lookup.

## Attribute downgrade

Unsupported attributes map within their family and drop decoration;
none is synthesised from colour, and no ambiguous sequence is emitted.

| Requested | When unsupported |
| --- | --- |
| Underline `double`, `curly`, `dotted`, `dashed` | single underline if `SGR 4` works, else nothing |
| Italic | underline when underline is available, else nothing |
| Bold | drop (it is in the floor; absence means the fallback tier is already wrong) |
| Dim | drop unless `SGR 2` is known to work; never rendered as bold |
| Blink | drop; v1 has no flashing text |
| Strikethrough | drop |
| Invisible | foreground set equal to background |
| Reverse | drop only on a terminal that fails the floor |

`SGR 21` is **never emitted**: ECMA-48 defines it as double underline,
several terminals treat it as bold-off, and the ambiguity is not
worth the risk. Extended underline styles come from `Smulx` or are
downgraded.

Downgrades are silent in the UI — no one-time warning on stderr,
which would corrupt the screen — and are visible only in diagnostics.

## Ownership and exposure

The Compositor owns the downgrade algorithm. The Python layer always
supplies tagged colours and abstract attributes and never branches on
terminal type; the seam exposes the capability snapshot read-only,
with provenance, so widgets can still make asset-level choices (box
drawing versus ASCII borders, compact labels). Putting downgrade logic
in a per-language layer would duplicate it and let two layers disagree
about the same terminal.

## Deferred

Recorded here so later tickets do not re-litigate them:

- **Active probing** of the terminal (`DA1`, `XTGETTCAP`) for
  capabilities terminfo cannot express: not in v1. If the opt-in
  friction of synchronized output or kitty keyboard bites, it returns
  as a host-driven pre-start step with a bounded read deadline, not
  as I/O inside the Compositor.
- **Runtime palette setter**: a seam call telling the Compositor the
  terminal's real palette. Not needed for correctness, because
  palette-addressed colours are never converted; additive later.
- **Bitmap dithering**: belongs with the Sixel and image-protocol
  work.
- **User configuration file**: remains unspecified on the map.
