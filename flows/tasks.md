---
domain: flows/tasks
tags: [xrgame, task, quest, objective, info-portion, pda, map, lua, flow]
sources: [src/xrGame/GameTask.cpp, src/xrGame/GameTask.h, src/xrGame/GametaskManager.cpp, src/xrGame/GametaskManager.h, src/xrGame/alife_smart_terrain_task.cpp, src/xrGame/stalker_alife_task_actions.cpp]
related: [flows/relations-info.md, flows/dialog-trade.md, flows/ai-squad-logic.md, flows/save-load.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: tasks (player quests)

How a player quest is created, given, evaluated each frame, and completed/failed. The engine owns the
task object + the per-frame update loop; **Lua (gamedata task_manager) configures and gives tasks**,
and **completion is gated on info-portions** (`flows/relations-info.md`) and/or Lua functors. This is
the C++ spine behind Anomaly's quest system. Note a naming trap: the unrelated **ALife
smart-terrain "task"** (an NPC movement target) is a different thing — covered briefly at the end.

**Provenance:** spine personally code-verified — `CGameTask::UpdateState` (the ordered completion
gates), `CheckInfo` (HasInfo-all), the **monolithic model** (no `m_Objectives` in `GameTask.h` — only
`m_completeInfos`/functors), the `eTaskStateChange` callback site, and the existence of
`CGameTask::SetTaskState` + `CGameTaskManager::GiveGameTaskToActor`/`UpdateTasks`/`SetTaskState`/
`ActiveTask`. The manager bodies (sort/active/map/UpdatePda), the Lua script surface, map-location
create/remove, save/load via the registry, `CheckFunctions`' exact combine, and the ALife
smart-terrain task/action are confirmed homes but *(agent-traced — not personally re-read)*.

## Data model (verified: monolithic, no objectives)

A **`CGameTask`** is one quest: id (`m_ID`), title/description (Lua-overridable), `m_task_state`
(`eTaskStateInProgress`/`eTaskStateCompleted`/`eTaskStateFail`), `m_task_type` (storyline/additional),
priority, icon, timing (`m_ReceiveTime`/`m_TimeToComplete`/`m_FinishTime`), a PDA map binding
(`m_linked_map_location`), and its **completion/fail conditions**: `m_completeInfos`/`m_failInfos`
(info-portion id lists) + `m_complete_lua_functions`/`m_fail_lua_functions` (functor lists), plus
"on-complete/on-fail" infos to grant and functors to call.

**Anomaly tasks have no sub-objectives.** Unlike vanilla X-Ray (which had an `SGameTaskObjective`
vector), `GameTask.h` here carries only the flat condition lists — a task is atomic. Multi-stage quests
are built Lua-side by chaining task gives or by swapping which info-portions are checked. Don't
document a per-objective API; it isn't here.

## Give → update → complete (verified spine; manager bodies agent-traced)

**Give:** Lua builds a `CGameTask` (sets id/title/conditions/map), then `actor:give_task(task, ttl,
check_existing, timer)` → `CGameTaskManager::GiveGameTaskToActor` (verified present): add to the task
registry, stamp receive time + deadline, mark in-progress, create the PDA map location, sort by
priority, set it active, fire the state-change callback, refresh the PDA. *(body agent-traced.)*

**Update (per frame):** `CGameTaskManager::UpdateTasks` (verified present) iterates in-progress tasks
and calls **`CGameTask::UpdateState()`** (verified) — the ordered gate:
1. **timeout** — if a deadline was set (`m_ReceiveTime != m_TimeToComplete`) and `GetGameTime() >
   m_TimeToComplete` → `eTaskStateFail`.
2. `CheckInfo(m_failInfos)` → Fail; `CheckFunctions(m_fail_lua_functions)` → Fail.
3. `CheckInfo(m_completeInfos)` → Completed; `CheckFunctions(m_complete_lua_functions)` → Completed.
4. else keep current state.
On a change the manager calls `SetTaskState` → `CGameTask::SetTaskState` (remove map location, grant the
on-complete/fail info-portions, call the on-complete/fail functors, fire the callback), clears the
active task, re-sorts, refreshes the PDA. *(SetTaskState body agent-traced.)*

**The completion check is the F40 seam (verified):** `CGameTask::CheckInfo(v)` returns true **iff the
actor `HasInfo` for every id in `v`** (loops, breaks false on the first missing) — i.e. a task
completes when the actor knows all of its `complete_info` portions. Those portions are set by dialog,
scripts, or `TransferInfo` (`flows/relations-info.md`). `CheckFunctions` calls each Lua
`functor(m_ID)`; *(exact AND/last-result combine agent-traced.)* Because completion is polled here, the
info→task link is **pull, not push** — the task system does not ride the actor-only
`actor_on_info_callback` (see the refined info-callback fact in `flows/relations-info.md`).

## Callback + Lua surface (callback verified; surface agent-traced)

State changes fire **`Actor()->callback(GameObject::eTaskStateChange)(this, GetTaskState())`**
(verified) — the per-object callback Lua listeners hook to react to give/complete/fail. The Lua surface
*(agent-traced)*: `CGameTask` is bound with `set_id`/`set_title`/`set_priority`/`set_type`/
`set_map_location`/`set_icon_name` and the condition builders `add_complete_info`/`add_fail_info`/
`add_on_complete_info`/`add_complete_func`/… (+ `task.task_state.{in_progress,completed,fail}` /
`task.task_type.{storyline,additional}` enums); actor-side `give_task`/`set_task_state`/`get_task_state`/
`set_active_task`/`is_active_task`/`get_task`. So Lua owns *content*; C++ owns *lifecycle + the update
poll*.

## Map / PDA (agent-traced)

A task owns a `CMapLocation` (PDA waypoint): `CreateMapLocation` binds it via `Level().MapManager()`
(reverse-linked by `m_owner_task_id`), `RemoveMapLocations` on complete/fail, `ChangeMapLocation` to
move it. `CGameTaskManager::MapLocationRelcase` handles the map→task back-reference when a location is
destroyed. UI refresh via `CurrentGameUI()->UpdatePda()`. The map subsystem itself is the (planned)
PDA/map flow.

## ALife smart-terrain task — a DIFFERENT thing (agent-traced)

`CALifeSmartTerrainTask` (`alife_smart_terrain_task.cpp`) is **not** a player quest: it is an **NPC
movement destination** — a patrol-path point or game/level-graph vertex an AI is directed to occupy
(`CStalkerActionSmartTerrain` in `stalker_alife_task_actions.cpp` is the GOAP action that walks the NPC
there). It does not touch `CGameTaskManager`, fire `eTaskStateChange`, or bind a PDA location. It
belongs to the smart-terrain/AI scheduling layer (`flows/ai-squad-logic.md`), filed here only to
disambiguate the word "task."

## Isolation ledger (the seams)

- **Info-portions (F40):** completion/fail polled via `Actor()->HasInfo` over the task's info lists;
  on-complete/fail infos are granted on state change (`flows/relations-info.md`).
- **Dialog:** dialog choices set the info-portions that complete tasks (`flows/dialog-trade.md`) — the
  link is the shared info-portion, not a direct dialog→task call.
- **PDA/map:** task ↔ `CMapLocation` (waypoint), `UpdatePda` on change.
- **Lua:** `eTaskStateChange` callback (react to give/complete/fail); the `CGameTask`/actor task API
  (configure + give). gamedata `task_manager` scripts are the actual quest definitions (not in the C++
  tree).
- **Save/load:** tasks persist via the task registry wrapper in the ALife save (`flows/save-load.md`).
- **AI (disambiguation only):** `CALifeSmartTerrainTask` is the NPC-movement "task", unrelated to player
  quests (`flows/ai-squad-logic.md`).
- **Config:** task definitions are Lua-side (gamedata `task_manager`), not an engine XML; the engine
  reads only what Lua sets on the `CGameTask`. PDA layout in `ui/` XML.

## Threading

Main thread (`UpdateTasks` on the level update; Lua single-threaded).

## Failure & edge cases

- **Completion is polled, not pushed:** a task only re-checks its info-portions in `UpdateTasks`; there
  is no event when an info is granted. A task whose `complete_info` is already satisfied completes on the
  next update tick, not instantly at the dialog line.
- **`CheckInfo` is AND-all:** every id in the list must be present; one missing keeps the task open. An
  empty info list never completes via info (needs a functor or timeout).
- **Deadline only if set:** the timeout fail path is skipped unless `m_ReceiveTime != m_TimeToComplete`
  (i.e. a TTL was given); most tasks have no deadline.
- **No objectives:** scripting a "step 2 of 3" means giving a new task or changing checked infos — there
  is no objective array to advance. Documenting one would be fiction.
- **Smart-terrain task confusion:** searching "task" hits the AI movement target too — keep the two
  separate (player `CGameTask` vs `CALifeSmartTerrainTask`).

## Deeper

- Task object: `src/xrGame/GameTask.cpp` (`UpdateState`, `CheckInfo`, `CheckFunctions`, `SetTaskState`,
  `CreateMapLocation`/`RemoveMapLocations`/`ChangeMapLocation`, the `eTaskStateChange` callback),
  `GameTask.h` (`m_completeInfos`/`m_failInfos`, `m_*_lua_functions`, state/type enums — no objectives)
- Manager: `src/xrGame/GametaskManager.cpp` (`GiveGameTaskToActor`, `UpdateTasks`, `SetTaskState`,
  `ActiveTask`, `SetActiveTask`, `MapLocationRelcase`), `GametaskManager.h` (the task registry)
- ALife NPC task (separate): `src/xrGame/alife_smart_terrain_task.cpp` (`CALifeSmartTerrainTask`),
  `stalker_alife_task_actions.cpp` (`CStalkerActionSmartTerrain`) — `flows/ai-squad-logic.md`
- Gates/consumers: `flows/relations-info.md` (info-portions), `flows/dialog-trade.md`,
  `flows/save-load.md`

## Maintenance

- Re-grep `CGameTask::UpdateState` if the completion logic changes — keep the ordered gate (timeout →
  fail-info → fail-func → complete-info → complete-func) and the HasInfo-all `CheckInfo` accurate; this
  is the F40 info-portion → task seam.
- Keep the "no objectives (monolithic), completion is polled" facts — both contradict vanilla X-Ray and
  are common misconceptions.
- Keep the player-`CGameTask` vs ALife-`CALifeSmartTerrainTask` disambiguation.
- Promote the *(agent-traced)* manager bodies, Lua surface, and map-location wiring to verified when a
  quest-tooling or PDA task needs them.
