# Compositor

The native layer (the core layer) of Orchidée: terminal capability
handling, input decoding, the surface tree, rendering, and the C ABI
that per-language layers code against. C++ internals behind a C ABI;
passive and loop-agnostic.

Design pages are indexed in
[docs/design/README.md](../../docs/design/README.md).

## Layout

- `include/orchidee/compositor.h` — public C ABI header (placeholder).
- `src/` — implementation sources, created when implementation starts.
- `tests/` — Compositor tests.

## Build

Configured from the repository root:

```
cmake -S . -B build
```

`src/compositor/CMakeLists.txt` currently defines an INTERFACE
placeholder target; it becomes a compiled library when implementation
starts.
