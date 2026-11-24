# Dependency policy: permissive-only, system-first, pinned fallback

Orchidée is MPL-2.0 and must stay embeddable, so every dependency
below the Compositor is permissive and passive: no copyleft, no
threads, no loops, no global terminal state. libtinfo is a required
system library because its database belongs to the machine; utf8proc
is system-first with a hash-pinned download fallback, because a
distro copy may be older than the Unicode version the width policy
needs, yet vendoring 2.4 MB of tables — or a git submodule, whose
contents GitHub's source tarballs omit — buys nothing for packagers
who would link the system copy anyway.

## Consequences

- An offline build without a system utf8proc must pre-seed
  `FETCHCONTENT_SOURCE_DIR_UTF8PROC`.
- Exact width goldens only hold against the pinned version; CI forces
  the download path and other configurations run invariants.
- The licence allowlist and the dependency constitution are enforced
  at review, by hand, until tooling exists.
