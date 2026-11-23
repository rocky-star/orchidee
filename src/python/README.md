# Python layer

The first per-language layer: widgets, event dispatch, window
management, and async integration. It codes against the Compositor's
seam and never against compositor internals.

The package name, module layout, binding technology, and packaging
shape are decided by
[Python binding technology and packaging](https://github.com/rocky-star/orchidee/issues/21).

Design pages are indexed in
[docs/design/README.md](../../docs/design/README.md).

## Layout

- `tests/` — Python-layer tests (driver, binding, widget behaviour).
