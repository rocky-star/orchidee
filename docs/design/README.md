# Design Documentation

Normative design for Orchidée. Each page states the decisions for one
topic; the reasoning lives in the ticket that resolved it, and later
tickets amend the page as they refine it. A page is written when its
ticket resolves, so the index marks pages as *planned* until the file
exists.

Ticket names match their GitHub issue titles.

## Pages

### Cross-cutting

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `architecture.md` | Layers, boundaries, and vocabulary | [Foundation and dependency policy](https://github.com/rocky-star/orchidee/issues/7) | planned |

### Compositor (core layer)

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `capabilities.md` | Capability model and downgrade policy | [Capability model and downgrade policy](https://github.com/rocky-star/orchidee/issues/9) | planned |
| `terminal-backend.md` | Backend lifecycle and terminal modes | [Terminal backend lifecycle and modes](https://github.com/rocky-star/orchidee/issues/10) | planned |
| `surface-tree.md` | Surface tree and scene model | [Surface tree and scene model](https://github.com/rocky-star/orchidee/issues/11) | planned |
| `frame-model.md` | Cell, attribute, and frame model | [Cell, attribute, and frame model](https://github.com/rocky-star/orchidee/issues/12) | planned |
| `present-pipeline.md` | Present and flush pipeline | [Present and flush pipeline](https://github.com/rocky-star/orchidee/issues/13) | planned |
| `input-decoding.md` | Input decoding and event taxonomy | [Input decoding and core event taxonomy](https://github.com/rocky-star/orchidee/issues/14) | planned |
| `unicode-text.md` | Text, grapheme, and width model | [Unicode text and grapheme model](https://github.com/rocky-star/orchidee/issues/15) | planned |
| `ffi-objects.md` | Handles, ownership, and lifetimes | [FFI object model and lifetime](https://github.com/rocky-star/orchidee/issues/16) | planned |
| `c-abi.md` | C ABI conventions and compatibility | [C ABI conventions and compatibility policy](https://github.com/rocky-star/orchidee/issues/17) | planned |

### Seam

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `seam-events.md` | Event vocabulary crossing the seam | [Seam event vocabulary](https://github.com/rocky-star/orchidee/issues/18) | planned |
| `seam-drawing.md` | Drawing commands crossing the seam | [Seam drawing contract](https://github.com/rocky-star/orchidee/issues/19) | planned |
| `seam-async.md` | Loop integration and async contract | [Seam async and loop-integration contract](https://github.com/rocky-star/orchidee/issues/20) | planned |

### Python layer

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `python-binding.md` | Binding technology and packaging | [Python binding technology and packaging](https://github.com/rocky-star/orchidee/issues/21) | planned |
| `python-driver.md` | Sync and asyncio driver API | [Python driver API: sync and asyncio](https://github.com/rocky-star/orchidee/issues/22) | planned |
| `python-widgets.md` | Widget model, layout, and windows | [Python widget model, layout, and window management](https://github.com/rocky-star/orchidee/issues/23) | planned |
| `python-events.md` | Dispatch, key bindings, signals/slots | [Python event dispatch, key bindings, and signals/slots](https://github.com/rocky-star/orchidee/issues/24) | planned |
| `python-palette.md` | Palette and colour scheme model | [Python palette and colour scheme model](https://github.com/rocky-star/orchidee/issues/25) | planned |

### Applications

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `file-manager.md` | Two-pane file manager spec and widgets | [Dogfood app spec: two-pane file manager](https://github.com/rocky-star/orchidee/issues/8), [Widget set and behaviours for the file manager](https://github.com/rocky-star/orchidee/issues/26) | planned |

### Verification

| Page | Topic | Ticket | Status |
| --- | --- | --- | --- |
| `testing.md` | Test and verification stack | [Test and verification stack for the Compositor](https://github.com/rocky-star/orchidee/issues/5) | planned |

When a ticket writes a page, replace its code span with a link and set
its status to *written*.

## Repository layout

```
.
├── CMakeLists.txt              build entry point
├── GLOSSARY-MAP.md             context index for glossaries
├── docs/
│   ├── design/                 normative design pages (this index)
│   └── adr/                    system-wide ADRs (created lazily)
└── src/
    ├── compositor/             core layer
    │   ├── include/orchidee/   public C ABI headers
    │   └── tests/
    └── python/                 first per-language layer
        └── tests/
```

Tests live beside the layer they verify, one `tests/` per layer. There
is deliberately no top-level `tests/` directory: it would blur
ownership and become a dumping ground. Cross-layer integration tests,
when they appear, get an explicit home at that point.

## Build

```
cmake -S . -B build
```

The root `CMakeLists.txt` configures the scaffold; `src/compositor/`
currently exposes an INTERFACE placeholder target. The C++ and C
standard levels are provisional until implementation starts.

## Continuous integration

CI is deferred. There is nothing to compile or test beyond configuring
the skeleton, so a pipeline now would be untested configuration.
It lands with the first buildable code and test target.
