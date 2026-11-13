---
domain: flows/dltx-config-load
tags: [engine, config, dltx, ltx, flow, lifecycle]
sources: [src/xrCore/Xr_ini.cpp, src/xrCore/xr_ini.h]
related: [engine/xrCore.md, modlayer/config-ltx.md, METHODOLOGY.md, GLOSSARY.md]
updated: 2026-11-13
---

# Flow: LTX parse + DLTX merge (config load)

How a `.ltx` config is parsed and how mods patch it non-destructively. When the engine builds a
`CInifile` from a root file (e.g. `system.ltx` → `pSettings`), DLTX discovers sibling
`mod_<name>_*.ltx` files and merges their sections/keys per operator semantics. All in
`src/xrCore/Xr_ini.cpp`. Line numbers are current-as-of-`updated` hints; trust `file:function` names.

## Trigger / entry

Engine (or Lua via `ini_file`/`ini_file_ex`) constructs a `CInifile` from a path —
`CInifile::CInifile(szFileName, …)` (`Xr_ini.cpp`). The global `pSettings` is `system.ltx`
loaded this way.

## Sequence

1. **Cache check** — the ctor checks `CachedData` under `CacheCS` (a critical section). On
   hit (when `dltx_use_cache` and the filename is cacheable), it reuses parsed data (`[DLTX] Found
   data in cache`) and skips parsing.
2. **Root parse** — otherwise `loadFile` → **`LTXLoad(F, …, bIsRootFile=TRUE, …)`** reads
   the root file line by line: strips comments (`;`, `//`, quote-aware), then classifies each line:
   - `#include "x.ltx"` (and `*.ltx` globs) → recurse `loadFile` at **depth+1**.
   - Section header: `![name]` = override fragment (`isOverrideSection`), `@[name]` = override
     fragment + create-if-missing (`isSafeOverrideSection`), plain `[name]` = base section. Override
     fragments go to `OverrideData` and merge onto base later (see Operator semantics); plain goes to
     `BaseData`. Switches the current section (`StashCurrentSection` finalizes the previous one).
   - `key = value` line → `insert_item` (see below).
3. **Mod-file discovery** — when the **root** file hits EOF, it stashes the last section,
   then in the root's directory:
   - lists `<FileName>_*.ltx` (`AmbiguousFiles`) and `mod_<FileName>_*.ltx` (`ModFiles`),
   - for each mod file, runs `bIsModfileMeantForMe` — a regex check that rejects a mod file if it
     actually targets a *different* base whose name is a prefix (resolves `mod_X_Y.ltx` ambiguity
     between base `X` and base `X_Y`),
   - `loadFile`s each accepted mod file at **negative depth** (`d = -200`, decreasing by `dt=-200`),
     so mods sort *below* the base and later mod files override earlier ones.
4. **Section finalize** — `StashCurrentSection` commits each parsed section. A duplicate base
   section that isn't marked override is a **fatal** (names the conflicting files).
5. **Filter + sort** — `SortAndFilterSection` drops items equal to `DLTX_DELETE` and orders
   the rest by `insertionIndex`.
6. **Merge + evaluate** — `EvaluateSection` builds each final section: start from
   `BaseData[name]`, **merge the `OverrideData[name]` fragment onto it key-by-key** via
   `MergeSections` (override value wins per key, base-only keys kept, `DLTX_DELETE` removes),
   then merge resolved parents (`[child]:parent`, via recursive `EvaluateSection` + `MergeParentSet`;
   `!parent` removes an inherited parent). The stashed `>`/`<` list modifications are applied here.
   Cyclical inheritance is a **fatal**.
7. **Cache store** — the merged result is stored in `CachedData` (under `CacheCS`) for reuse.

## Operator semantics (what the line prefixes mean)

**Section operators are a key-level MERGE, not a whole-section replace.** Plain `[section]` lines go
to `BaseData`; `![section]`/`@[section]` lines go to `OverrideData`. At evaluation the override
fragment is merged **onto** the base section key-by-key (`EvaluateSection`→`MergeSections`):

- `![section]` / `@[section]` — mark an **override fragment**. Its keys **replace** the base's value
  for those keys; base keys you don't mention are **kept**; keys only in the fragment are **added**.
  Multiple fragments for the same section (across files) **accumulate**. Neither wipes the section.
  - Difference between the two: `@[section]` also **creates** the section if no base defines it
    (`sectionsMarkedForCreate`); `![section]` is for overriding a section that already exists. (Merge
    behavior is otherwise identical.)
