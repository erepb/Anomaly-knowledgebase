---
domain: flows/helicopter-ai
tags: [modlayer, gameplay, unpacked, lua, ai, alife, flow]
sources: [unpacked/scripts/bind_heli.script, unpacked/scripts/heli_move.script, unpacked/scripts/heli_fly.script, unpacked/scripts/heli_look.script, unpacked/scripts/heli_fire.script, unpacked/scripts/heli_combat.script, unpacked/scripts/heli_alife.script, unpacked/scripts/heli_alife_warfare.script, unpacked/scripts/heli_snd.script]
related: [flows/vehicles-holder.md, flows/binder-architecture.md, flows/scheme-engine-core.md, flows/se-others.md, flows/sim-board.md, flows/scripted-squad.md, flows/smart-terrain-core.md, flows/scheme-combat.md, flows/db-registry.md, flows/alife-services.md, flows/warfare-metagame.md, modlayer/gameplay/vehicles-heli.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: helicopter AI (F95)

How a helicopter actually behaves: the engine `CHelicopter` (an AI aircraft, NOT a mountable holder —
F29) is driven entirely from Lua through **three stacked layers**: the `heli_binder` (lifecycle +
health), the **combat manager** (`heli_combat`, always-on, preempts everything), and below that either
**story logic** (the `heli_move` scheme, `stype_heli`) or the **ALife simulation layer** (`heli_alife`,
Alundaio) for faction-owned sim helis. `se_heli` (the server half) is F84.

**Provenance:** **verified, no agent** — `bind_heli`/`heli_move`/`heli_fly`/`heli_look`/`heli_fire`/
`heli_alife`/`heli_snd` READ-IN-FULL; `heli_combat` spine read (`__init`/`is_enemy`/`set_enemy`/
`find_valid_target` head/`forget_enemy`/`update_combat_type`/`update`), per-type bodies
(`round/search/flyby/retreat_update`, custom-data readers) **not read**; `heli_alife_warfare`
characterized by structure only. **No `gamedata/` override of any** (verified by listing).

## Registration & layering (verified)

`stype_heli = 3`; the **only** registered heli scheme is `heli_move` (+ the generic `xr_hit`, which is
LoadScheme'd for stalker/mobile/heli alike). `heli_fly`/`heli_look`/`heli_fire` are per-id helper
singletons (`get_heli_flyer/looker/firer` keyed by object id); `heli_combat` is instantiated per heli by
the binder (`st.combat`); `heli_alife(_warfare)` are plain modules called from the binder's update.

## `heli_binder` (read in full)

- `reinit`: creates `db.storage[id]`, caches `st.heliObject = get_helicopter()`, registers
  `helicopter_on_point`/`helicopter_on_hit` (+ key input callbacks), builds `st.combat`, reads hit/
  damage/down sound names from `spawn_ini()`; **`is_story_heli` = spawn ini has a `[logic]` section** —
  the story-vs-simulation fork.
- `update`: `xr_logic.initialize_obj(…, stype_heli)` + issue scheme `update`; **non-story** helis with a
  faction (`se_obj.player_id`) additionally run `heli_alife.update`; then `check_health` +
  `xr_sound.update`.
- `net_spawn`: non-story helis get smart-terrain logic via `setup_gulag_and_logic_on_spawn(…,
  stype_heli)` (Alundaio); the `alife/general/heli_spawn` option off → `self_destruct`. Registers in
  `db.add_obj` + `db.add_heli`.
- `check_health` (via `get_heli_health`, `invulnerable` pins 1): health < `flame_start_health` (0.1) →
  `StartFlame` + damage sound; ≤ 0.005 and not `immortal` → `heli_die` (engine `Die()`, callbacks off,
  `db.del_heli`) + smart `unregister_npc` + respawn-point count decrement. `heli_die` contains a
  hardcoded marsh-level task hack (`mar_base_owl_stalker_trader_task_2`).
- `on_hit`: feeds `heli_fire.enemy`, issues `hit_callback` to `st.hit` (the xr_hit generic), and — for
  sim helis — sets the **`heli_enemy_flag`** (actor pstor var / NPC storage flag): *shooting a faction
  heli is what makes you its valid enemy* (consumed by `heli_combat:is_enemy`). Also `hitted_by`,
  `helicopters_downed` achievements, `SendScriptCallback("heli_on_hit_callback")`.
- `on_point` → `waypoint_callback` to the active scheme. `save/load` = `xr_logic.save_obj` +
  `st.combat:save/load`.

## `heli_combat` — the always-on combat manager *(spine verified)*

`st.combat:update()` runs **before** both `heli_alife` and `heli_move` acts (move's `update` returns
while `st.combat.enemy_id` is set); returns true while fighting. Gates: a `combat_ignore` condlist,
**exclusive-job helis opt out** (their custom-data logic owns combat), surge or **night**
(`utils_obj.is_day`) forgets the enemy. `is_enemy`: alive stalker/monster, not wounded, not
combat-ignored (F76 overrides + safe-zone squads), different community than `se_obj.community`; the
**actor only qualifies via `heli_enemy_flag` or `actor_always_enemy`**; non-WARFARE NPCs also need their
own `heli_enemy_flag`. Targets are kept ≥25 s (`__keep_target_until`) to stop thrash. Combat types
**flyby ⇄ round ⇄ search** switch on visibility + distance vs `flyby_attack_dist`/`search_attack_dist`
(`update_combat_type`); in round mode the mgun **toggles in bursts** (~4–5 s on, 1–2 s off) via
`m_use_mgun_on_attack` + `SetEnemy/ClearEnemy`. Tuning (velocities, vis accumulation quanta,
`forget_timeout`, `combat_use_mgun/rocket`, `actor_always_enemy`) comes from the heli's **system-ini
section** (`ini_sys`), not logic. Per-type movement bodies + retreat *(not read)*.

## `heli_move` — the story scheme (read in full)

The single `stype_heli` scheme: patrol `path_move`/`path_look` (`path_look = actor` tracks the player;
`stop_fire` hovers in place while the actor is visible), or — with no patrol path — **coordinate nodes**
`pt1..ptN = "x,y,z[,delay][,signal]"` read straight from the logic section
(`create_path_nodes`/`iterate_nodes`) with a `path_end` condlist (`loop`/`reverse`/section switch);
`retreat_mode` 1/2 flies to the level bounding-volume corner (the despawn flight). Pushes weapon
config onto the engine object (`m_max/min_mgun/rocket_dist`, `m_use_mgun/rocket_on_attack`,
`UseFireTrail`) and drives `heli_fire` (`enemy = actor|all|<story-id>|nil`, `fire_point` fallback).
Movement: `update_movement_state` → `heli_fly:fly_on_point_with_vector(last, next, v)` → engine
`helicopter_on_point` → `waypoint_callback` (combat gets first refusal; waypoint `sig` → signals).
State persists via pstor (`save_var` st/li/ni/wc/path_index). `engine_sound` is double-gated by the
`alife/general/heli_engine_sound` option.

## Helpers (read in full)

- **`heli_fly`** — the flight planner: accelerate-vs-brake fork from `heliLAccFW/BW`, an intermediate
  point computed by **3-point Lagrange interpolation** (`calc_point`) for smooth curved approach,
  `correct_velocity` caps speed by remaining distance, `SetDestPosition`/`SetSpeedInDestPoint`.
- **`heli_look`** — velocity-weighted `LookAtPoint` while flying.
- **`heli_fire`** — scheme-combat enemy selection: `actor`/`all` (nearest visible from
  `db.heli_enemies`, 50 % bias to the actor)/story-id; dead or invisible enemy → re-pick; no enemy →
  fire at `fire_point`. The CustomStatic health-bar code is commented out (superseded by
  `ui_enemy_health`; `show_health` storage flag remains).
- **`heli_snd`** — see-enemy/hit/damaged/down voice banks + priority `play_snd`; consumed only from
  `xr_effects` (the binder plays its hit/damage/down lines through `xr_sound` instead).

## `heli_alife` — the simulation layer (read in full)

Alundaio's Helicopter Alife Simulator, run from the binder for non-story faction helis. Same
target-FSM shape as squads (F85): `current_action` REACH(1) → `try_to_reach_target` (climb to target
+70 alt first, then `SetDestPosition` for smarts / `GoPatrolByRoundPath` circling for squads) until
`is_target_reached` (same level + ≤5 m 2D, squads count as reached) → STAY(0) for 6800 game-seconds
(`update_heli_movement`: slow 12-velocity circle above the squad commander at max level height) →
re-pick. Target pick `get_smart_terrain_target` = the **same `simulation_objects` registry the squads
use** (F85/F86): top-5 by `evaluate_prior` where `target.props[player_id] > 0`, else its own smart.
Smart-terrain population is honored: `register_npc`/`unregister_npc` on arrive/leave (F72).
`force_target_destination(id, target_id)` is the **help-signal API** (same-community only) — used by
`axr_radio_in_heli` (NPCs radio for air support). Surges clear targets/help. `heli_alife_warfare` is a
modified copy for WARFARE mode (`create_heli_weak` faction spawns) *(structure only)*.
**Refined by F105 (verified):** the swap that would install it —
`warfare_options.override_functions` assigning `heli_alife.update = heli_alife_warfare.update` —
is dead code (no caller), so warfare-purchased helis are *created* by `heli_alife_warfare` but
*updated* by this stock module. See `flows/warfare-metagame.md`.

## Isolation ledger (the seams)

- **Engine (F29):** `CHelicopter` via `get_helicopter()` — `SetDestPosition`/`SetMaxVelocity`/
  `GoPatrolByRoundPath`/`LookAtPoint`/`SetEnemy`/`StartFlame`/`Die`, `helicopter_on_point/on_hit`
  callbacks; NOT a holder (no player mounting).
- **Binder (F88):** `heli_binder` = the F88 contract + `stype_heli`; `se_heli` server half = F84.
- **Scheme engine (F66/F68):** `heli_move` + generic `xr_hit`; coordinate-node and `path_end`
  condlists.
- **Simulation world (F85/F86):** `heli_alife` picks targets from `simulation_objects` with
  squad-style `evaluate_prior`; smart population via `register/unregister_npc` (F72).
- **Combat-ignore (F76):** `is_enemy` honors `ignore_enemy_by_overrides` + safe-zone squads.
- **Registries (F69):** `db.add_heli`/`db.heli_enemies`; `heli_enemy_flag` in actor pstor/NPC storage.
- **Options (F17):** `alife/general/heli_spawn` (self-destruct) + `heli_engine_sound`.
- **AXR seams:** `axr_radio_in_heli` → `force_target_destination`; `axr_npc_vs_heli` (NPCs engaging
  helis) reads `heli_alife.distance_2d_sqr`.

## Threading / failure / edge cases

- **Helis don't attack the actor by default** — `heli_enemy_flag` (set when you hit one) or
  `actor_always_enemy` in the section config is required; in non-WARFARE the same applies to NPCs.
- **Combat stops at night and during surges** (`is_day` gate in `heli_combat:update`) — a passive heli
  at night is by design.
- **Layer priority is combat → alife/scheme:** while `st.combat.enemy_id` is set, `heli_move` does
  nothing and `heli_alife.update` returns early — story logic can't steer a fighting heli without
  `combat_ignore`.
- **Exclusive-job helis** skip the shared combat manager entirely — their job logic must handle combat.
- **`heli_move` without `path_move`** silently falls back to coordinate nodes (`pt1…`) or hovers;
  a missing patrol logs and returns rather than aborting.
- **`heli_die` has a hardcoded marsh task hook** — don't be surprised by task completion on heli death
  on `k00_marsh`.
- **No `gamedata/` override** of any heli file at writing — re-check per session.

## Deeper

- Binder/lifecycle: `unpacked/scripts/bind_heli.script` (`heli_binder`, `check_health`, `heli_die`,
  `get_heli_health`)
- Combat: `heli_combat.script` (`is_enemy`, `update`, `update_combat_type`, combat types)
- Story scheme: `heli_move.script` (`create_path_nodes`/`iterate_nodes`, `waypoint_callback`);
  helpers `heli_fly.script` (`fly_on_point_with_vector`/`calc_point`), `heli_look.script`,
  `heli_fire.script` (`set_enemy`/`update_enemy_arr`), `heli_snd.script`
- Simulation: `heli_alife.script` (`update`, `get_smart_terrain_target`, `force_target_destination`),
  `heli_alife_warfare.script` (WARFARE copy)
- Engine class: `flows/vehicles-holder.md` (F29); server entity: `flows/se-others.md` (F84, `se_heli`)
- Map: `modlayer/gameplay/vehicles-heli.md` (M12)

## Maintenance

- Promote `heli_combat` per-type movement bodies (`round/search/flyby/retreat_update`,
  `calc_position_in_radius`) and `heli_alife_warfare` from not-read when a task needs them.
- Re-check `gamedata/scripts/heli_*`/`bind_heli` for overrides per session (none at writing).
