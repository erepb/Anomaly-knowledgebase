---
domain: flows/sim-board
tags: [modlayer, gameplay, unpacked, lua, alife, squad, simulation, flow]
sources: [unpacked/scripts/sim_board.script, unpacked/scripts/simulation_objects.script, unpacked/scripts/smart_terrain.script]
related: [flows/scripted-squad.md, flows/smart-terrain-core.md, flows/gulag-jobs.md, flows/se-others.md, flows/alife-services.md, flows/relations-info.md, flows/options-runtime.md, flows/spawns-alife-graphs-configs.md, engine/server-entities.md, modlayer/gameplay/squads-simulation.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: sim_board — the simulation board (F86)

The `simulation_board` singleton (`SIMBOARD` / `get_sim_board()`): the **registry of smart_terrains and the
matchmaker between squads and smarts**. It is the other half of the squad-sim loop — where the squad (F85)
decides it *wants* a target and asks for one, the board **supplies the target** (`get_squad_target`),
**records the squad↔smart assignment + population** (`assign_squad_to_smart`), and **populates the world at
game start** (`fill_start_position`). One global instance, lazily created, nilled on new game.

**Provenance:** **verified** — `sim_board.script` read in full this session. Callers/helpers grepped:
`register_smart`/`init_smart`/`unregister_smart` are called from `smart_terrain.script` (F72);
`smart_terrain_squad_count` lives in `smart_terrain.script`; `in_time_interval` + `change_team_squad_group`
in `_g.script`; `simulation_objects.object_registry`/`available_by_id`/`evaluate_prior` are the F85 scorer.
**No `gamedata/` override** of `sim_board.script` (verified by listing).

## Concepts (verified)

- **SIMBOARD** — the singleton (`_G.SIMBOARD`, accessor `get_sim_board()`; `clear()` nils it on new game).
  Owns `smarts` (id → `{smrt, squads={}, population}`), `smarts_by_names`, `squads`, `tmp_assigned_squad`.
- **smart registry** — every smart_terrain registers here on its `on_register` (F72); the board is the
  name→smart and id→record lookup the rest of the sim uses (`get_smart_by_name`, `SIMBOARD.smarts[id]`).
- **assignment + population** — a squad is "in" a smart via `smarts[id].squads[squad.id]`; `population`
  counts assigned squads (excluding those with a script target), gating how full a smart is (F85's
  `evaluate_prior` rejects smarts at `population >= max_population`).

## Sequence (spine — verified)

### Smart registry — `register_smart` / `unregister_smart` / `init_smart`
`register_smart(obj)` (called from `smart_terrain:on_register`, F72) adds `smarts[id] = {smrt=obj,
squads={}, population=0}` + `smarts_by_names[name]=obj` (dup-guarded). `init_smart(obj)` flushes any
`tmp_assigned_squad[id]` — squads that asked to assign to this smart *before* it had registered get
assigned now. `unregister_smart` drops both entries.

### The matchmaker — `assign_squad_to_smart(squad, smart_id)` (the central op)
This is what the F85 runtime override routes through. Steps: (1) **deferral** — if `smart_id` is given but
that smart isn't registered yet, queue the squad in `tmp_assigned_squad[smart_id]` and return (drained by
`init_smart`). (2) **leave old** — clear `smarts[old].squads[squad.id]`, recompute
`smarts[old].population = smart_terrain.smart_terrain_squad_count(...)`, fire `squad_on_leave_smart`. (3)
if `smart_id == nil` → `squad:assign_smart(nil, old)` and return (detach). (4) **join new** —
`squad:assign_smart(smarts[smart_id].smrt, old)` (registers the squad's members on the smart, F85), then
add `smarts[smart_id].squads[squad.id]=true` and **increment population** *only if the squad has no script
target* (script-targeted squads don't count toward smart fullness), fire `squad_on_enter_smart`.

### Target supply — `get_squad_target(squad)` (the other half of F85 generic_update)
Iterate `simulation_objects.object_registry`; for each candidate that is `not locked`, `~= squad.id`, and
`available_by_id[id]`, score it with `se_target:evaluate_prior(squad)` (F85; type-gated props × distance).
Keep a **top-5** buffer by prior (replacing the lowest when a better one appears), then pick the result as
**either a random one of the 5 (50% of the time) or the single highest-prior (50%)** — the Tronex
randomization that stops every squad converging on the one best target. Returns the chosen `se_target` (a
smart/squad/actor server object) or nil. Reuses a module `priority_tasks` table to avoid GC churn.

### World population — `fill_start_position()`
Called once from `se_actor:on_register` at game start (F84). Reads `misc\simulation.ltx` (or
`misc\simulation_survival_mode.ltx` in Survival): for each section that names a registered smart, spawn
each listed `squad_section` × `count`, where `count` is scaled by the `alife_stalker_pop`/`alife_mutant_pop`
options (F17) for sections flagged `common` (0.5 → random 0/1, else rounded). A `new_game_test` config flag
short-circuits to an empty test map. Fires `fill_start_position`.

### Squad creation — `create_squad` / `create_squad_at_named_location`
`create_squad(spawn_smart, sq_id)`: `alife_create` the squad section at the smart's position →
`squad:create_npc(spawn_smart)` (F85 spawns members) → `assign_squad_to_smart(squad, spawn_smart.id)` →
`setup_squad_and_group` each member + fire `squad_on_npc_creation`. The named-location variant reads a
position from `named_locations.ltx` and spawns there instead of at a smart.

### Member grouping — `setup_squad_and_group(se_obj)`
Resolves the member's squad + that squad's smart, then `change_team_squad_group(se_obj, team, smart.squad_id
or se_obj.squad, group)` — assigns the engine **team:squad:group** triplet so the engine's online combat /
agent-manager (F35) treats squadmates as one group.

### Target preconditions — module `general_*_precondition(squad, target)`
Per-prop-type gates a target's `target_precondition` consults: `general_squad_precondition` → **false**
(squads are not auto-targetable, matching F85 `sim_available`); `general_base`/`territory_precondition` →
faction + time gates (own/empty always; enemy-occupied only in narrow hours via `in_time_interval` +
`game_relations.is_factions_enemies`); `general_resource_precondition` → `in_time_interval(10,6)`;
`general_lair_precondition` → true (monsters fill lairs first).

## Function digests (spine)

- **`simulation_board:register_smart` / `init_smart` / `unregister_smart`** — *Does:* smart registry +
  deferred-assignment drain. *Called by:* `smart_terrain` (F72).
- **`assign_squad_to_smart`** — *Does:* move a squad between smarts, maintaining `squads`/`population` +
  firing enter/leave callbacks; defers if the smart isn't registered yet. **The F85 override's assign path.**
- **`get_squad_target`** — *Does:* score all available sim objects via `evaluate_prior`, return a top-5
  random-or-best pick. *Reads:* `simulation_objects.object_registry`/`available_by_id`.
- **`fill_start_position`** — *Does:* spawn the initial world population from `simulation.ltx` scaled by
  pop options. *Called by:* `se_actor:on_register` (F84).
- **`create_squad` / `create_squad_at_named_location`** — *Does:* `alife_create` + member spawn + assign +
  group. *Fires:* `squad_on_npc_creation`.
- **`setup_squad_and_group`** — *Does:* set the engine team:squad:group for a member.
- **`get_sim_board` / `clear`** — *Does:* singleton accessor / new-game reset.

## Isolation ledger (the seams)

- **smart_terrain seam (F72):** smarts register here on spawn; `assign_squad_to_smart` calls back into
  `squad:assign_smart` → `smart:register_npc`/`unregister_npc`; population via `smart_terrain_squad_count`.
- **squad seam (F85):** `get_squad_target` is called from `sim_squad_scripted:generic_update`;
  `assign_squad_to_smart` is the target of the F85 *runtime override*; `evaluate_prior`/`target_precondition`
  are squad methods.
- **simulation_objects seam:** `object_registry`/`available_by_id` (the candidate pool) drive
  `get_squad_target`.
- **ALife seam (F41/F16):** `alife_create` spawns squads; squads persist offline; `setup_squad_and_group`
  bridges to engine online groups.
- **Options seam (F17):** `fill_start_position` reads `alife_stalker_pop`/`alife_mutant_pop` via `ui_options.get`.
- **Relations seam (F40):** the `general_*_precondition` time/faction gates use `game_relations`.
- **Callback bus (F1):** `squad_on_enter_smart`/`_leave_smart`/`_on_npc_creation`, `fill_start_position`.

## Threading / failure / edge cases

- **Deferred assignment (`tmp_assigned_squad`):** a squad can target a smart that hasn't registered yet
  (load order); it's queued and drained by `init_smart`. A squad stuck unassigned often means its smart
  never registered.
- **Population excludes script-targeted squads:** `assign_squad_to_smart` only increments population for
  squads without a `get_script_target` — story/scripted squads don't make a smart "full". This is why
  scripted squads can pile into a smart past `max_population`.
- **Top-5 + 50/50 random/best:** `get_squad_target` deliberately doesn't always pick the best target — half
  the time it picks a random one of the top 5. Two identical squads won't deterministically chase the same
  smart. The reused `priority_tasks` table is a GC optimization (don't assume it's empty between calls — it's
  `iempty_table`'d on use).
- **Singleton lifetime:** `SIMBOARD` is lazily created by `get_sim_board()` and **nilled by `clear()` on new
  game** — never cache the instance across a new-game boundary.
- **`general_squad_precondition` is false:** squads aren't auto-targetable through the board; squad-vs-squad
  targeting comes from the scripted/help-target path (F85), not `get_squad_target`.

## Deeper

- Board: `unpacked/scripts/sim_board.script` (`simulation_board`, `get_sim_board`, `assign_squad_to_smart`,
  `get_squad_target`, `fill_start_position`, `create_squad*`, `general_*_precondition`)
- Squad (the caller): `flows/scripted-squad.md` (F85); scorer: `simulation_objects.evaluate_prior`
- Smarts/jobs: `flows/smart-terrain-core.md` (F72), `flows/gulag-jobs.md` (F73)
- World-start actor hook: `flows/se-others.md` (F84, `se_actor:on_register` → `fill_start_position`)
- Map: `modlayer/gameplay/squads-simulation.md` (M5)

## Maintenance

- Verified core (smart registry, `assign_squad_to_smart` population bookkeeping, `get_squad_target` top-5
  pick, `fill_start_position` population) is stable. Member-spawn details inside `create_squad` are spine-read.
- `smart_terrain_squad_count` (population recompute) lives in F72; cross-check if population accounting changes.
- No `gamedata/` override at writing — re-check per session.
