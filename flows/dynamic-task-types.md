---
domain: flows/dynamic-task-types
tags: [modlayer, gameplay, unpacked, lua, task, quest, flow]
sources: [unpacked/scripts/tasks_clear_map.script, unpacked/scripts/tasks_bounty.script, unpacked/scripts/tasks_defense.script, unpacked/scripts/tasks_fetch.script, unpacked/scripts/tasks_assault.script, unpacked/scripts/tasks_guide.script, unpacked/scripts/tasks_measure.script, unpacked/configs/misc/task/tm_deadcity.ltx]
related: [flows/task-system-lua.md, flows/condlist-library.md, flows/lua-callback-lifecycle.md, flows/sim-board.md, flows/scripted-squad.md, flows/tasks.md, flows/axr-managers.md, flows/treasures-secrets.md, modlayer/gameplay/tasks-lua.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: dynamic task types (`tasks_*`) (F98)

How the ~22 `tasks_*.script` files (bounty/assault/defense/delivery/fetch/clear_map/guide/measure/…)
implement repeatable quest *types* on the F97 core. There is **no class framework** — each file is a
**plugin** that injects named functions into the shared registries the F97 core and condlist engine
already resolve, plus (for the simulation-heavy types) its own script-callback-driven world logic.

**Provenance:** **verified, no agent** — pattern established from `tasks_clear_map.script`
READ-IN-FULL + `tasks_bounty.script` head/structure + a function-shape survey of all 22 files +
callback-wiring greps (`tasks_defense`/`tasks_fetch`) + one config consumer verified end-to-end
(`tm_deadcity.ltx` clear-map task). The other ~19 file bodies **not read** — they are leaves on this
pattern. **No `gamedata/` override of any** (verified by listing).

## The plugin pattern (verified)

Each `tasks_X.script` does some subset of:

1. **Namespace injection** — defines functors/conditions/effects directly into the other modules'
   tables, in two equivalent styles: `function task_functor.mutants_in_map_target(…)` (clear_map) or
   `xr_effects.setup_bounty_task = function(…)` (bounty). Injected names land in `task_functor` /
   `task_status_functor` (resolved by F97's `CGeneralTask`) and `xr_conditions` / `xr_effects`
   (resolved by the condlist engine, F67). Census across all 22: ~5 task_functor, ~15 xr_conditions,
   ~12 xr_effects in declaration style, plus assignment-style ones (bounty etc.).
2. **Config consumption** — `tm_*.ltx` task sections reference the injected names:
   `target_functor = mutants_in_map_target`, `condlist_0 = {!mutants_in_map(l09_deadcity)}
   %=set_task_stage(…)%` etc. **The `target` ini field is reused as the functor's argument** (the
   config comments admit "abusing this to pass map name") — `p` in `f(task_id, field, p, tsk)` is
   whatever `target =` says.
3. **Multi-step via `stage`** — `tsk.stage` is the step counter: `xr_conditions.task_stage_is(_not)`
   + `xr_effects.set_task_stage`; functors branch on `tsk.stage` (clear_map's target returns the
   task giver at stage 1, else the next mutant squad).
4. **Script-callback simulation** (the heavy types) — `on_game_start()` registers F1-bus callbacks:
   `actor_on_first_update`, **`save_state`/`load_state`** (own tables into the marshal `m_data`,
   independent of F97's task persistence), `on_before_surge`/`on_before_psi_storm` (defense skips
   weather), `on_before_level_changing` (fetch), npc/squad update hooks. This is where defense waves
   get spawned, assault squads evaluated, measurement HUDs run.
5. **Setup effects from dialog** — the type's `xr_effects.setup_X_task(actor,npc,p)` (fired from the
   task's `on_init` or the offering dialog) picks the concrete target (e.g. bounty: scan
   `SIMBOARD.squads` for a victim squad of the configured factions, excluding base-smart squads and
   blacklisted/underground maps) and stashes it; the paired generic functor
   (`task_functor.general_bounty_task`) renders title/target from that stash.

