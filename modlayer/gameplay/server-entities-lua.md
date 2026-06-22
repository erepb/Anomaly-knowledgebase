---
domain: modlayer/gameplay/server-entities-lua
tags: [modlayer, gameplay, unpacked, lua, alife, server, save-load, map]
sources: [unpacked/scripts/se_stalker.script, unpacked/scripts/se_monster.script, unpacked/scripts/se_artefact.script, unpacked/scripts/se_item.script, unpacked/scripts/se_zones.script, unpacked/scripts/se_car.script, unpacked/scripts/se_heli.script, unpacked/scripts/se_level_changer.script, unpacked/scripts/se_smart_cover.script, unpacked/scripts/se_actor.script, unpacked/scripts/class_registrator.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/se-stalker.md, flows/alife-online-offline.md, flows/save-load.md, flows/db-registry.md, flows/smart-terrain-core.md, flows/gameplay-boot.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Lua server entities (`se_*`) — L1 map

The **Lua tier of the ALife server objects**. Every ALife entity has a C++ server class
(`CSE_ALife*`, `engine/server-entities.md`); when `class_registrator` gives it a `script_clsid`, the
object_factory instantiates a **Lua subclass** (`se_*`) that overrides the ALife lifecycle hooks to
attach the gameplay corpus (smart_terrain jobs F72, the `db` registry F69, the callback bus F1) to each
object. The `se_*` files are uniformly **thin subclasses**: they call up to the engine base (`cse_*.METHOD(self, …)`)
and add register/save/online-switch/death bookkeeping. This map routes the family; mechanics live in
the flows.

**Read first:** `flows/se-stalker.md` (F83) — the canonical pattern (every `se_*` is a delta on it).
Always grep `gamedata/scripts/` for an override before trusting an `unpacked/` `se_*` file.

## The family (verified by listing `unpacked/scripts/se_*`)

| File | Engine base(s) | Role | Flow |
|------|----------------|------|------|
| `se_stalker` | `cse_alife_human_stalker`, `cse_alife_trader` | NPC stalker + trader — the template | F83 |
| `se_monster` | `cse_alife_monster_base` | monsters (one class for ALL monster clsids) — near-clone of F83 | F84 |
| `se_artefact` | `cse_alife_item_artefact` | artefacts (+150 m proximity online gate) | F84 |
| `se_item` | `cse_alife_item*` + outfit/helmet/weapon/physic/lamp/ammo/… | the **whole item/object server catalog** (~19 classes) | F84 |
| `se_zones` | `cse_anomalous_zone`/`cse_torrid_zone`/`cse_zone_visual`/`cse_alife_space_restrictor` | `se_zone_*` anomalies + `se_restrictor` (teleport-route map spots) | F84 |
| `se_car` | `cse_alife_car` | drivable cars | F84 (F96) |
| `se_heli` | `cse_alife_helicopter` | helicopters (`se_helicopters` table, target/respawn persistence) | F84 (F95) |
| `se_level_changer` | `cse_alife_level_changer` | level-transition triggers (enabled/hint) | F84 |
| `se_smart_cover` | `cse_smart_cover` | smart-cover server objects (loophole persistence) | F84 (F33) |
| `se_actor` | `cse_alife_creature_actor` | the player's server object — **a simulation target** | F84 |

`gamedata/scripts/` adds `se_projector` + `se_stmgun` (new classes, **not** overrides of the above —
verified by listing). Engine bases above are **verified** from the `class "X" (base)` declarations (F84).

## The uniform pattern (verified in F83/F84 — almost every `se_*` is a minimal delta)

1. **Registration** — `class_registrator.cs_register(object_factory, "<ClientClass>", "<file.se_class>",
   "<CLSID>", "<script_clsid>")` binds client+server class under a clsid (F65). *(verified for
   `AI_STL_S`/`AI_TRD_S`.)*
2. **`__init`** — `super(section)` + seed any per-object registry slot (e.g. `db.offline_objects[id]`).
3. **`on_register`/`on_unregister`** — engine call + `SendScriptCallback("server_entity_on_register/…")`
   + attach/detach to smart_terrain / registries.
4. **`can_switch_online/offline` + `switch_*`** — the F16 ALife online↔offline hooks.
5. **`STATE_Write`/`STATE_Read`** — extend the engine save packet with Lua-side state (F15).
6. **`on_death`/`on_spawn`/`update`** — lifecycle hooks firing callbacks + notifying smart/squad.

## Subsystem deep-dives (delegates to flows)

- Stalker + trader server class → `flows/se-stalker.md` (F83) ✅
- All other `se_*` classes (monster/artefact/item-catalog/zones/car/heli/level_changer/smart_cover/actor)
  → `flows/se-others.md` (F84) ✅ — the family shape + the classes that add real behavior
  (`se_actor` simulation target, `se_restrictor` teleport routes, `se_smart_cover` loopholes, `se_heli`).
- ALife switch mechanism (engine) → `flows/alife-online-offline.md` (F16)
- Save/load mechanism → `flows/save-load.md` (F15); C++ server objects → `engine/server-entities.md`
- Registry the `se_*` write → `flows/db-registry.md` (F69); smart_terrain seam → `flows/smart-terrain-core.md` (F72)
- Squad-simulation seam (`se_actor` is a target; squads path to it) → F85 (scripted squad) / F86 (sim_board)

## Maintenance

- This map is the `se_*` routing hub; `ROADMAP-gameplay.md` (Phase G3, F83–F87) is the authoritative
  plan/status. F83+F84 cover the whole `se_*` family; engine bases are verified from the class decls.
- Re-list `unpacked/scripts/se_*` + `gamedata/scripts/se_*` if the corpus changes; honor the override rule.
