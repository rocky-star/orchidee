# Compositor

The native layer of Orchidée: terminal backend, input decoding, the
surface tree, rendering, and the C ABI. This glossary fixes the
vocabulary of terminal capability handling before it spreads into
implementation.

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
