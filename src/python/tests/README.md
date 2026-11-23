# Python layer tests

Tests for the Python layer live here: driver behaviour in sync and
asyncio modes, binding and lifetime rules, widget behaviour, and
palette application.

Use injected transport, clock, and capability seams so tests need no
real terminal; keep PTY smoke tests few. Compositor tests live in
`src/compositor/tests/`; there is no top-level `tests/` directory.
