# Compositor

The native layer of Orchidée: terminal backend, input decoding, the
surface tree, rendering, and the C ABI. This glossary fixes the
Compositor's vocabulary before it spreads into implementation.

## Capabilities

**Capability snapshot**:
The frozen record of what one terminal can do, resolved once when the
Compositor starts and consulted instead of the terminal description
afterwards.
_Avoid_: terminal info, caps, capabilities object

**Colour depth**:
The most colours a terminal can display, as one of monochrome, 8, 16,
256, or truecolor.
_Avoid_: colour support, bit depth

**Rendering depth**:
The colour depth the Compositor is currently asked to render at, never
above the terminal's colour depth.
_Avoid_: output depth, colour mode

**Fallback tier**:
A compiled-in substitute for a terminal description, used when no
usable description can be found.
_Avoid_: default terminal, built-in terminfo

**Capability provenance**:
The source a capability value came from, such as a terminal
description, the environment, an override, or a fallback tier.
_Avoid_: origin, source

## Terminal backend

**Terminal backend**:
The Compositor component that holds the terminal for a session: it owns
the saved terminal settings, the mode set, and every byte written to
the screen.
_Avoid_: terminal driver, screen, tty layer

**Session**:
One period of terminal ownership, from enter to leave, including any
suspensions in between.
_Avoid_: run, instance, connection

**Mode set**:
The terminal modes a session turns on — raw input, alternate screen,
mouse, paste, focus, keyboard protocol — each gated by the capability
snapshot, enabled as one set on enter and disabled as one set on leave.
_Avoid_: terminfo flags, feature set, terminal options

**Restore script**:
The precomposed, unconditional byte sequence that returns the terminal
to the state the backend found it in.
_Avoid_: cleanup, teardown, reset

**Suspend**:
Leaving the terminal so a stopped job or a spawned program can use it,
followed by an enter when the backend resumes.
_Avoid_: pause, detach, background

## Rendering

**Cell**:
One column of the screen's character grid: the text shown there and the
style it is drawn in. A character two columns wide occupies two adjacent
cells, the second of which carries no text.
_Avoid_: character, position, slot

**Continuation cell**:
The second of the two cells a two-column character occupies; it holds no
text of its own and exists only to give the glyph its second column.
_Avoid_: trailer, shadow cell, blank cell

**Style**:
The foreground and background colour of a cell together with its
abstract attributes, such as bold or reverse.
_Avoid_: attribute, format, appearance

**Frame**:
The complete grid of cells composed for the screen at one point,
together with the cursor state; what presentation compares against what
the terminal has already been sent.
_Avoid_: screen, buffer, snapshot

**Clear style**:
The style given to cells nothing has drawn; the screen's background.
_Avoid_: default colour, background colour

**Display width**:
The number of columns a text cluster occupies under the Compositor's
rules, which is not always its number of code points or its raw Unicode
width; a lone combining mark is drawn on a base and counts as one.
_Avoid_: string length, character count
