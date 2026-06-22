---
domain: flows/scripted-squad
tags: [modlayer, gameplay, unpacked, lua, alife, server, squad, simulation, flow]
sources: [unpacked/scripts/sim_squad_scripted.script, unpacked/scripts/simulation_objects.script, gamedata/scripts/sim_squad_scripted.script]
related: [flows/se-stalker.md, flows/se-others.md, flows/smart-terrain-core.md, flows/gulag-jobs.md, flows/alife-online-offline.md, flows/save-load.md, flows/db-registry.md, flows/alife-services.md, flows/ai-squad-logic.md, flows/relations-info.md, engine/server-entities.md, modlayer/gameplay/squads-simulation.md, modlayer/gameplay/server-entities-lua.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: scripted squad behavior (F85)

The **squad server class** `sim_squad_scripted` (subclass of the engine `cse_alife_online_offline_group`)
and its target-selection helper `simulation_objects`. A squad is an offline ALife group of NPCs that, each
update, **picks a target** (a smart_terrain, another squad, or the actor), moves toward it, and on arrival
hands its members to that smart's job board (F73). The squad has a **dual role**: it is both a *simulation
participant* (it selects targets) and a *simulation target* (other squads can path to it) — it implements
the same target interface as `se_actor` (F84). This is the macro layer above the per-NPC scheme engine: it
decides *where a group of NPCs goes*, while smart_terrain/gulag (F72/F73) decide *what each does there*.

**Provenance:** **verified** — the spine of `sim_squad_scripted` (`update`, `get_script_target`,
`specific_update`, `generic_update`, `assign_smart`, `assign_squad_member_to_smart`, member control,
`check_online_status`, `can_switch_*`, the target interface) read this session, plus `simulation_objects`
in full. `SIMBOARD:get_squad_target`/`assign_squad_to_smart`/`setup_squad_and_group`/
`general_squad_precondition` confirmed present in `sim_board.script` (F86). `is_squad_monster` (`_g.script`)
confirmed. **`sim_squad_bounty`/`sim_squad_warfare` are NOT subclasses** — they are separate manager modules
(verified: no `class` decl). Engine base (`cse_alife_online_offline_group`) is C++ (`engine/server-entities.md`).

## The runtime override (critical — `gamedata/` wins)

There **is** a `gamedata/scripts/sim_squad_scripted.script` override, same length, **10 changed lines in 2
spots**: in `specific_update` and `generic_update`, where the canonical `unpacked/` code calls
`self:assign_smart(se_target, self.smart_id)`, the runtime version calls
`sim_board.get_sim_board():assign_squad_to_smart(self, se_target.id)` instead. So at runtime squad→smart
assignment is routed **through the sim_board** (which does proper population/registry bookkeeping) rather
than the squad assigning itself directly. Everything else is identical. Cite the `gamedata/` copy for the
assignment path.

## Concepts (verified)

- **squad** — `sim_squad_scripted`, an `online_offline_group_s` server object holding member NPC ids
  (`squad_members()`, `commander_id()`). Persists across online/offline (offline = a point on the graph).
- **target** — what the squad is heading to: a smart_terrain id, another squad id, or `0` (= the actor).
  Stored as `assigned_target_id` (chosen) vs `current_target_id` (reached). `current_action`: `0` = moving,
  `1` = reached/staying, `nil` = none.
- **simulation_objects** — the **target registry + scorer**: `register`/`unregister` keep `object_registry`
  + `available_by_id`; `get_props` loads per-target weight props from `misc\simulation_objects_props.ltx`;
  `evaluate_prior(target, squad)` scores a candidate. *(verified.)*

## Sequence (spine — verified)

### Per-update tick — `sim_squad_scripted:update()`
`cse_alife_online_offline_group.update(self)` first. On the **first update**: optionally grant
`item_on_all`, set `rush_to_target`, fire `squad_on_first_update`. Then every tick: `self:refresh()` →
bail if `disabled` → reset location masks if flagged → **`check_online_status()`** (computes
`forced_online_status`) → fire `squad_on_update` → compute `dist_to_actor` → `check_invulnerability` →
**target dispatch**: `local id = self:get_script_target()`; if it returns an id → `specific_update(id)`
(scripted target), else → `generic_update()` (auto target). Finally fire
`squad_on_after_game_vertex_change`/`squad_on_after_level_change` on graph movement.

### Target choice — `get_script_target()`
Priority order: explicit `self.scripted_target` → actor-hostage (`axr_task_manager.hostages_by_id`) →
`random_targets` (pick one while idle) → `action_condlist` (a condlist resolving to a target name). Special
cases: companions that can't teleport and are off-level return `self.id` (stay put); `"actor"` → `0`. A
name is resolved via `SIMBOARD.smarts_by_names` (smart) or `alife_object(tonumber)` (squad/smart id). Returns
`nil` when there is no scripted target → caller falls to `generic_update`.

### Scripted target — `specific_update(script_target_id)`
Resolve `se_target = alife_object(id)`; set `assigned_target_id = id`. **If the target is a smart and not
yet our smart → `assign_smart`** (runtime: `sim_board:assign_squad_to_smart`). Then run the **action FSM**:
if `current_action == 0` (moving) and `se_target:am_i_reached(self)` → `se_target:on_after_reach(self)`,
mark `current_action = 1`, record `stay_time`. If `current_action == 1` and the idle timer
(`game_time():diffSec(stay_time) < idle_time`) hasn't elapsed → hold. When done staying →
`se_target:on_reach_target(self)`, clear target, re-`setup_squad_and_group` each member.

### Auto target — `generic_update()`
`__lock` short-circuits (avoid re-searching every tick and tanking FPS). `assigned_target_id =
get_help_target_id(self) or assigned or current`. Validate the current target via `target_precondition`;
run the same reached/idle FSM (with smart-assign). **If no current target → find one:**
`SIMBOARD:get_squad_target(self)` (F86; the candidate scorer that ranks all sim objects via
`simulation_objects.evaluate_prior`). No target → `on_reach_target(self)`, clear, set `__lock`. Got one →
`on_reach_target`, set `assigned_target_id = target.id`, `current_action = 0`, re-`setup_squad_and_group`.

### Target scoring — `simulation_objects.evaluate_prior(target, squad)` (verified)
Base prior `5`, gated by target type: **actor** → 0 unless same level + `target_precondition`; **squad** →
0 if self / different level / either in a combat-ignore safe zone / target already targeting another squad
(anti-clusterfuck) / fails precondition; **smart_terrain** → 0 if disabled / population ≥ max_population /
no squads table / (off-level and (monster or transition disallowed)). Then weight: add the target's
`props[squad.player_id]` koeff + a behaviour-table sum (`default_monster_behaviour` or
`default_squad_behaviour` × the target's per-tag props), and finally **× `evaluate_prior_by_dist`**
(`1 + 1/graph_distance`). Higher = more attractive. Props come from `simulation_objects_props.ltx` per
target; `sim_avail` is a condlist evaluated on the PH thread (`AddUniqueCall` in `on_game_start`).

### Member control
`create_npc`/`add_squad_member` spawn members from the squad section; `remove_npc(id, force)` unregisters a
member from its smart + releases it (offline) and, when the squad empties, runs `death_condlist`, detaches
from its smart, drops `SIMBOARD.squads[id]`, hides + releases the squad; `on_npc_death(se_npc, se_killer)`
clears the dead NPC's binder squad ref, fires `squad_on_npc_death`, then `remove_npc`. `assign_smart` /
`assign_squad_member_to_smart` register/unregister members on smart change (the override routes the squad-level
one through sim_board).

### Online/offline — `check_online_status` → `can_switch_*`
`check_online_status` computes `forced_online_status` (`true`/`false`/`nil`): story objects + survival mode +
companions force online; `__lock` (no target) forces offline; monster day/night `player_id`s gate by hour;
`enforce_online_exclusion` / proximity exclusion can force offline near the actor. `can_switch_offline` =
engine-allows **and** not `forced_online_status==true`; `can_switch_online` = engine-allows **and** not
`forced_online_status==false` (engine checked first to avoid the "object with ID already exists" crash).

### The squad as a target (interface — same shape as `se_actor`, F84)
`get_location`, `am_i_reached` (true if empty or self-target), `on_after_reach`, `on_reach_target` (clears
members' `db.offline_objects`/`db.spawned_vertex_by_id`, detaches smart), `get_alife_task`
(`CALifeSmartTerrainTask` at the squad's gvid/lvid), `get_current_task` (engine-called; prefers the assigned
target's per-commander job task), `sim_available` (returns **false** — squads are not auto-targetable by
default), `target_precondition` (self or `sim_board.general_squad_precondition`), `evaluate_prior` (delegates
to `simulation_objects.evaluate_prior`).

### Persistence
`STATE_Write/Read` (+ `save_state`/`load_state` via `m_data`) persist the squad's target/action state,
member list, smart/respawn ids, and flags atop the engine group packet (F15).

## Function digests (spine)

- **`sim_squad_scripted:update`** — *Does:* the per-squad tick; first-update hooks, refresh, online-status,
  target dispatch. *Fires:* `squad_on_first_update`/`_on_update`/`_on_after_*`.
- **`get_script_target`** — *Does:* resolve the scripted/condlist/random/hostage target → id, or nil.
- **`specific_update` / `generic_update`** — *Does:* the move→reach→stay FSM against a scripted vs an
  auto-selected target; smart-assign on arrival.
- **`simulation_objects.evaluate_prior`** — *Does:* score a candidate target (type gates → props×behaviour
  weights → ×1/dist). *Reads:* `simulation_objects_props.ltx`, `SIMBOARD.smarts`.
- **`assign_smart` / `assign_squad_member_to_smart`** — *Does:* (un)register members on the squad's smart.
  *(runtime override routes the squad-level one through `sim_board:assign_squad_to_smart`.)*
- **`check_online_status` / `can_switch_*`** — *Does:* compute + honor `forced_online_status` for the F16 switch.
- **target interface (`get_location`/`am_i_reached`/`on_reach_target`/`evaluate_prior`/…)** — *Does:* makes
  the squad a sim target (F84 `se_actor` shares this shape).

## Isolation ledger (the seams)

- **Engine base:** `cse_alife_online_offline_group` (`engine/server-entities.md`); `update`/`STATE_*`/
  `can_switch_*` call up to it.
- **sim_board seam (F86):** `SIMBOARD:get_squad_target` (target scoring), `assign_squad_to_smart`,
  `setup_squad_and_group`, `general_squad_precondition`, `SIMBOARD.smarts*`/`squads`. **The runtime override
  routes smart-assignment here.**
- **smart_terrain seam (F72/F73):** members are registered to a smart's job board on arrival; `register_npc`/
  `unregister_npc`.
- **simulation_objects seam:** the target registry + `evaluate_prior` scorer; `available_by_id` shared with
  `se_actor`; props from `misc\simulation_objects_props.ltx` + `ai_tweaks\simulation_objects.ltx`.
- **Registry seam (F69):** `db.offline_objects`/`db.spawned_vertex_by_id` cleared on reach; `db.storage`
  binder refs.
- **ALife switch seam (F16):** `forced_online_status` → `can_switch_*`.
- **Callback bus (F1):** `squad_on_first_update`/`_on_update`/`_on_npc_death`/`_on_after_game_vertex_change`/
  `_on_after_level_change`, `server_entity_on_register`/`_unregister`.
- **Adjacent modules (NOT this class):** `sim_squad_bounty` (bounty-squad spawn manager) + `sim_squad_warfare`
  (warfare metagame, F105) are separate files, not subclasses.

## Threading / failure / edge cases

- **`__lock` is an FPS guard:** a squad that found no target sets `__lock` so `generic_update` stops
  re-searching every tick. A "stuck" squad with no movement is usually `__lock`ed pending a valid target.
- **Anti-clusterfuck rule:** `evaluate_prior` returns 0 for a squad already targeting another squad — prevents
  chains of squads endlessly chasing each other.
- **Engine-first switch check:** `can_switch_online` *must* call the engine method before returning false, or
  an already-online object triggers "object with ID already exists" (noted in-code).
- **`sim_available` is false:** squads are not generically auto-targetable; only the scripted/condlist path or
  explicit help-target makes them a destination. The body's auto-availability logic is commented out.
- **`get_props` ignores squad-targeting-squad in CoC:** `evaluate_prior` still scores squads, but `get_props`
  early-returns for `online_offline_group_s` — squad-target props come from the precondition path, not props.
- **Runtime vs canonical:** always cite `gamedata/sim_squad_scripted.script` for the assignment path; the rest
  matches `unpacked/`.

## Deeper

- Squad class: `unpacked/scripts/sim_squad_scripted.script` (runtime: `gamedata/scripts/sim_squad_scripted.script`)
- Target scorer: `unpacked/scripts/simulation_objects.script` (`evaluate_prior`, `get_props`, level-link checks)
- Board (target supply + assignment): `sim_board.script` → `flows/...` (F86, pending)
- Jobs members land on: `flows/smart-terrain-core.md` (F72), `flows/gulag-jobs.md` (F73)
- Server-entity tier + the shared target interface: `flows/se-others.md` (F84, `se_actor`)
- Map: `modlayer/gameplay/squads-simulation.md` (M5)

## Maintenance

- Verified core (the update→target→FSM→assign loop, `evaluate_prior` scoring, the switch gate, the dual
  target role) is stable. The `idle_time`/`stay_time` staying logic + member spawn details are spine-read.
- **Always re-check the `gamedata/` override** before trusting the assignment path (currently the only
  functional delta: `assign_smart` → `sim_board:assign_squad_to_smart`).
- `SIMBOARD:get_squad_target` is the other half of target selection — document it in F86 (`sim_board`).
