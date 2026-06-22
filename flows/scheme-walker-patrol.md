---
domain: flows/scheme-walker-patrol
tags: [modlayer, gameplay, unpacked, lua, schemes, walker, patrol, flow]
sources: [unpacked/scripts/xr_walker.script, unpacked/scripts/xr_patrol.script, unpacked/scripts/xr_remark.script, unpacked/scripts/xr_logic.script]
related: [flows/scheme-engine-core.md, flows/state-manager-core.md, flows/logic-ltx-switching.md, flows/smart-cover.md, flows/script-npc-actions.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: walker / patrol / remark schemes — the scheme-authoring template (F74)

The simplest stalker behavior schemes, and the **canonical template every `xr_*` scheme follows**:
`set_scheme` parses config + binds, `add_to_binder` wires an evaluator + action into the NPC's GOAP
action manager, and the action's `execute` drives `move_mgr` (patrol movement) + `state_mgr` (animation,
F70) each frame. Read this once and the rest of the scheme catalog (F75–F82) reads as deltas on this
pattern. Built on the scheme engine F66 (`flows/scheme-engine-core.md`).

**Provenance:** the walker template is **verified** in `xr_walker.script` (`set_scheme`, `add_to_binder`,
`evaluator_need_walker:evaluate`, and the `execute` entry read). `xr_patrol` (group formation) and
`xr_remark` (stationary anim/dialog point) are documented as **deltas**, *(agent-traced — not personally
re-read)*. No `gamedata/` overrides of any of the three (verified by listing).

## The canonical scheme pattern (verified via walker)

Every scheme module exposes two namespaced functions + an evaluator class + an action class:

### `set_scheme(npc, ini, scheme, section, gulag_name)` (verified)
1. `st = xr_logic.assign_storage_and_bind(npc, ini, scheme, section)` — creates `db.storage[id][scheme]`
   and calls this module's `add_to_binder` (F66).
2. `st.logic = xr_logic.cfg_get_switch_conditions(ini, section, npc)` — the switch list (F68).
3. read the scheme's config fields onto `st` (walker: `path_walk`/`path_look` via
   `utils_data.prefix_r_string`, `sound_idle`, `use_camp`, `def_state_standing`/`def_state_moving` into
   `st.suggested_state`). Lazy fields (`path_walk_info`/`path_look_info`) are left nil for `reset_scheme`.

### `add_to_binder(npc, ini, scheme, section, storage)` (verified)
1. `manager = npc:motivation_action_manager()`.
2. define property + operator ids (walker: `need_walker = xr_evaluators_id.zmey_walker_base+1`,
   `state_mgr_logic_active = xr_evaluators_id.state_mgr+4`; operator `action_walker`).
3. create the **evaluator** (`evaluator_need_walker`) and **action** (`action_walker_activity`);
   `manager:add_evaluator(need_walker, evaluator)`.
4. **preconditions** on the action: `property_alive=true`, `property_danger=false`, `property_enemy=false`,
   `need_walker=true`, + `modules.add_common_precondition`. **effects:** `need_walker=false`,
   `state_mgr_logic_active=false`. `manager:add_action(action_walker, action)`.
5. `xr_logic.subscribe_action_for_events(npc, storage, action)` — register the action to receive
   `issue_event` (activate_scheme/deactivate/etc., F66).
6. gate the engine's `alife` action: add precondition `need_walker=false` to `manager:action(xr_actions_id.alife)`
   so generic alife wandering yields while the scheme is active.

### evaluator + action (verified)
- `evaluator_need_walker:evaluate` → `xr_logic.is_active(self.object, self.st)` (true when this scheme's
  section is the NPC's active section). This is how the planner knows to run the action.
- `action_walker_activity:execute` (verified entry) drives `move_mgr:update()` (advance the patrol),
  reads `state_mgr.get_state`, handles camp registration (`sr_camp`) + idle sound, and `set_state` for
  posture. `activate_scheme`/`reset_scheme` (re)parse the paths and `move_mgr:reset(...)`; `finalize`
  cleans up move_mgr + camp.

So: **a scheme = config-on-storage + an (evaluator, action) pair on the GOAP manager + a per-frame
execute that drives move_mgr/state_mgr.** Switching between schemes is F66's job (the action just runs
while its section is active).