## Sub-pattern census (structure verified per file; bodies not read)

- **Functor-only**: `tasks_clear_map` *(read in full — the canonical example)*,
  `tasks_anomaly_scanner`, `tasks_faction_control` — pure injections, no callbacks.
- **Target-picker + sim scan**: `tasks_bounty` (squad scan w/ faction list + map blacklist),
  `tasks_assault`/`tasks_dominance` (smart/squad evaluation + next-frame postpone helpers),
  `tasks_fate`, `tasks_smart_control`.
- **Full simulations**: `tasks_defense`/`tasks_pump_station_defense` (wave spawns, chatter, surge
  skip), `tasks_agent_rescue` (squad loadouts, join/leave messaging), `tasks_measure` (own 3D
  `UI3D_Anomaly` HUD class), `tasks_guide` (cross-level route collector with own save/load),
  `tasks_chimera_scan`, `tasks_recover_*`.
- **Item logistics**: `tasks_fetch`/`tasks_multifetch`/`tasks_delivery`/`tasks_stash`
  (has_item checks, level-change hooks, `xr_effects.multifetch_remove`).
- Side effect worth knowing: clear_map's `mutants_in_map` sets `force_online = true` on every
  matching squad — the task itself drags target squads online.

## Isolation ledger (the seams)

- **F97 core:** functor names resolved from `task_functor[...]`/`task_status_functor[...]`;
  `tsk.stage`/`task_giver_id` read from `CGeneralTask`; outcomes via the same `condlist_N`.
- **Condlists (F67):** injected `xr_conditions.*`/`xr_effects.*` become ordinary condlist vocabulary.
- **Callback bus (F1):** `RegisterScriptCallback` for sim ticks + own `save_state`/`load_state`
  (marshal m_data, F15 via alife_storage_manager's bus).
- **Simulation world (F85/F86):** bounty/assault scan `SIMBOARD.squads`/smarts;
  `simulation_objects.base_smarts` filters.
- **Dialogs (F18):** setup effects fire from offering dialogs; `axr_task_manager` (F114, ⬜) is the
  randomizer/registry that picks which `tm_*` task a giver offers.

## Threading / failure / edge cases

- **Injection order matters:** these files mutate `xr_conditions`/`xr_effects`/`task_functor` tables
  at load time — a same-name collision between two plugins silently overwrites (last loaded wins).
- **`target` is an argument slot, not always an object id** — don't assume `target =` in a `tm_*`
  section is a story id; check the functor.
- **Each sim plugin persists its own state** independently of F97 — a "lost" dynamic task after load
  is usually the plugin's `load_state`, not the task core.
- **clear_map forces squads online** — population/perf side effect of merely having the task.
- **No `gamedata/` override** of any `tasks_*` at writing — re-check per session.

## Deeper

- Canonical example: `unpacked/scripts/tasks_clear_map.script` (whole pattern in 94 lines) +
  consumer `unpacked/configs/misc/task/tm_deadcity.ltx` (merc barman task 5)
- Target-picker style: `tasks_bounty.script` (`xr_effects.setup_bounty_task`,
  `task_functor.general_bounty_task`)
- Sim style: `tasks_defense.script` (`on_game_start` callback set), `tasks_fetch.script`
  (save/load + level-change), `tasks_measure.script` (`UI3D_Anomaly`)
- Core: `flows/task-system-lua.md` (F97); map: `modlayer/gameplay/tasks-lua.md` (M13)

## Maintenance

- Per-type bodies (~19 files) are not read — promote individually when a quest type needs work; the
  plugin pattern + the clear_map exemplar are the verified spine.
- New `tasks_*` files follow the same pattern; add them to the census here.
- Re-check `gamedata/scripts/tasks_*` for overrides per session (none at writing).
