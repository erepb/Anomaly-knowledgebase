---
domain: flows/se-others
tags: [modlayer, gameplay, unpacked, lua, alife, server, save-load, flow]
sources: [unpacked/scripts/se_monster.script, unpacked/scripts/se_artefact.script, unpacked/scripts/se_item.script, unpacked/scripts/se_zones.script, unpacked/scripts/se_actor.script, unpacked/scripts/se_smart_cover.script, unpacked/scripts/se_level_changer.script, unpacked/scripts/se_car.script, unpacked/scripts/se_heli.script, unpacked/scripts/class_registrator.script]
related: [flows/se-stalker.md, flows/alife-online-offline.md, flows/save-load.md, flows/db-registry.md, flows/smart-terrain-core.md, flows/smart-cover.md, flows/detector-artefact.md, flows/vehicles-holder.md, flows/special-actors.md, flows/lua-callback-lifecycle.md, engine/server-entities.md, modlayer/gameplay/server-entities-lua.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: server entities — the rest of the `se_*` family (F84)

The remaining Lua server-entity classes beyond `se_stalker` (F83): the **catalog** of `se_*` subclasses
covering monsters, items/weapons/outfits, zones, the actor, smart covers, level changers, cars, and
helicopters. The dominant fact is that **almost every one is a minimal delta on the F83 template** —
`on_register`/`on_unregister` fire `server_entity_on_register`/`_unregister` around the engine call,
`can_switch_online` passes through, `STATE_Write/Read` pass through — and only a handful add real
gameplay behavior. This flow documents the family shape + those exceptions; read `flows/se-stalker.md`
first for the template.

**Provenance:** **verified** — all nine `se_*` files read in full this session; the class→clsid
registrations grepped in `class_registrator.script`; `set_save_marker` (`_g.script`),
`simulation_objects.register` (`simulation_objects.script`) confirmed present. Engine base bodies
(`cse_alife_*`) are C++ (`engine/server-entities.md`) and not re-read. **No `gamedata/` override** of any
of these files (gamedata adds `se_projector`/`se_stmgun`, separate classes).

## Two surprises worth stating up front (verified)

- **`se_item.script` is the whole item/object server catalog**, not one class. It defines ~19 classes:
  `se_outfit`, `se_helmet`, `se_weapon`(+`_shotgun`/`_automatic_shotgun`/`_magazined`/`_magazined_w_gl`),
  `se_item`, `se_item_torch`, `se_physic`, `se_lamp`, `se_ammo`, `se_grenade`, `se_eatable`, `se_invbox`,
  `se_explosive`, `se_pda`, `se_detector`, `se_mgun`. All are minimal passthroughs (the only non-trivial
  ones are `se_physic`'s AWR `repack.direction` write hook and the `keep_saved_data_anyway()=true` on
  physic/lamp/eatable).
- **`se_monster.script` is a near-verbatim clone of `se_stalker`** — same `db.offline_objects` seeding,
  same `can_switch_offline` squad gate (`group_id ~= 65535 → true`), same `can_choose_alife_tasks(false)`
  + smart `register_npc` in `on_register`, same `on_death`→`smart_terrain.on_death`+`squad:on_npc_death`.
  Every monster clsid (bloodsucker/boar/dog/burer/controller/…) registers to this **one** `se_monster`
  class (verified in `class_registrator`). So "monster server entity" = the F83 pattern.

## The classes that add real behavior (verified)

### `se_actor` — the player is a simulation target
Subclass of `cse_alife_creature_actor`. Beyond the callback wrapper it: registers with
`simulation_objects.register(self)` on register + `SIMBOARD:fill_start_position()` once; persists a
`start_position_filled` bool (guarded by save markers). Critically it **implements the simulation-target
interface** the squad sim (F85/F86) calls on a target: `get_location`, `am_i_reached` (always false),
`on_reach_target` (clears squad members' `db.offline_objects` + `db.spawned_vertex_by_id`, then
`SIMBOARD:assign_squad_to_smart(squad, nil)`), `get_alife_task` (a `CALifeSmartTerrainTask` at the
actor's gvid/lvid, or `db.storage[0].loud_*` if set), `sim_available` (false if nearest smart < 50 m and
not available), `target_precondition` (true only in Survival mode), `evaluate_prior`. So the actor is a
**squad destination** in the offline simulation, not just a creature.

### `se_artefact` — proximity keeps it online
Subclass of `cse_alife_item_artefact`. Minimal except `can_switch_offline`: returns **false** if
`alife():actor()` is within 150 m of the artefact (keeps artefacts simulated near the player), else
defers to engine.

### `se_restrictor` (in `se_zones.script`) — teleport-route map spots
Subclass of `cse_alife_space_restrictor`. `on_register` reads `sr_teleport_sections.ltx`: if the
restrictor's name is a section there, it adds or removes the route's map spot
(`level.map_add_object_spot_ser`/`map_remove_object_spot`) depending on whether the route is opened
(`txr_routes.scan_route` / a saved `routes_<name>` var / `opened_routes` state). `keep_saved_data_anyway()
= true`. The other zone classes in this file (`se_zone_anom` ← `cse_anomalous_zone`, `se_zone_torrid` ←
`cse_torrid_zone`, `se_zone_visual` ← `cse_zone_visual`) are minimal — note their `STATE_Read`
early-returns under `editor()`. (Header comment: anomaly respawn check moved out of `update` to avoid
running per-frame over hundreds of zones.)

### `se_smart_cover` — loophole persistence + registry
Subclass of `cse_smart_cover` (ties F33 `flows/smart-cover.md`). Maintains two module-level registries
`registered_smartcovers` (by name) + `registered_smartcovers_by_lv_id` (by level→level_vertex), populated
in `on_before_register`/`on_register`, cleared in `on_unregister`. `STATE_Write/Read` persist
`last_description` + a per-loophole enabled map, **validated on read against
`smart_covers.descriptions[desc].loopholes`** (drops stale loophole ids; `script_version >= 9` gate).
`FillProps` exposes loopholes in the editor.

### `se_heli` — AI aircraft bookkeeping
Subclass of `cse_alife_helicopter`. Registers into a module `se_helicopters[id]` table (the comment notes
the heli class has **no update function**, so it is "spoofed in actor update"). Reads `player_id`/`faction`
from `ini_sys`. `STATE_Write/Read` persist `assigned_target_id`/`current_target_id`/`m_smart_terrain_id`/
`respawn_point_id`/`respawn_point_prop_section` (u16 with 65535 = nil sentinel). `on_unregister` decrements
the respawn point's spawned count + unregisters from its smart. `clear_smart_terrain` resets its logic via
`xr_logic.reset_logic`. `keep_saved_data_anyway()=true`. (Ties F95.)

### `se_level_changer` / `se_car` — small persistence
`se_level_changer` (← `cse_alife_level_changer`) persists `enabled` + `hint`. `se_car` (← `cse_alife_car`,
ties F96) is minimal with `keep_saved_data_anyway()=true`.

## Registration map (verified in `class_registrator.script`)

`cs_register(object_factory, "<ClientClass>", "<file.se_class>", "<CLSID>", "<script_clsid>")` per row.
Highlights: **all monster clsids → `se_monster.se_monster`**; item/weapon/outfit/detector clsids →
specific `se_item.*` classes; `ce_script_zone → se_zones.se_restrictor`; `smart_cover_object →
se_smart_cover`; `CActor → se_actor`; `CHelicopter → se_heli`; `CCar → se_car`; `CPhysicObject` and
`CDestroyablePhysicsObject` both → `se_item.se_physic`; `hanging_lamp → se_item.se_lamp`.

## Function digests (the non-trivial spine; rest = F83 template)

- **`se_monster:*`** — *Does:* the F83 NPC template verbatim (offline blackboard, squad offline gate,
  `can_choose_alife_tasks(false)`, smart register, death→squad). One class for all monster clsids.
- **`se_artefact:can_switch_offline`** — *Does:* false within 150 m of actor, else engine.
- **`se_actor`** — *Does:* `simulation_objects.register` + `fill_start_position` + the simulation-target
  interface (`get_location`/`on_reach_target`/`get_alife_task`/`sim_available`/`evaluate_prior`).
- **`se_restrictor:on_register`** — *Does:* teleport-route map-spot add/remove from `sr_teleport_sections.ltx`.
- **`se_smart_cover:STATE_Read`** — *Does:* restore loopholes validated against `smart_covers.descriptions`.
- **`se_heli`** — *Does:* `se_helicopters[id]` registry + target/respawn persistence + smart unregister.
- **`se_item.se_physic:STATE_Write`** — *Does:* AWR `repack.direction` cse_abstract patch then engine write.

## Isolation ledger (the seams)

- **Engine base classes:** each subclasses a distinct `cse_alife_*` (`engine/server-entities.md`); the
  Lua class adds only the gameplay hooks. The same C++ ALife machine (F16/F15) drives all of them.
- **Callback bus (F1):** uniform `server_entity_on_register`/`_unregister` (+ `se_actor_on_STATE_Read/Write`,
  `se_stalker_on_spawn` for stalkers) via `SendScriptCallback`.
- **Registry seams:** `se_monster` owns `db.offline_objects` (F69, like F83); `se_actor` →
  `simulation_objects` (F85/F86) + `SIMBOARD`; `se_smart_cover` → `registered_smartcovers*`; `se_heli` →
  `se_helicopters`.
- **Smart_terrain seam (F72):** `se_monster`/`se_heli` register/unregister with smarts; `se_actor` is a
  *target* (squads path to it) rather than a job-holder.
- **Save markers:** `se_actor`/`se_level_changer` bracket their extra fields with `set_save_marker`
  (`_g.script`) — the standard versioned-save guard (F15).
- **Config seams:** `se_restrictor` ← `sr_teleport_sections.ltx`; `se_heli` ← `ini_sys` player/faction.

## Threading / failure / edge cases

- **One `se_monster` for all monsters:** monster-specific behavior is NOT here — it lives in the online
  client object + the `mob_*` schemes (F94). The server class is identical across species by design.
- **`editor()` guards:** several `STATE_Read` (zones, level_changer, actor) early-return under the level
  editor — don't expect saved fields when running in-editor.
- **Loophole validation drops stale ids:** a smart cover whose `.description` changed between saves loses
  the loopholes that no longer exist (verified in `se_smart_cover:STATE_Read`) — intentional migration.
- **65535 = nil sentinel:** `se_heli` (and the F83 squad/smart checks) treat `65535`/`u16 max` as "none";
  a value of 65535 is not a real id.
- **`se_actor.on_reach_target` wipes offline state:** when a squad reaches the actor it clears each
  member's `db.offline_objects` + `db.spawned_vertex_by_id` and reassigns the squad — a stale offline
  entry after a squad reaches the player indicates this didn't run.

## Deeper

- NPC clones: `unpacked/scripts/se_monster.script` (template = `flows/se-stalker.md`)
- Item catalog: `unpacked/scripts/se_item.script` (~19 classes); detectors → `flows/detector-artefact.md`
- Zones/restrictors: `unpacked/scripts/se_zones.script` (`se_zone_*`, `se_restrictor` + teleport routes)
- Actor: `unpacked/scripts/se_actor.script` (simulation-target interface → F85/F86)
- Smart cover: `unpacked/scripts/se_smart_cover.script` (→ `flows/smart-cover.md` F33)
- Vehicles: `unpacked/scripts/se_car.script`, `se_heli.script` (→ `flows/vehicles-holder.md`, F95/F96)
- Registration: `unpacked/scripts/class_registrator.script`; map: `modlayer/gameplay/server-entities-lua.md` (M6)

## Maintenance

- The verified core (uniform F83 delta + the listed exceptions + the registration map) is stable; re-read
  a specific class only if a task needs its exact persistence fields.
- Promote engine base-class bodies to verified only when a task needs the C++ side
  (`engine/server-entities.md`).
- Re-check `gamedata/scripts/se_*` for new overrides (currently only `se_projector`/`se_stmgun`, which
  add classes rather than shadow these).
