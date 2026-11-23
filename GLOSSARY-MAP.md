# Glossary Map

Orchidée is split into two glossary contexts. Each context's glossary
is created lazily, when its terms harden; the paths below are the
eventual homes.

## Contexts

- **Compositor** — `src/compositor/GLOSSARY.md`: the native layer.
  Owns the terminal backend, input decoding, the surface tree, the
  rendering pipeline, and the C ABI.
- **Python layer** — `src/python/GLOSSARY.md`: the first per-language
  layer. Owns widgets, event dispatch, window management, and async
  integration.

## Relationships

- **Compositor → Python layer**: the Compositor exposes the seam, the
  language-agnostic contract made of the C ABI and its conventions. It
  knows nothing about Python.
- **Python layer → Compositor**: the Python layer draws and receives
  input through the seam, never through compositor internals. It keeps
  the host loop; the Compositor stays passive.

Shared terms are defined once, in the context that owns them:
`surface` and `seam` belong to the Compositor; widget vocabulary
belongs to the Python layer.
