---
domain: flows/task-system-lua
tags: [modlayer, gameplay, unpacked, lua, task, quest, flow]
sources: [unpacked/scripts/task_manager.script, unpacked/scripts/task_objects.script, unpacked/scripts/task_functor.script, unpacked/scripts/task_status_functor.script, unpacked/configs/misc/task/task_manager.ltx, unpacked/scripts/bind_stalker.script, unpacked/scripts/bind_stalker_ext.script]
related: [flows/tasks.md, flows/condlist-library.md, flows/logic-ltx-switching.md, flows/binder-architecture.md, flows/dialog-trade.md, flows/pda-map-news.md, flows/relations-info.md, flows/save-load.md, flows/dynamic-task-types.md, flows/axr-managers.md, modlayer/gameplay/tasks-lua.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: task system core (Lua) (F97)

How quests actually work above the engine: the engine `CGameTask` (F38) is a dumb polled shell —
**all quest logic lives in Lua**. `task_manager.CRandomTask` (a singleton) owns active
`task_objects.CGeneralTask` instances built from `tm_*.ltx` config; per-frame functors re-evaluate
title/description/target live, condlists decide complete/fail, and the engine's two registered
functors just read back the verdict. Dynamic task *types* (`tasks_*`) are F98; the Alundaio extender
`axr_task_manager` (bounties, faction questlines) is F114 (`flows/axr-managers.md`) — both sit on this core.

**Provenance:** **verified, no agent** — `task_manager.script` + `task_objects.script` READ-IN-FULL;
`task_functor`/`task_status_functor` structure + generic functors verified, story-specific bodies
**not read** (a catalog of ~40+10 per-quest functions); wiring (tick site, callback registration,
config includes) verified by grep. **No `gamedata/` override of any** (verified by listing).

## Config (verified)

`misc\task\task_manager.ltx` `#include "tm_*.ltx"` (wildcard — drop-in task packs). Each `[task_id]`
section: `title`/`descr`/`target` + `*_functor` names (default `condlist`/`target_condlist`),
`condlist_0..N` (outcome conditions returning `complete`/`fail`/`reversed`), `status_functor`,
`on_init`/`on_complete`/`on_fail`/`on_cancel`/`on_reversed` effect condlists, `reward_money`/
`reward_item` (note: `reward_item` is read from the **`reward_money`** ini line — a verified bug, item
rewards via this field only work if money and items share a line), `storyline`, `prior`, `icon`,
`repeat_timeout`, `dont_send_update_news`.

## Lifecycle (verified end-to-end)

1. **Give** — `xr_effects.give_task` (dialog/logic effect) or story scripts →
   `CRandomTask:give_task(task_id, giver_id)` → `CGeneralTask(task_id)` parses the section and runs
   the title/descr/target functors once → `CGeneralTask:give_task()` builds the engine `CGameTask`
   (`set_id/set_type` storyline-vs-additional, title/descr/prior/icon) and registers the **two engine
   functors** `add_complete_func("task_manager.task_complete")` + `add_fail_func("task_manager.task_fail")`
   — the F38 `CheckFunctions` seam. Map spot (`storyline_task_location` vs `secondary_task_location`)
   + blink on the target; storyline tasks also flash a fade message. Runs `on_init`.
2. **Tick** — the actor binder (`bind_stalker:update`) calls `get_task_manager():update()` → per task
   `check_task()` (self-throttled 500–1000 ms): re-runs the three functors; a changed title/descr is
   pushed into the engine task, a changed target swaps the map location (`change_map_location` +
   blink) — **quest text and markers are live-recomputed, not static**. Then `condlist_N` are
   evaluated (F67/F68) and the first `complete|fail|reversed` lands in `last_check_task`; else the
   `status_functor` (kill/loot/time counters in `task_status_functor`) gets a shot.
3. **Engine poll** — `CGameTask::UpdateState` (F38) calls the registered Lua functors;
   `task_complete`/`task_fail` translate `last_check_task` + `forced_status` into a bool. So Lua
   decides, the engine just polls.
4. **State change** — the engine fires the actor `task_state` callback → `bind_stalker`'s callback
   registry → `bind_stalker_ext.actor_on_task_callback` → `task_manager.task_callback`: fail →
   `on_fail` (or `on_cancel` if `cancelled_by_actor`) + news + statistics; complete → `on_complete` +
   `give_reward()` + statistics + toolkit-reward counter. **Repeatable tasks** (`repeat_timeout`)
   reset to a timeout state (stage 255) instead of being dropped; `CRandomTask:update` later removes
   them once the cooldown elapses (with a gossip news ping) — that's the repeating-task loop.
5. **Forcing** — `set_task_completed/failed/cancelled` set `forced_status`, consumed on the next
   engine poll. This is how dialogs/`xr_effects` short-circuit outcomes.

## Functor catalogs (structure verified; story bodies not read)

- **`task_functor`** — `f(task_id, field, p, tsk)` → display string or target object id. Generic:
  `condlist` (evaluate ini condlist for title/descr), `target_condlist`, `nil_target`. ~40
  story/warfare-specific ones (guides, escorts, faction-base defense, surge/psi-storm/fallout tasks).
- **`task_status_functor`** — `f(tsk, task_id)` → `complete|fail|reversed`. Generic counters:
  `check_if_killed_enough`, `check_if_looted_enough`, `check_if_enough_time_passed`; plus per-quest
  specials.
- A missing/misnamed `target_functor` logs and leaves the task target-less (no abort).

## Rewards & guiders (verified)

`give_reward` resolves `reward_money`/`reward_item` condlists and transfers via
`dialogs.relocate_money/relocate_item_section` **to `get_speaker()`** — i.e. rewards through this
path only pay out while a dialog is open (the ini comment says as much); outside dialogs it returns
silently. Most Anomaly rewards instead flow through `axr_task_manager`/dialog effects (F114).
`check_level` puts a "talk to the guide" spot on a guider NPC when the target is on another level —
but `guiders_by_level` only knows the CoP trio (zaton/jupiter/pripyat); on every other Anomaly level
the guider feature is inert.

## Persistence (verified)

Two paths switched by `USE_MARSHAL` (`marshal ~= nil`, the modded-exe Lua serializer — true here):
marshal path = `save_state/load_state` into `alife_storage_manager`'s `m_data`
(`m_data.task_info` + `m_data.task_objects[id]` plain tables), called from `bind_stalker`'s
save/load; legacy path = hand-rolled packet `save/load` (status byte, times, current strings, stage).
Completed/failed repeatables persist only their `timeout`. Per-task pstor (`save_var(db.actor, id)`)
is wiped on completion/failure.

## Isolation ledger (the seams)

- **Engine task (F38):** `CGameTask` + `db.actor:give_task`/`get_task`; `add_complete/fail_func`
  polled by `UpdateState`; `task_state` actor callback; `task.storyline/additional` enum.
- **Actor binder (F88):** the tick site (`bind_stalker:update`) + the `task_state` callback route +
  save/load hooks.
- **Condlists (F67/F68):** `condlist_N`, all `on_*` effects, reward condlists, generic functors.
- **Map/news (F39):** `level.map_add_object_spot`/`change_map_location` spots + blink;
  `news_manager.send_task` (updated/fail/reversed).
- **Dialog (F18):** `give_reward` pays the current speaker; `xr_effects.give_task` is mostly fired
  from dialogs.
- **Storage (F15):** `alife_storage_manager` m_data (marshal) vs packet path.
- **Extender (F114, ⬜):** `axr_task_manager` (bounties/questlines) sits on this core;
  `xr_conditions.has_task/has_task_not_completed/...` are the condlist surface.

## Threading / failure / edge cases

- **Tasks are monolithic** (F38: no objectives) — multi-step quests are faked by functors swapping
  title/descr/target per `stage` (the `stage` field is the de-facto step counter, mutated by story
  functors/dialogs).
- **`reward_item` reads the `reward_money` line** — config item rewards through `CGeneralTask` are
  broken as authored; use dialog effects instead.
- **Rewards need an open dialog** (`get_speaker()`), by design.
- **Repeatables never leave `task_info`** until the cooldown removal in `CRandomTask:update`, and
  removal is blocked while the engine task object still exists.
- **`give_task` on an existing id** logs an error and overwrites the Lua object (the engine task is
  reused via `get_task(id, true)`).
- **Cross-level guider spots are CoP-only** (`guiders_by_level`).
- **`task_complete/task_fail` return false during precache/no-actor** — outcomes can't land during
  level load.
- **No `gamedata/` override** of the four files at writing — re-check per session.

## Deeper

- Manager: `unpacked/scripts/task_manager.script` (`CRandomTask`, `task_callback`,
  `save_state/load_state`)
- Task class: `task_objects.script` (`CGeneralTask:give_task/check_task/give_reward/check_level`)
- Functors: `task_functor.script` (`condlist`/`target_condlist` + story catalog),
  `task_status_functor.script` (counters)
- Config: `unpacked/configs/misc/task/task_manager.ltx` (+ `tm_*.ltx` packs)
- Wiring: `bind_stalker.script` (tick + `task_state` route + save/load),
  `bind_stalker_ext.script` (`actor_on_task_callback`), `xr_effects.script` (`give_task`)
- Engine half: `flows/tasks.md` (F38); map: `modlayer/gameplay/tasks-lua.md` (M13)

## Maintenance

- Story-specific functor bodies (~50) are not read — promote individually when a quest task needs
  them. The core spine is read-in-full and stable.
- F98 (`tasks_*` dynamic types) and F114 (`axr_task_manager`) build on this — keep cross-links when
  they land. Re-check `gamedata/scripts/task_*` for overrides per session (none at writing).
