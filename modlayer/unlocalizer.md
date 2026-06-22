---
domain: modlayer/unlocalizer
tags: [modlayer, unlocalizer, lua, config]
sources: [src/xrServerEntities/script_storage.cpp, src/xrServerEntities/script_engine.cpp, gamedata/configs/unlocalizers/unlocalizer_test.ltx]
related: [modlayer/_overview.md, flows/lua-callback-lifecycle.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Unlocalizer

A load-time line-rewrite that promotes selected `local` variables/functions in a `.script` to
script-global scope, so other code can reach them — without editing the base script. Verified by
reading the engine implementation in `CScriptStorage::do_file`
(`src/xrServerEntities/script_storage.cpp`).

## Implementation (verified)

- **Config gather (once)** — `do_file` (`script_storage.cpp`), guarded by global
  `unlocalizerPassed`: lists `$game_config$/unlocalizers/` (root, `*.ltx`), parses each with
  `CInifile`, and for each section stores `unlocalizers[sectionName] = {variable names}`. **Section
  name is lowercased** (`ToLowerCase`); keys are the variable names. The global
  `unlocalizers` map (`script_storage.cpp`) is cleared on script-engine reinit
  (`script_engine.cpp`).
- **Per-script transform** — also in `do_file`: runs only if the script's namespace
  (the `.script` name), **lowercased**, is a key in `unlocalizers`. It splits the file into
  lines and rewrites each matching line, then loads the modified buffer (`load_buffer`); if no
  line changed it loads the original file unchanged.

## Supported forms (from the two regexes, verified)

1. **`local function name(...)`** — the local-function regex; if `name` is listed → rewritten to
   `function name(...)` (drops `local`, keeps the rest via capture `$3$4$5$6$7`).
2. **`local <vars> [= <values>]`** — the local-vars regex. Strips a trailing `-- comment`, splits on the
   first `=` into vars/value, splits vars on `,`. If **any** listed variable appears among the vars,
   the whole line's `local` is dropped (replace with capture `$3`); if there was no `=`,
   ` = nil` is appended. So:
   - `local a = v` → `a = v`
   - `local a` → `a = nil`
   - `local a,b,c` → `a,b,c = nil` (triggered if any of a,b,c is listed → all unlocalized)
   - `local a,b,c =...` → `a,b,c =...`

## Unsupported (verified from the logic)

- Multiple `local` keywords on one line (`local x; local y`, `local x local y`) — only the leading
  `^local` matches, so trailing locals are missed (confirmed by the `^local` anchor and the code
  comment in `do_file`).

## How to use

Drop a `.ltx` in `gamedata/configs/unlocalizers/` (any filename). Each `[section]` is the **target
script name without `.script`** (case-insensitive); list the variable names to unlocalize under it.
Canonical example: `unlocalizer_test.ltx` (its comments match the implementation above).

## Why it exists

Surgically expose one symbol a base script kept `local`, instead of forking the whole script (which
would break other mods' overrides/DLTX). Runs as part of normal script loading.

## Deeper

- Engine impl: `src/xrServerEntities/script_storage.cpp` (`CScriptStorage::do_file`, `unlocalRegex`),
  globals + reset: `src/xrServerEntities/script_engine.cpp`
- Reference config: `gamedata/configs/unlocalizers/unlocalizer_test.ltx`
- Script load context: `flows/lua-callback-lifecycle.md`, `engine/server-entities.md`

## Maintenance

- If the regexes in `CScriptStorage::do_file` change, re-derive the supported-forms list from
  them (they are the source of truth, not the `.ltx` comments).
- Re-grep `unlocalizers` / `do_file` if symbols move or are renamed.
