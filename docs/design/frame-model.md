# Cell, attribute, and frame model

Normative design for
[Cell, attribute, and frame model](https://github.com/rocky-star/orchidee/issues/12).
It fixes what the Compositor stores and compares between draws: the
cell, how text and style are packed into it, what a frame is, and the
invariants presentation may rely on.

Colour meaning and downgrade belong to
[capabilities.md](capabilities.md) and
[ADR-0002](../adr/0002-tagged-colour-values.md); sanitisation and the
binding's text obligations to
[#19](https://github.com/rocky-star/orchidee/issues/19); segmentation
and width policy to
[#15](https://github.com/rocky-star/orchidee/issues/15); the bytes a
frame becomes to
[#13](https://github.com/rocky-star/orchidee/issues/13). This page
fixes the data, not the emission.

## The cell

A cell is one column of the screen grid, or the first column of a glyph
two columns wide. Every cell occupies the same 16 bytes:

| Word | Contents |
| --- | --- |
| `text` | flags byte plus 7 bytes of inline UTF-8, or a 56-bit pool id |
| `style` | tagged foreground, tagged background, abstract attributes |

The layout is internal and free to change; nothing cell-shaped crosses
the seam. The seam's drawing calls take fields — a string and a style —
never cells.

### The text word

The flags byte holds:

- 3-bit **inline length**: 1–7 bytes in the low 56 bits; 0 means the low
  56 bits are a **pool id**;
- 2-bit **width**: 1 or 2 on a text-bearing cell, 0 on a continuation
  cell;
- **continuation**: set on the second cell of a wide glyph;
- two spare bits, zero.

A continuation cell's text is empty and identified by its flag before
any length is read, so the zero word is not a pool reference; its style
copies the glyph's. Pool id 0 is invalid.

A cell always holds one complete grapheme cluster. Space is an ordinary
cluster: the blank cell is a space in the clear style, not a special
state.

### The style word

| Field | Bits | Contents |
| --- | --- | --- |
| foreground | 26 | 2-bit tag plus 24-bit payload |
| background | 26 | 2-bit tag plus 24-bit payload |
| attributes | 10 | seven flags plus 3-bit underline style |

The colour tags are:

- `0` **Palette**, payload 0–255, so the user's palette is honoured;
- `1` **RGB**, payload 8:8:8;
- `2` reserved for a terminal-default colour, rejected in v1;
- `3` invalid.

The seven attribute flags are bold, dim, italic, blink, reverse,
invisible, and strikethrough. The underline style is none, single,
double, curly, dotted, or dashed. Underline colour, and every other
extended attribute, are not in v1.

The style word stores the *request*, not the terminal's rendering:
colours stay tagged and attributes abstract, so changing the rendering
depth is a cache flush at present time, never a rewrite of the grid.
Reverse and invisible are resolved when the bytes are built
([#9](https://github.com/rocky-star/orchidee/issues/9)). There are no
semantic roles — "selected", "disabled" — in the core: a per-language
layer maps its theme onto colours and attributes and never branches on
terminal type.

Two spare bits stay zero, so a style word with either set is invalid and
can take part in the unknown sentinel.

### Equality

Two cells are equal when both words are equal. Because the pool dedupes
by content, equal text always carries the same id, so word equality is
content equality and the diff never compares strings.

## Text placement

Text arrives as UTF-8 spans. The Compositor sanitises, segments with
utf8proc (UAX #29), computes each cluster's **display width** by policy,
and places clusters one at a time. Sanitisation is
[#19](https://github.com/rocky-star/orchidee/issues/19)'s; segmentation
and the width policy are shared with
[#15](https://github.com/rocky-star/orchidee/issues/15). This page fixes
what the grid can hold.

### Wide clusters

A cluster of width 2 needs two adjacent columns inside the target clip:

- both cells are written; the first carries the text, the second is a
  continuation cell with the same style;
- if only one column is available, the cluster is written as a space in
  the span's style at that column, and no cell beyond it is touched;
- a write that lands on either half of a wide glyph replaces the *other*
  half with a space in that half's existing style.

A half glyph therefore cannot survive in a grid by accident. Surface
clipping ([#11](https://github.com/rocky-star/orchidee/issues/11)) applies
the same rule at its boundary.

### Width-0 clusters

A cluster can compute to width 0: a combining mark with no base, a stray
variation selector, or a joiner — usually at the start of a span, or
because a caller sliced UTF-8 mid-cluster. Such a cluster never becomes
a cell. The rules, in order:

1. **Fold left.** If a cell exists to the left inside the same clip,
   the cluster is appended to that cell's text — a continuation folds
   into its lead — provided the merged cluster's width is unchanged. A
   stray variation selector could otherwise widen the base, and that
   fold is refused.
2. **Lone mark.** With no cell to fold into, a cluster containing a
   combining mark (Mn or Me) is stored as `U+25CC` DOTTED CIRCLE
   followed by the cluster, in the span's style. The dotted circle is
   the conventional base for displaying an isolated mark, so the mark
   stays visible and keeps its identity.
3. **Format only.** With no cell to fold into, a cluster of only format
   characters — variation selectors, ZWJ, ZWNJ, tag characters — is
   dropped and counted in diagnostics. A circle around an invisible
   joiner would show nothing and still claim a column.

The dotted circle is applied by the Compositor, in the same cluster walk
that advances the pen; bindings never inspect clusters. Because the
circle occupies a column, the **display width** of such a cluster is 1,
and the measure function reports that. Measurement is the single source
of truth for layout: a caller that measures the same text with raw
utf8proc or `wcwidth` elsewhere will be one column short on that line.

### Frame invariants

A frame that is ready to present satisfies:

- every text-bearing cell has width 1 or 2, and every continuation cell
  is flagged as one;
- every width-2 lead is immediately followed by its continuation in the
  same row;
- every continuation is immediately preceded by a width-2 lead;
- no cell holds a width-0 cluster.

Composition can break the last two: a top surface may cover one half of
a glyph drawn by a lower one. Before a frame is presentable, any lead
whose continuation is not the next cell becomes a space in its own
style, and so does any continuation without a lead. Each normalisation
is counted in diagnostics, because it means a widget's clip arithmetic
was wrong. Clipping a wide cluster at a write boundary is not a
normalisation: it happens before the cells are written.

## The frame

A frame is the complete grid of cells composed for the screen at one
commit point, plus the cursor state. It is internal; begin, commit, and
present belong to the drawing and present tickets
([#19](https://github.com/rocky-star/orchidee/issues/19),
[#13](https://github.com/rocky-star/orchidee/issues/13)).

The Compositor keeps two grids:

- the **front grid**, exactly what the terminal is believed to show;
- the **back grid**, what the application has composed.

Present compares them cell by cell and turns differences into terminal
output. Damage rectangles
([#11](https://github.com/rocky-star/orchidee/issues/11)) narrow
composition and may narrow the compare, but they are never the source of
truth: a lost damage mark costs bytes, not correctness.

Full repaint needs no separate path. The front grid can be filled with
an **unknown sentinel** — a text word of all ones, whose flags byte
claims the impossible combination of a continuation, a text width of 3,
and a length of 7, plus a style word with both spare bits set — and the
next diff emits every cell. It is used at enter, after resume, after a
resize, and after a present that failed part-way.

### Clear style

Cells nothing has painted still have to mean something, so the frame
carries a **clear style**: a full style word, set by the host through
the seam. Allocation, clearing, and resize fill every cell with a space
in that style. Until the host sets one, the provisional clear style is
`Palette(15)` on `Palette(0)`, documented as an application theme value
the host is expected to replace — the descendant of a Turbo Vision
background attribute.

### Resize

On a size change both grids are reallocated. The front becomes the
unknown sentinel, the back is filled with the clear style, the cursor is
clamped into the new grid, and the frame generation advances. The old
grid is not copied: surfaces re-layout on resize anyway (#11), so reuse
would add per-cell validity tracking for no gain. Allocation failure is
reported to the host; dimensions are `uint16`.

### Damage and generation

Each commit advances a monotonic frame generation, which present may use
to skip a frame that changed nothing and which diagnostics carry.
Composition produces damage as rectangles; the frame model requires only
that damage is an optimisation, never a precondition for a correct
present.

## The cursor

The frame carries the cursor as `{row, col, visible}`:

- it is frame state, never a flag on a cell, so moving it dirties no
  cells and disturbs no equality;
- the position is clamped into the grid at commit and normalised to the
  lead cell when it would land on a continuation;
- cursor shape is never set
  ([#10](https://github.com/rocky-star/orchidee/issues/10)); only
  position and visibility are;
- present places the cursor after the text it emits.

A software cursor that draws itself into cells, because a terminal
cannot show one, is out of scope; frame-level state would feed it if it
ever arrives.

## The text pool

Only clusters longer than 7 UTF-8 bytes reach the pool: emoji ZWJ
sequences, flags and skin-tone pairs, and dotted-circle repairs. The
pool is a content-deduplicated table:

- a cell referencing an entry increments its reference count;
  overwriting or clearing that cell decrements it; zero frees the entry;
- live entries are bounded by live cells, and the rare path is the only
  one that pays;
- the pool is capped at 1 MiB. On overflow the Compositor substitutes
  `U+FFFD`, counts the loss in diagnostics, and continues; it never
  blocks output the way an unmanaged pool can.

Refcounts, not session lifetimes or sweeps: a browsing session that
shows a thousand distinct clusters must not leak them, and a scan over
live grids at present would put the cost on the wrong path.

## Limits and failure

Rows and columns are `uint16`. A grid allocation failure, or a pool
allocation failure outside the capped path, is reported to the host and
leaves the previous frame intact; nothing is partially presented. The
error model itself belongs to the C ABI page
([#17](https://github.com/rocky-star/orchidee/issues/17)).

## Verification hooks

The testing stack
([#5](https://github.com/rocky-star/orchidee/issues/5)) should be able
to assert:

- word equality is content equality at every inline/pool boundary and
  after every fold;
- a width-0 cluster folds, circles, or drops per the table above;
- a wide glyph clipped at a clip edge leaves a space, not a half;
- overlapping a half glyph normalises it and counts the repair;
- the unknown sentinel never equals a cell the grid can produce;
- pool refcounts return to zero when a grid is cleared or resized;
- an orphan continuation is repaired before present.

## Deferred

- A terminal-default colour tag, reserved as tag `2` here.
- Underline colour and other extended attributes.
- Complex-script shaping across cell boundaries, bidi text, and input
  methods (#15 and the internationalisation fog on the map).
- Diff emission, run batching, synchronized output, partial writes, and
  the performance budget:
  [#13](https://github.com/rocky-star/orchidee/issues/13).
- Per-surface cell buffers and composition order:
  [#11](https://github.com/rocky-star/orchidee/issues/11).
- Reading text back out of the grid — selection and copy — not in v1.
