# Compositor tests

Tests for the Compositor live here, beside the layer they exercise.
Python-layer tests live in `src/python/tests/`; there is no top-level
`tests/` directory.

The test stack — in-process fd injection rather than PTY-first,
cell-grid goldens across TERM plus pinned byte goldens, libvterm as an
oracle, and fuzzed decoder invariants — is recorded in the research
note for
[Test and verification stack for the Compositor](https://github.com/rocky-star/orchidee/issues/5).
The normative page `docs/design/testing.md` is planned.

Fixtures and goldens get subdirectories here as they appear.