- plain `[section]` duplicated without `!`/`@` → **fatal** (that's what the markers suppress).
- `>key = a, b` — **append** `a, b` to the list value of `key`.
- `<key = a` — **remove** `a` from the list value of `key`.
- `key = DLTX_DELETE` — delete that key from the merged section.
- (`>`/`<` are detected in `insert_item` via the first char → `ModifyListType`; applied during
  evaluation, which reads the real key as `(*key)+1`, skipping the operator char.)
- `[child]:parent` inheritance; `:!parent` removes an inherited parent (`MergeParentSet`).

Within a single section, when the same key appears more than once, **depth then insertion order**
decide the winner (`SortAndFilterSection` sort) — see "Subtleties".

## Function digests (L3)

- **`CInifile::CInifile(szFileName, …)`** — *Does:* build a config — cache lookup, else
  `loadFile`/`LTXLoad`, then cache. *Reads/Writes:* `CachedData` (under `CacheCS`). *Gotcha:* cache
  governed by `dltx_use_cache` + `IsValidFileNameForCache`.
- **`CInifile::LTXLoad(F, path, bIsRootFile, fileName, depth, …)`** — *Does:* line-parse one
  file; on root EOF, discover and load `mod_*` files at negative depth; handle `#include`, section
  markers, and items. *Calls:* `loadFile` (mods/includes), `StashCurrentSection`, `insert_item`.
  *Gotcha:* mod discovery happens once, only for the root file.
- **`CInifile::loadFile(...)`** — *Does:* open a reader for a path and invoke `LTXLoad` (used
  for root, includes, and mod files; `depth` sets precedence).
- **`CInifile::insert_item(Sect* tgt, Item& I)`** — *Does:* if key's first char is `>`/`<`
  (`ModifyListType::Insert/Remove`), stash into `OverrideModifyListData[section]` with `insertionIndex`
  for later; else `tgt->Data.push_back`. *Writes:* `OverrideModifyListData` or `Sect::Data`.
- **`CInifile::StashCurrentSection(...)`** — *Does:* commit the in-progress section; record
  section→file. *Gotcha:* duplicate non-override base section → `Debug.fatal` naming the files.
- **`CInifile::SortAndFilterSection(Sect&)`** — *Does:* remove `DLTX_DELETE` items, sort by
  `insertionIndex`.
- **`CInifile::MergeSections(base, override, DeletedItems, IsMergingBaseAndMod)`** — *Does:*
  **key-by-key merge** of two sorted item lists — base-only key kept, override-only key added,
  key in both → override value wins, `DLTX_DELETE` removes the key. *Gotcha:* this
  is the proof that `!`/`@` merge rather than replace; reused for base+mod and parent+child merges.
- **`CInifile::EvaluateSection(name, …)`** — *Does:* build a section = `BaseData[name]` with
  `OverrideData[name]` merged onto it, then parents merged in (recursive); applies
  `>`/`<` list mods (key read at `(*key)+1`) + `DLTX_DELETE`. *Gotcha:* cyclical deps fatal; warns on
  override/missing parents.
- **section markers** — `isOverrideSection` (`!`) and `isSafeOverrideSection` (`@`) both route a
  section to `OverrideData` (merge); `@` additionally adds it to `sectionsMarkedForCreate` so
  an empty base is created if missing. Plain `[section]` → `BaseData` (duplicate = fatal in
  `StashCurrentSection`).
- **`MergeParentSet(base, override, …)`** — *Does:* merge parent lists; `!parent` removes an
  inherited parent.

## State touched

- Per-`CInifile`: `Sect::Data` (final items), `OverrideModifyListData` (pending `>`/`<` mods),
  section→filename map (for diagnostics).
- Static/shared: `CInifile::CachedData` + `CacheCS` (parse cache); flags `dltx_use_cache`,
  `print_dltx_warnings`.

## Threading

The parse cache (`CachedData`) is guarded by `xrCriticalSection CacheCS` — relevant because the MT
build loads resources/configs from multiple threads. Use `InvalidateCache(path)` when files change.
`InvalidateCache()` with no/empty path clears the whole cache; Lua `reload_system_ini()`
(`script_ini_file_script.cpp`) now does that (2026.10.11) instead of invalidating only `system.ltx`.

## Extension points

This *is* the mod-config mechanism. Drop `mod_<basefile>_<modname>.ltx` next to the base file with
`!`/`@` sections and `>`/`<`/`DLTX_DELETE` keys (see `modlayer/config-ltx.md`). New engine-read params
are catalogued in `gamedata/scripts/ltx_help_ex.script`.

## Failure & edge cases

- **Duplicate section without `!`/`@`** → fatal (names base file, the section's file, and the
  duplicate's file). The single most common DLTX authoring error.
- **Cyclical inheritance** → fatal.
- **Ambiguous mod filename** (`mod_X_Y.ltx`) → routed to the correct base by regex; mis-naming
  silently sends a mod to the wrong/no base.
- **Malformed `key=` line** → warning, line skipped, section may be altered unexpectedly.
- Enable `print_dltx_warnings` for verbose diagnostics.

## Subtleties

- **Depth = precedence**: base 0, includes depth+1, mod files negative and decreasing — lower depth
  wins, so mods override base and later mods override earlier ones.
- `>`/`<` keys are stored with the operator char prefixed; evaluation reads the true key via pointer
  `+1`. Searching item data for a bare key name won't match the stashed modification entries.

## Deeper

- All logic: `src/xrCore/Xr_ini.cpp` (functions above), header `src/xrCore/xr_ini.h`
- Operator recap + repo's `mod_*.ltx` + param catalog: `modlayer/config-ltx.md`
- Foundation context: `engine/xrCore.md`

## Maintenance

- Re-grep anchors if lines drift: `CInifile::LTXLoad`, `insert_item`, `StashCurrentSection`,
  `MergeSections`, `EvaluateSection` in `Xr_ini.cpp`; markers `DLTX_DELETE`, `dltx_use_cache`,
  `file_list(... "mod_"`.
- If the operator set or ambiguity resolution changes, sync this file, `modlayer/config-ltx.md`, and
  `GLOSSARY.md#dltx`.
