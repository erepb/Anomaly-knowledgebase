---
domain: meta
tags: [meta, maintenance, format]
sources: [claudedocs/kb/]
related: [INDEX.md, GLOSSARY.md]
updated: 2026-08-17
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
  the current working tree. See the branch caveat in `INDEX.md`. Per rule 0 below, such files are NOT
  proactively updated while their feature is still in development — they are reconciled when it merges.
- `synced` / `synced_branch` / `synced_commit` — **`INDEX.md` only.** The KB-wide sync marker: the date,
  branch and short HEAD sha the whole KB was last reconciled against. They are mirrored in the banner at
  the top of `INDEX.md` and are the input to the next sync (see the protocol below). Per-file `updated`
  dates say when *that file* was last touched; these three say how current the KB *as a whole* is.

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

Keep tags lowercase, hyphenated, reused. **Reality check (measured 2026-08-17): ~230 distinct tags are
in use** — the list below is the *core* set (domain/layer/format), and files additionally carry
descriptive subject tags (`animation`, `map`, `schemes`, `squad`, `pda`, `xrgame`, `gameplay`,
`unpacked`, …). So this is not an enforced closed vocabulary; the rule that matters is **reuse an
existing tag rather than invent a synonym** (the corpus already contains near-duplicates such as
`outfit`/`outfits`, `scheme`/`schemes`, `xr-logic`/`xr_logic` — don't add more). To enumerate what
exists, grep the `tags:` lines across the KB before picking a new one.

Core set (extend deliberately, add to GLOSSARY if the tag names a concept):

`engine`, `modlayer`, `meta`, `cpp`, `lua`, `config`, `renderer`, `dx9`, `dx10`, `dx11`, `r1`, `r2`,
`r3`, `r4`, `shaders`, `ai`, `ui`, `imgui`, `weapons`, `actor`, `inventory`, `physics`, `collision`,
`sound`, `net`, `alife`, `server`, `scheduler`, `build`, `dltx`, `dxml`, `ltx`, `unlocalizer`,
`callbacks`, `options`, `luabind`, `hot-reload`, `spom`, `assets`, `localization`, `mt`,
`flow`, `lifecycle`, `methodology`, `boot`, `frame`, `save-load`, `alife-transition`.

## Self-maintenance protocol

The KB documents **merged (master) code only**. It is a map of what's in the tree today, not a
changelog of in-progress work. Documentation for a new feature is pulled in from `master`
**separately, after it merges** — never authored or kept in sync while the feature is being built.

Rules:

0. **No proactive KB updates during feature development.** While building or iterating on a feature, do
   NOT create, expand, or "keep in sync" a KB file for it — not a new file, and not an existing
   feature/branch-specific one, even for code you just wrote and even if a KB file for that feature
   already exists. A fast-moving feature churns the KB for no benefit; its docs are written once, later,
   when it lands on `master`. The ONLY KB edits allowed mid-development are **corrections to
   already-merged material you notice is wrong or stale while reading it** (rule 1). When in doubt,
   leave the KB alone.
1. **Read-before-trust drift check.** When you open a KB file to use it and notice the repo has
   diverged (renamed dir, moved symbol, changed behavior on `master`), fix the file in the same
   session: update the body, fix `sources`/`related`, bump `updated`, and adjust the one-line entry
   in `INDEX.md`.
2. **Touch-what-you-learn (about merged code).** When a task forces you to re-derive how an existing
   domain works (because the KB was thin or wrong), write that understanding back into the relevant
   file before finishing. This covers understanding of code already on `master`, not features you are
   adding.
3. **New domain → new file + INDEX entry + glossary terms.** Don't grow one file past ~1 screen of
   prose; split and cross-link instead.
4. **Pointers over prose.** If you find yourself pasting code, replace it with a `file:symbol`
   pointer.
5. **Stale beats silent.** If unsure whether a file is current, note the uncertainty in its body
   rather than deleting; the next session can verify.

## Repo sync protocol (the recurring "update the KB to current" task)

Rules 0–5 are incremental and opportunistic. This is the **periodic full sync**: bring the whole KB up
to the tree's current state. It is deliberately mechanical, and it always starts from the marker rather
than from a guessed date.

1. **Read the marker.** The banner at the top of `INDEX.md` (mirrored in its `synced` /
   `synced_branch` / `synced_commit` frontmatter) gives the last sync's date, branch and short sha.
   Confirm today's branch: `git branch --show-current`.
2. **Enumerate the range.** `git log --no-merges <synced_commit>..HEAD` — use the *commit*, not the
   date: commits arrive out of author-date order via merges, so a `--since=<date>` walk silently misses
   work. Cross-check against `README.md`'s "Changelog", which names the PR behind each batch.
3. **Get the file-level shape.** `git diff --stat <synced_commit>..HEAD -- src/ gamedata/` and cluster
   the changes by KB domain, not by commit.
4. **Read the code, then write.** For each cluster read the current code (headers and the changed
   functions — not just the diff, which hides later reverts and follow-up rewrites), then update the KB
   file that *owns that behaviour*. Removing claims about code that no longer exists matters as much as
   adding new ones: a feature branch's docs routinely describe an approach that was replaced before
   merge. Bump each touched file's `updated`.
5. **Sweep for contradictions.** Grep the whole KB for the symbols/attributes you just removed or
   renamed — including the `INDEX.md` "Audit status" rows, which restate per-file facts and go stale
   the same way the bodies do.
6. **Run the integrity checks** in "Verifying the whole KB" below.
7. **Record the sync.** Update the `INDEX.md` banner + the three `synced*` fields to today's date,
   branch and `git rev-parse --short HEAD`; append an entry to the `INDEX.md` **Sync log** naming the
   range, what was reconciled, and anything deliberately left open. That entry is what makes the next
   sync cheap.

## Verifying the whole KB

- Every `sources:` path resolves: list each and confirm it exists. **Checked 2026-08-17: all resolve.**
- Every `related:` target resolves to a KB file or a real `GLOSSARY.md` anchor. **Known failures
  (2026-08-17, pre-existing):** 16 files link to `modlayer/gameplay/npc-schemes.md`,
  `modlayer/gameplay/scheme-engine.md`, `modlayer/gameplay/smart-terrain-jobs.md` and
  `modlayer/gameplay/state-manager.md`, none of which exist — that material ended up in the `flows/`
  scheme files instead. Either create the four maps or repoint the links; not done yet because the
  files themselves are otherwise current.
- Every tag that names a concept has a GLOSSARY entry.
- **No line numbers anywhere**: grep the KB for `:\d` — the only legitimate matches are inside
  `sources:`/paths and code examples (e.g. LTX `>key = a,b`), never a `file:NNN` citation.
- `git status` does not show `claudedocs/kb/` (it is ignored).
