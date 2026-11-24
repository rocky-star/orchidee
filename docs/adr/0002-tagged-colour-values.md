# Colour values crossing the seam are tagged palette-or-RGB

A colour crossing the Compositor seam is either `Palette(index)` for
an index in 0–255 or `RGB(r, g, b)`, never RGBA alone. Terminal
palettes vary, so the tag is what lets a per-language layer say
"theme colour 4" and keep the user's palette instead of freezing it to
xterm's RGB guess. The Compositor owns the mapping from either form to
what the terminal can display; the per-language layer owns which form
a theme uses.

## Considered options

- **RGB only**: a smaller C ABI and one conversion path everywhere,
  but selecting a 4-bit or 8-bit rendering depth would then be a pure
  colour approximation. "Light red" would become xterm's light red on
  every terminal, and a theme written in ANSI terms could not survive
  a palette change.
- **Palette only**: exact palette intent, but truecolor terminals
  could not express arbitrary colours — far too restrictive for a
  modern UI.
- **Tagged (`Palette | RGB`)**, chosen: a few extra bytes per colour
  and one branch in the mapper, in exchange for palette fidelity,
  meaningful depth selection, and a theme model that maps cleanly onto
  both ANSI and truecolor terminals.

## Consequences

The C ABI and the drawing contract carry a tagged colour type; every
per-language layer must preserve the tag rather than
flattening to RGB. The downgrade tables live in the Compositor, not
in each language layer. Adding a runtime palette setter later is
additive, because palette-addressed colours need no conversion.

Normative detail is in
[capabilities.md](../design/capabilities.md#colour-values-at-the-seam).
