---
domain: flows/se-stalker
tags: [modlayer, gameplay, unpacked, lua, alife, server, save-load, flow]
sources: [unpacked/scripts/se_stalker.script, unpacked/scripts/class_registrator.script, unpacked/scripts/db.script, unpacked/scripts/sim_board.script]
related: [flows/alife-online-offline.md, flows/save-load.md, flows/db-registry.md, flows/smart-terrain-core.md, flows/gameplay-boot.md, flows/lua-callback-lifecycle.md, flows/special-actors.md, engine/server-entities.md, modlayer/gameplay/server-entities-lua.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: server entity — stalker (Lua) (F83)

The **Lua server-entity class** for an NPC stalker: `se_stalker` subclasses the engine's
`cse_alife_human_stalker` (C++ ALife server object, `engine/server-entities.md`) and overrides the
ALife lifecycle hooks (register, save, online/offline switch, death) to bolt the **gameplay layer**
onto each server object. This is the seam where the engine's ALife simulation (F16) meets the Lua
gameplay corpus (smart_terrain jobs F72, `db` registry F69, the callback bus F1). The same file also
defines the much thinner `se_trader` (subclass of `cse_alife_trader`).

**Provenance:** **verified** — `se_stalker.script` read in full; the registration seam, `SIMBOARD`,
and `db.offline_objects` cross-references grepped this session. **No `gamedata/` override** of
`se_stalker.script` (verified by listing — `gamedata/scripts/` has `se_projector`/`se_stmgun` but does
**not** shadow `se_stalker`). The engine base-class bodies (`cse_alife_human_stalker.*`) are
C++ (`engine/server-entities.md`, F16/F15) and are *not* re-read here — Lua calls up to them via
`cse_alife_human_stalker.METHOD(self, …)`.

## Concepts (verified)

- **server entity (Lua side)** — every ALife object has a C++ server class (`CSE_ALifeHumanStalker`)
  *and*, when registered with a `script_clsid`, a Lua subclass that the object_factory instantiates.
  `se_stalker` is that Lua subclass for stalker NPCs. *(verified: `class "se_stalker" (cse_alife_human_stalker)`.)*
- **registration binding** — `class_registrator.script:cs_register(object_factory, "CAI_Stalker",
  "se_stalker.se_stalker", "AI_STL_S", "script_stalker")` pairs the client class `CAI_Stalker` + the
  Lua server class `se_stalker.se_stalker` under clsid `AI_STL_S` / script_clsid `script_stalker`.
  `se_trader` ↔ `CAI_Trader` / `AI_TRD_S` / `script_trader`. *(verified by grep.)*
- **offline blackboard** — `db.offline_objects[id]` (F69) holds the offline NPC's `level_vertex_id` +
  `active_section` so its position/logic survive the online→offline boundary without a live engine
  object. *(verified: `__init`/`STATE_*`/`on_register`/`on_unregister` all touch it; `db.offline_objects = {}`.)*

## Sequence (spine — all verified in `se_stalker.script`)

### Construct — `se_stalker:__init(section)`
Calls `super(section)` (the engine ctor) then seeds `db.offline_objects[self.id]` (via `empty_table`).
Every stalker gets an offline blackboard slot at construction.

### Online/offline gate — `can_switch_offline` / `can_switch_online` (the F16 hook)
The engine ALife switch (F16 `flows/alife-online-offline.md`) polls these. `can_switch_offline`:
returns `true` immediately if `self.group_id ~= 65535` (NPC is in a squad ⇒ always allowed to go
offline), else defers to `cse_alife_human_stalker.can_switch_offline`. `can_switch_online` is a plain
pass-through to the engine (the squad-forcing branch is commented out). `switch_online`/`switch_offline`
are thin wrappers that just call the engine counterpart — the Lua override exists only as a hook point
(commented debug writes).

### Save / load — `STATE_Write` / `STATE_Read` (the F15 seam)
`STATE_Write`: call `cse_alife_human_stalker.STATE_Write` first (engine packet), then append three
fields — `level_vertex_id` (from the **live** `level.object_by_id(id):level_vertex_id()` if online,
else from `db.offline_objects[id].level_vertex_id`), `active_section` (from the offline blackboard),
and a `w_bool(false)` placeholder (the old `death_droped`, now hard-false). `STATE_Read`: engine read
first, then — **gated on `self.script_version > 10`** — read `old_lvid` + `active_section` back into
`db.offline_objects[id]` (string `"nil"` decoded to Lua `nil`), then `r_bool()` to consume the
placeholder. So the Lua layer **persists offline position + active logic section** alongside the engine
state. *(Note the asymmetry: writer always emits the fields; reader only consumes the lvid/section pair
when `script_version > 10` — older saves skip it. The trailing `r_bool` is always consumed.)*

### Register — `se_stalker:on_register()`
Order: `cse_alife_human_stalker.on_register(self)` → `SendScriptCallback("server_entity_on_register",
self, "se_stalker")` (F1 bus) → ensure `db.offline_objects[id]` exists → `self:brain():
can_choose_alife_tasks(false)` (**disables the engine's autonomous ALife task-picking** so the Lua
scheme/job layer drives the NPC instead) → read `spawn_ini():r_string_ex("logic","smart_terrain")` and,
if that smart resolves via `SIMBOARD:get_smart_by_name(name)`, call `smart:register_npc(self)` (F72 —
hands the NPC to its smart_terrain job board). This is the **birth of the gameplay binding**: from here
the smart_terrain assigns a job and the scheme engine (F66) takes over once the object goes online.

### Unregister — `se_stalker:on_unregister()`
Symmetric teardown: if `smart_terrain_id() ~= 65535`, resolve the smart via `alife_object(id)` and call
`smart:unregister_npc(self)`; clear `db.offline_objects[id] = nil`; fire
`server_entity_on_unregister`; finally `cse_alife_human_stalker.on_unregister(self)`.

### Spawn / death / update
- `on_spawn`: engine `on_spawn` + `SendScriptCallback("se_stalker_on_spawn", self)` (the Alundaio hook).
- `on_death(killer)`: engine `on_death` → `smart_terrain.on_death(self)` (release the NPC's job, F72) →
  if `group_id < 65535` resolve the squad via `alife_object(group_id)` and call
  `squad:on_npc_death(self, killer)` (F85 scripted-squad seam) → remove a `red_location` map spot if
  present.
- `update`: sets a one-shot `first_update` flag then calls engine `update`.

### `se_trader` (thin sibling)
`class "se_trader" (cse_alife_trader)`: `on_register`/`on_unregister` only fire the
`server_entity_on_register`/`_unregister` callbacks around the engine call; `keep_saved_data_anyway()`
returns `true` (the trader's persistent state is never pruned). No offline-blackboard / smart / squad
logic — a trader is a dialog/economy fixture (pairs `flows/special-actors.md` F36).

## Function digests (spine)

- **`se_stalker:__init`** — *Does:* engine ctor + seed `db.offline_objects[id]`.
- **`se_stalker:can_switch_offline` / `can_switch_online`** — *Does:* F16 switch gate; squad members
  always offline-able, else defer to engine. *Reads:* `self.group_id`.
- **`se_stalker:STATE_Write` / `STATE_Read`** — *Does:* engine packet ± offline `level_vertex_id` +
  `active_section` + placeholder bool; read gated on `script_version > 10`. *Reads/Writes:*
  `db.offline_objects[id]`, live `level.object_by_id(id)`.
- **`se_stalker:on_register`** — *Does:* engine register → fire callback → `can_choose_alife_tasks(false)`
  → smart_terrain `register_npc`. *Calls:* `SIMBOARD:get_smart_by_name`, `smart:register_npc`. *Fires:*
  `server_entity_on_register`.
- **`se_stalker:on_unregister`** — *Does:* smart `unregister_npc` + clear blackboard + fire callback +
  engine unregister. *Fires:* `server_entity_on_unregister`.
- **`se_stalker:on_death`** — *Does:* engine death + `smart_terrain.on_death` + `squad:on_npc_death` +
  map-spot cleanup. *Calls:* `alife_object(group_id)`.
- **`se_trader:*`** — *Does:* callback-wrapped engine register/unregister; `keep_saved_data_anyway=true`.

## Isolation ledger (the seams)

- **Engine base class:** `cse_alife_human_stalker` (C++ `CSE_ALifeHumanStalker`, `engine/server-entities.md`);
  every override calls up to it. The Lua class only *adds* gameplay bookkeeping.
- **ALife switch seam (F16):** `can_switch_offline/online` + `switch_offline/online` are the hooks the
  engine's `update_switch`/`CSwitchPredicate` polls (`flows/alife-online-offline.md`).
- **Save/load seam (F15):** `STATE_Write/Read` extend the engine's spawn/update packet
  (`flows/save-load.md`); the Lua additions are the offline position + active logic section.
- **Registry seam (F69):** owns `db.offline_objects[id]` (created in `__init`/`on_register`, cleared in
  `on_unregister`) (`flows/db-registry.md`).
- **Smart_terrain seam (F72):** `on_register`→`register_npc`, `on_unregister`→`unregister_npc`,
  `on_death`→`smart_terrain.on_death` (`flows/smart-terrain-core.md`); `SIMBOARD` = the global
  `simulation_board` (`sim_board.script`, F86).
- **Squad seam (F85):** `on_death`→`squad:on_npc_death` when `group_id < 65535`.
- **Callback bus seam (F1):** `server_entity_on_register`/`_unregister`/`se_stalker_on_spawn` via
  `SendScriptCallback` (`flows/lua-callback-lifecycle.md`).
- **Class binding seam (F65):** `class_registrator.cs_register` ties this Lua class to the engine
  object_factory by clsid (`flows/gameplay-boot.md`).

## Threading / failure / edge cases

- **`can_choose_alife_tasks(false)` is load-bearing:** without it the engine's ALife brain would pick
  its own offline tasks and fight the Lua smart_terrain job assignment — this line is *why* scripted
  logic wins for registered stalkers.
- **`script_version > 10` read guard:** loading an old save (version ≤ 10) skips the lvid/section read,
  so a migrated NPC may have no persisted `active_section` until it re-registers with a smart. The
  writer is unconditional, so new saves always carry the fields.
- **Offline blackboard lifetime:** `db.offline_objects[id]` is created at construct/register and nilled
  at unregister; a stale entry means a missed `on_unregister` (object removed without proper teardown).
- **Squad resolution can fail:** `on_death` only notifies the squad if `alife_object(group_id)` resolves
  — a stalker whose squad already unregistered simply skips `on_npc_death`.
- **`se_trader` keeps data forever:** `keep_saved_data_anyway()` returning `true` means trader state is
  never garbage-collected from the save — intentional for persistent economy NPCs.

## Deeper

- Lua class: `unpacked/scripts/se_stalker.script` (`se_stalker`, `se_trader`)
- Registration: `unpacked/scripts/class_registrator.script` (`cs_register` for `AI_STL_S`/`AI_TRD_S`)
- Engine base: `engine/server-entities.md` (`CSE_ALifeHumanStalker`, ALife server objects)
- Switch / save: `flows/alife-online-offline.md` (F16), `flows/save-load.md` (F15)
- Registry / smart / squad: `flows/db-registry.md` (F69), `flows/smart-terrain-core.md` (F72), F85 (scripted squad)
- Other se_ classes: `flows/se-others.md` (F84, when written), `modlayer/gameplay/server-entities-lua.md` (M6)

## Maintenance

- Verified core (registration binding, offline blackboard, the F16/F15/F72 override seams) is stable;
  re-read if the ALife switch or save format changes.
- Promote the engine base-class bodies (`cse_alife_human_stalker.*`) to verified only if a task needs the
  C++ side — they live in `engine/server-entities.md`.
- Re-check `gamedata/scripts/` for an `se_stalker` override before trusting (none at writing — gamedata
  only adds `se_projector`/`se_stmgun`).
- The `script_version` constants in the file's header comment (3/5/6 = job_online/was_in_smart_terrain/
  death_droped) are historical; the live read gate is `script_version > 10`.
