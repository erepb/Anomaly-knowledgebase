---
domain: modlayer/gameplay/utils-db
tags: [modlayer, gameplay, unpacked, lua, utils, db, registry, helper, map]
sources: [unpacked/scripts/utils_obj.script, unpacked/scripts/utils_item.script, unpacked/scripts/utils_data.script, unpacked/scripts/utils_xml.script, unpacked/scripts/utils_ui.script, unpacked/scripts/utils_stpk.script, unpacked/scripts/db.script, unpacked/scripts/_g.script, unpacked/scripts/class_registrator.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/utils-libraries.md, flows/db-registry.md, flows/gameplay-boot.md, flows/scheme-engine-core.md, flows/lua-callback-lifecycle.md, GLOSSARY.md]
updated: 2026-06-21
verified: map
---

# Utility & registry foundation — L1 map (M22)

The **bottom of the gameplay stack**: the cross-cutting helper libraries and the global registries
everything else stands on. Nothing here is a "system" — these are the leaves the rest of the corpus
calls into. This map routes three things: the `utils_*` **helper libraries**, the `db` **global state
registry**, and the Lua **bootstrap** (`_g`, `class_registrator`) that wires the script VM together.

## The pieces

| Cluster | File(s) | Role | Flow |
|---------|---------|------|------|
| helper libraries | `utils_obj`, `utils_item`, `utils_data`, `utils_xml`, `utils_ui`, `utils_stpk` | the cross-cutting `utils_*` helper layer (~500 functions) | F115 ✅ (`flows/utils-libraries.md`) |
| global registry | `db` | the central mutable Lua state registry (`db.storage`, `db.actor`, `db.used_level_vertex_ids`, offline tables) | F69 ✅ (`flows/db-registry.md`) |
| script bootstrap | `_g`, `class_registrator` | the global VM prelude (helpers, `DEV_DEBUG`, error reporter) + the OOP class registration that names every server/client class | F69 (`_g`, `flows/db-registry.md`) + ⬜ (`class_registrator`) |

## How it fits

These are **dependency-ordered last** in the roadmap precisely because everything depends on them and
they depend on almost nothing. The `utils_*` libraries (F115) are pure helper collections — object/squad
navigation, item/price logic, INI parsing + the `cfg_file` writer, UI XML/cell rendering, and
server-packet read/write. `db` (F69) is the shared mutable state every script reads/writes — the
`utils_*` helpers and `db` are tightly paired (e.g. `utils_obj.lmove` reserves cover vertices in
`db.used_level_vertex_ids`). `_g` is the VM prelude loaded first (global functions, the `DEV_DEBUG`
gate, the error reporter — F69/F65/F111), and `class_registrator` registers the Lua names for the
engine's C++ class hierarchy so `clsid`→Lua-class resolution works during spawning.

## Subsystem deep-dives (delegates to flows)

- The `utils_*` helper libraries (catalog: what helper lives where + the load-bearing ones) →
  `flows/utils-libraries.md` (F115) ✅
- The `db` global state registry (`db.storage` per-object state, online/offline tables, actor cache) →
  `flows/db-registry.md` (F69) ✅ (also covers `_g`)
- VM bootstrap / callback bus the helpers register through → `flows/gameplay-boot.md` (F65),
  `flows/lua-callback-lifecycle.md`

## Maintenance

- F115 ✅ (utils libraries) and F69 ✅ (db + `_g`). **`class_registrator` and any remaining global
  registries are not yet a dedicated flow** — they are small and mostly mechanical; fold them into
  `flows/db-registry.md` or a short note here if a task needs them.
- **No `gamedata/scripts/` override** of any `utils_*` file (verified by `find` at F115). If a modded
  exe ever forks a `utils_*`, re-check here — these are high-fan-in, so an override has wide blast radius.
- The `utils_stpk` parse/fill packet functions must track the engine's C++ `STATE_Read`/`STATE_Write`
  layout (`flows/save-load.md`, `modlayer/gameplay/server-entities-lua.md`) — flag drift if a cse class
  changes serialization.
