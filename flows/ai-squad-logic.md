---
domain: flows/ai-squad-logic
tags: [modlayer, lua, ai, smart-terrain, gulag, xr-logic, flow]
sources: [unpacked/scripts/smart_terrain.script, unpacked/scripts/gulag_general.script, unpacked/scripts/xr_logic.script, unpacked/scripts/bind_stalker.script, unpacked/scripts/bind_monster.script, gamedata/scripts/bind_monster.script, unpacked/scripts/xr_remark.script, unpacked/scripts/xr_combat_ignore.script, unpacked/scripts/sim_squad_scripted.script, unpacked/scripts/modules.script]
related: [flows/ai-planner.md, flows/lua-callback-lifecycle.md, modlayer/scripting.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: AI squad / smart_terrain ↔ xr_logic seam

The **Lua** layer that sits on top of the engine GOAP planner: smart_terrain (gulag) hands each NPC a
job whose `logic` section selects an `xr_logic` scheme, which sets the NPC's state and — via the
binder — toggles engine script-control so the planner runs script-driven actions. This is the
high-level "what is this NPC doing here" orchestration, mostly **base-game Lua** (`unpacked/`) per the
base+override rule (`METHODOLOGY.md`).

**Provenance:** researched with a read-only agent, then spot-verified. File existence and the key
function homes (`xr_logic.configure_schemes`/`activate_by_section`/`issue_event`/
`try_switch_to_another_section`/`initialize_obj`; `se_smart_terrain:register_npc`/`select_npc_job`/
`setup_logic`/`setup_gulag_and_logic_on_spawn`; the `generic_object_binder:update` loop) are
personally verified by grep. **Correction to the raw agent trace: this repo *does* override
`bind_monster.script` — `gamedata/scripts/bind_monster.script` exists and is the LIVE binder** (it
wins over `unpacked/`), so the per-frame logic loop is cited from the repo override. The internal
behavior of each scheme is *(agent-traced — not personally re-read)*. Base Lua is read-only reference
(`unpacked/`); never edit it.

## The layering (verified homes)

Three layers stack on the engine object:

- **smart_terrain** (`unpacked/scripts/smart_terrain.script`, `se_smart_terrain`) — the job board: it
  registers NPCs (`register_npc`), picks a job per NPC (`select_npc_job`), and binds the job's logic
  (`setup_logic`); `setup_gulag_and_logic_on_spawn` is the spawn-time entry. Job definitions come from
  `gulag_general.script` (`load_job` + job tables/priorities). *(all homes verified; job-table
  internals agent-traced.)*
- **xr_logic** (`unpacked/scripts/xr_logic.script`) — the scheme engine: `configure_schemes` loads
  the logic ini + enables the generic schemes, `activate_by_section` selects and activates the scheme
  named by a section, `issue_event` dispatches scheme events (activate/update/reset/deactivate/
  callbacks), `try_switch_to_another_section` re-evaluates switch conditions each frame. *(all homes
  verified.)*
- **the binder** — `generic_object_binder` in **`gamedata/scripts/bind_monster.script`** (repo
  override, verified): its `update(delta)` calls `xr_logic.try_switch_to_another_section` then
  `xr_logic.issue_event(..., "update", delta)` for the active scheme, and routes engine callbacks
  (`death_callback`, `hit_callback`, `waypoint_callback`) into `issue_event`. The actor side is
  `bind_stalker.script` (`actor_binder`). The base `unpacked/scripts/bind_monster.script` is the
  canonical version this override replaces.

## The seam, in order (homes verified; scheme internals agent-traced)

1. **Spawn** — the NPC spawns (engine backbone, `flows/object-lifecycle.md`); the binder's net-spawn
   calls `smart_terrain.setup_gulag_and_logic_on_spawn`.
2. **Register + job select** — the smart_terrain (`se_obj.m_smart_terrain_id`) runs `register_npc` and
   `select_npc_job` to pick the highest-priority available job (by priority, precondition, NPC type,
   ownership) and assigns its `logic` section.
3. **Logic bind** — `setup_logic` calls `xr_logic.configure_schemes` (load ini, enable generic
   schemes) then `activate_by_section`.
4. **Scheme activate** — `activate_by_section` determines the scheme from the section name and calls
   that scheme's `set_scheme` (e.g. `xr_remark.set_scheme`), which reads its config from the ini
   section, registers a GOAP action/evaluator (`add_to_binder`), and `issue_event` fires
   `activate_scheme`.
5. **Per-frame** — the binder `update` runs `try_switch_to_another_section` (checks switch conditions:
   distance/info/timer/signal via condlists) and `issue_event(..., "update")` to drive the active
   scheme; a scheme that needs motor control enables engine **script-control** so the planner runs the
   script-driven action instead of pure GOAP (`flows/ai-planner.md` `GetScriptControl` fork).
6. **Online↔offline** — on going offline the binder saves the active section; on return it
   re-registers and re-binds (the ALife transition, `flows/object-lifecycle.md` / pending F16).

So GOAP still decides *when* to switch schemes; the Lua layer decides *which* scheme/section is active
and supplies the script-controlled behavior. `xr_logic.issue_event` is the dispatcher and
`xr_conditions`/`xr_effects` condlists are the in-section decision language.

## Isolation ledger (the seams)

- **Data read:** the NPC's `logic` ini section (from spawn_ini `[logic]` or the gulag-generated job
  ltx): scheme name, `active` condlist, switch conditions (`on_actor_dist_le`, `on_info`, `on_timer`,
  `on_signal`…), per-scheme params (anim/sound/path/target). smart_terrain config (job tables/
  priorities). *(section keys agent-traced.)*
