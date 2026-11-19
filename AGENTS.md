## Text formatting

Keep textual content to a soft limit of 72 columns per line. This
applies to Markdown documentation, commit messages, and any other prose
in this repo.

Break lines at logical boundaries, choosing the largest unit that fits:

- **Sentences**: after a period, question mark, or exclamation mark.
- **Clauses**: after a word like "and" or "which", after "if ... then",
  or after a comma.
- **Phrases**: between large phrases when no sentence or clause
  boundary fits.

Tables, code blocks, and URLs are exempt when reformatting them would
be awkward or would change their meaning.

## License headers

Every source code file starts with this block, written in the comment
syntax of the file's language (`//` for C-style languages, `#` for
Python and shell, `--` for SQL and Lua, `<!-- -->` for HTML, and so
on). Keep the wording unchanged.

```
This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at https://mozilla.org/MPL/2.0/.
```

## Agent skills

### Issue tracker

Issues live in GitHub Issues (`rocky-star/orchidee`), managed with the
`gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default triage labels, each string equal to its role name.
See `docs/agents/triage-labels.md`.

### Domain docs

Multi-context: a root `GLOSSARY-MAP.md` pointing at one `GLOSSARY.md`
per context, plus system-wide `docs/adr/` and per-context
`src/<context>/docs/adr/`. See `docs/agents/domain.md`.
