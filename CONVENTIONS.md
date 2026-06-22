---
domain: meta
tags: [meta, maintenance, format]
sources: [claudedocs/kb/]
related: [INDEX.md, GLOSSARY.md]
updated: 2026-06-05
---

# KB Conventions

Spec for this knowledge base: file format, tag vocabulary, and how to keep it in sync with the repo.
Read this before adding or editing KB files.

## Purpose

The KB lets a session get architecture + domain context immediately, then drill down via pointers
instead of re-grepping the repo each time. It is **local-only** (`claudedocs/kb/` is gitignored).

## File format

Every KB file (except this one's example) is: frontmatter → body → `## Deeper` → `## Maintenance`.

Frontmatter schema (YAML):

- `domain` — logical path in the tree, e.g. `engine/renderer`, `modlayer/scripting`. This is the
  tree position; it need not match the file's directory.
- `tags` — controlled vocabulary (see below). Used for relevance matching during recall.
- `sources` — real repo paths/dirs this file summarizes. **Must exist in the repo.** Verified by the
  integrity check.
- `related` — links to other KB files or glossary anchors. These edges *are* the tree.
- `updated` — ISO date of last sync with the repo.
- `branch` *(optional)* — set only when the file documents work that exists on a specific git branch
  rather than the mainline working branch (e.g. `branch: spom`). Such files also carry a banner and
  must have their in-tree code pointers verified with `git grep`/`git show` against that branch, not
  the current working tree. See the branch caveat in `INDEX.md`.

Body rules:

- Condensed prose. Explain how the domain works: concepts, control flow, where logic lives.
- **No engine/mod code snippets.** Code rots; use path and `file:symbol` pointers instead. Avoid
  fenced code blocks entirely — frontmatter is shown as a schema list below, not pasted code.
- **NEVER reference line numbers.** Line numbers go stale almost immediately and are no help locating
  anything (a symbol grep finds it instantly). The **deepest allowed source reference is file +
  symbol name** — a function, field, class, or macro name, e.g. `Xr_ini.cpp` (`CInifile::LTXLoad`) or
  `script_engine.cpp:load_common_scripts`. Forms like `file.cpp:456`, `(:923)`, `~:720`, or line
  ranges `:50–79` are forbidden. If a symbol is hard to find, name a nearby unique string to grep, not
  a line. The integrity check rejects any `:<digit>` reference.

`## Deeper` — bullet list of path / `file:symbol` pointers for going deeper, grouped by sub-topic.

`## Maintenance` — concrete checks that confirm the file still matches the repo (which dirs to list,
which symbols/markers to grep) and what to update on drift.

## Tag vocabulary

Keep tags lowercase, hyphenated, reused. Current set (extend deliberately, add to GLOSSARY if the
tag names a concept):

`engine`, `modlayer`, `meta`, `cpp`, `lua`, `config`, `renderer`, `dx9`, `dx10`, `dx11`, `r1`, `r2`,
`r3`, `r4`, `shaders`, `ai`, `ui`, `imgui`, `weapons`, `actor`, `inventory`, `physics`, `collision`,
`sound`, `net`, `alife`, `server`, `scheduler`, `build`, `dltx`, `dxml`, `ltx`, `unlocalizer`,
`callbacks`, `options`, `luabind`, `hot-reload`, `spom`, `assets`, `localization`, `mt`,
`flow`, `lifecycle`, `methodology`, `boot`, `frame`, `save-load`, `alife-transition`.

## Self-maintenance protocol

The KB is only useful if it tracks reality. Rules:

1. **Read-before-trust drift check.** When you open a KB file to use it and notice the repo has
   diverged (renamed dir, moved symbol, changed behavior), fix the file in the same session: update
   the body, fix `sources`/`related`, bump `updated`, and adjust the one-line entry in `INDEX.md`.
2. **Touch-what-you-learn.** When a task forces you to re-derive how a domain works (because the KB
   was thin or wrong), write that understanding back into the relevant file before finishing.
3. **New domain → new file + INDEX entry + glossary terms.** Don't grow one file past ~1 screen of
   prose; split and cross-link instead.
4. **Pointers over prose.** If you find yourself pasting code, replace it with a `file:symbol`
   pointer.
5. **Stale beats silent.** If unsure whether a file is current, note the uncertainty in its body
   rather than deleting; the next session can verify.

## Verifying the whole KB

- Every `sources:` path resolves: list each and confirm it exists.
- Every `related:` target resolves to a KB file or a real `GLOSSARY.md` anchor.
- Every tag that names a concept has a GLOSSARY entry.
- **No line numbers anywhere**: grep the KB for `:\d` — the only legitimate matches are inside
  `sources:`/paths and code examples (e.g. LTX `>key = a,b`), never a `file:NNN` citation.
- `git status` does not show `claudedocs/kb/` (it is ignored).
