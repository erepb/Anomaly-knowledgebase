---
domain: modlayer/gameplay/tasks-lua
tags: [modlayer, gameplay, unpacked, lua, task, quest, map]
sources: [unpacked/scripts/task_manager.script, unpacked/scripts/task_objects.script, unpacked/scripts/axr_task_manager.script, unpacked/configs/misc/task/task_manager.ltx]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/task-system-lua.md, flows/tasks.md, flows/condlist-library.md, flows/dialog-trade.md, flows/pda-map-news.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Task system (Lua) — L1 map

The quest tier above the engine `CGameTask` shell (F38): config-driven task objects with live
functor-recomputed text/targets, condlist outcomes, repeatable cooldowns, and the dynamic/bounty
extenders. Core mechanics live in `flows/task-system-lua.md` (F97); this map routes the family.

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| `CRandomTask` | `task_manager.script` | singleton manager: give/tick/force/persist + engine functors | F97 ✅ |
| `CGeneralTask` | `task_objects.script` | per-task object: functor refresh, condlist outcomes, map spots, rewards | F97 ✅ |
| functor catalogs | `task_functor.script`, `task_status_functor.script` | title/descr/target + status evaluators (generic `condlist`/`target_condlist` + ~50 story ones) | F97 ✅ |
| config | `configs/misc/task/task_manager.ltx` + `tm_*.ltx` (wildcard include) | `[task_id]` sections: functors, condlist_N, on_* effects, storyline/prior/repeat_timeout | F97 ✅ |
| dynamic task types | `tasks_*.script` (bounty/assault/defense/delivery/fetch/clear_map/guide/…) | quest-type plugins: namespace injection + sim callbacks | F98 ✅ → `flows/dynamic-task-types.md` |
| AXR extender | `axr_task_manager.script` | bounties, available/ongoing/finished registries, faction-questline honchos | F114 ⬜ |

## How it fits

Dialog/logic effect (`xr_effects.give_task`, F18/F67) → `CRandomTask:give_task` → engine `CGameTask`
with Lua complete/fail functors (F38) → `bind_stalker:update` ticks `check_task` (live functors +
condlists) → engine polls verdict → `task_state` callback → `task_callback` (effects, rewards via
dialog speaker, news F39, repeat cooldown). Persistence via `alife_storage_manager` m_data (marshal)
from the actor binder's save/load (F15/F88).

## Subsystem deep-dives (delegates to flows)

- Core lifecycle/config/functors/persistence + known traps (broken `reward_item` line, dialog-only
  rewards, CoP-only guider spots) → `flows/task-system-lua.md` (F97) ✅
- Dynamic types (plugin pattern, stage stepping, sim plugins) → `flows/dynamic-task-types.md` (F98) ✅
- Engine half (CGameTask monolithic poll) → `flows/tasks.md` (F38)
- AXR bounties/questlines → F114 ⬜ (`ROADMAP-gameplay.md`)

## Maintenance

- Promote the F114 row when it lands. Re-check `gamedata/scripts/{task_*,tasks_*,axr_task_manager}`
  for overrides on pickup (none at writing).