## Deltas (agent-traced)

- **`xr_patrol`** — group patrol with a **formation**. Adds a `PatrolManager` (per `patrol_key =
  path+squad.id`) and splits into `action_commander` (leads, drives the formation) vs `action_patrol`
  (followers poll the commander for their relative slot ~1×/sec). Extra config: `formation`
  (line/around/back), `commander`, `silent`, `move_type`. Same template, two actions + a shared manager.
- **`xr_remark`** — **stationary** anim/dialog point: no `move_mgr`. Reads `anim` (a condlist),
  `snd`/`snd_anim_sync`, `target` (story/path/job → a look target), `tips_id`. Its action runs a small
  state machine (initial → animation → sound) via `state_mgr.set_state(npc, anim, callback, 0, target)`
  and sets `st.signals` (anim_end/sound_end/action_end) for logic switching.

## Function digests (spine)

- **`xr_walker.script:set_scheme`** — *Does:* the verified template above. *Calls:* `assign_storage_and_bind`,
  `cfg_get_switch_conditions`, `prefix_r_string`. *(verified.)*
- **`xr_walker.script:add_to_binder`** — *Does:* wire evaluator+action, gate alife. *Calls:* manager
  `add_evaluator`/`add_action`, `subscribe_action_for_events`. *(verified.)*
- **`xr_walker.script:evaluator_need_walker:evaluate`** — *Does:* `xr_logic.is_active`. *(verified.)*
- **`xr_walker.script:action_walker_activity:execute`/`reset_scheme`/`finalize`** — *Does:* drive
  move_mgr + state_mgr + camp. *(execute entry verified; sub-steps agent-traced.)*
- **`xr_patrol` / `xr_remark`** — *Does:* formation patrol / stationary remark deltas. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `db.storage[id][scheme]` (`logic`, paths, `suggested_state`, `signals`, the action/
  evaluator). Patrol adds the shared `PatrolManager` (`patrols[patrol_key]`).
- **Engine seam:** registers an evaluator+action on the NPC's `motivation_action_manager` (GOAP,
  `flows/ai-planner.md`); the planner runs the action when its evaluator is true.
- **Driven by:** F66 — `activate_by_section` calls `set_scheme`; `issue_event` calls the action's
  activate/deactivate; `try_switch_to_another_section` switches away.
- **Out:** `move_mgr` (patrol movement), `state_mgr` (F70 animation/posture), `sr_camp` (camp activity),
  `xr_sound` (idle sound). Remark also → `news_manager` (tips).

## Threading / failure / edge cases

- **`path_walk == path_look` errors** (verified guard) — and a missing `path_walk` patrol path is a hard
  `!ERROR`. Walker requires valid patrol paths.
- **Scheme yields to combat/danger:** the preconditions (`property_enemy=false`, `property_danger=false`)
  mean the action stops when the NPC has an enemy/danger — combat/danger schemes (F76) take over.
- **alife gating:** the `need_walker=false` precondition added to the alife action is what stops the NPC
  from wandering off-script; a scheme that forgets this lets alife compete.
- **Patrol formation needs a squad:** `patrol_key` includes `squad.id`; a lone NPC patrol degenerates.

## Deeper

- Template: `unpacked/scripts/xr_walker.script` (`set_scheme`, `add_to_binder`,
  `action_walker_activity:execute`/`reset_scheme`/`finalize`, `evaluator_need_walker`)
- Deltas: `unpacked/scripts/xr_patrol.script` (PatrolManager, commander/follower),
  `unpacked/scripts/xr_remark.script` (stationary state machine, `init_target`)
- Framework: `flows/scheme-engine-core.md` (activate/switch/issue_event); animation: `flows/state-manager-core.md`;
  switch list: `flows/logic-ltx-switching.md`; C++ action/evaluator side: `flows/script-npc-actions.md`

## Maintenance

- The verified walker template is the reference for all `xr_*` schemes — when documenting another scheme
  (F75/F77/F78/F81), describe only its delta over this and link here.
- Promote `xr_patrol`/`xr_remark` to verified on next read.
- Check `gamedata/scripts/` for overrides before trusting (none for these three at writing).