- **Engine seam:** server entity `CSE_ALifeSmartZone` (the smart_terrain) + squad
  (`sim_squad_scripted` over the online/offline group); engine **script-control** toggled per scheme
  so the GOAP planner runs script actions (`flows/ai-planner.md`); the binder hooks the game object's
  callbacks (death/hit/waypoint) and forwards them to `issue_event`. *(CSE binding agent-traced/
  INFERRED.)*
- **Scheduling:** driven by the binder `update(delta)` (per-frame, on the object), plus the
  smart_terrain's own periodic `update` (job re-evaluation, stepped over NPCs).
- **Base vs override (verified):** smart_terrain / gulag_general / xr_logic / bind_stalker /
  xr_remark / xr_combat_ignore / sim_squad_scripted / modules are **base** (`unpacked/`, unmodified
  here). **`bind_monster.script` is overridden in `gamedata/scripts/`** — the live monster/NPC binder
  is the repo copy. (Confirmed: no gamedata override of the other spine files.)
- **Other flows:** sits on `flows/ai-planner.md` (the script-control seam + GOAP scheme-switch);
  uses the callback bus of `flows/lua-callback-lifecycle.md`; scheme `.script` model is
  `modlayer/scripting.md`.

## Threading

Lua runs on the main thread (engine drives the binder/smart_terrain updates from the scheduler).

## Failure & edge cases

- **The live binder is the repo override** (`gamedata/scripts/bind_monster.script`), not the
  `unpacked/` base — read the gamedata copy when changing per-frame NPC logic, or changes won't take.
- **GOAP vs scheme responsibility split:** "the NPC ignores my GOAP action" usually means a scheme has
  script-control; "the NPC won't change behavior" usually means a switch condition
  (`try_switch_to_another_section`) isn't met, not a planner bug.
- **Job assignment is priority/availability-based:** an NPC may not get the job you expect if a
  higher-priority job is free or the precondition fails (`select_npc_job`).
- **base files are reference only:** `unpacked/` is never loaded at runtime where a `gamedata/`
  override exists; don't reason about live behavior from a base file that's overridden.

## Deeper

- smart_terrain / jobs: `unpacked/scripts/smart_terrain.script` (`se_smart_terrain:register_npc`,
  `select_npc_job`, `setup_logic`, `setup_gulag_and_logic_on_spawn`),
  `unpacked/scripts/gulag_general.script` (`load_job`, job tables)
- scheme engine: `unpacked/scripts/xr_logic.script` (`configure_schemes`, `activate_by_section`,
  `issue_event`, `try_switch_to_another_section`, `initialize_obj`)
- binder (LIVE): `gamedata/scripts/bind_monster.script` (`generic_object_binder:update`); base:
  `unpacked/scripts/bind_monster.script`, `unpacked/scripts/bind_stalker.script` (`actor_binder`)
- example scheme: `unpacked/scripts/xr_remark.script` (`set_scheme`, `add_to_binder`);
  `unpacked/scripts/xr_combat_ignore.script`
- squads: `unpacked/scripts/sim_squad_scripted.script`
- engine entity: `CSE_ALifeSmartZone` (`engine/server-entities.md`)

## Maintenance

- Re-grep the verified function names in `xr_logic.script` / `smart_terrain.script` and re-check the
  `gamedata/scripts/bind_monster.script` override if AI logic changes; re-confirm which spine files are
  overridden in `gamedata/` (only `bind_monster.script` at time of writing).
- Promote the *(agent-traced)* scheme internals (job-table structure, per-scheme `set_scheme` bodies,
  the `CSE_ALifeSmartZone` binding) to verified by reading the base Lua / engine when a task needs it.
- Keep the base-vs-override correction prominent — it's the trap that makes edits to the base binder
  silently ineffective.
