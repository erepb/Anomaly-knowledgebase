---
domain: flows/script-npc-actions
tags: [xrgame, ai, script-control, action-queue, xr-logic, flow]
sources: [src/xrGame/script_entity.cpp, src/xrGame/script_entity.h, src/xrGame/script_entity_action.h, src/xrGame/script_movement_action.h, src/xrGame/script_watch_action.h, src/xrGame/ai/stalker/ai_stalker_script_entity.cpp, src/xrGame/script_action_planner_wrapper.h, src/xrGame/CustomMonster.cpp]
related: [flows/ai-squad-logic.md, flows/ai-planner.md, flows/stalker-execution.md, flows/monster-ai-control.md, flows/cscript-game-object.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: script-driven NPC actions (C++ side of xr_logic)

The C++ machinery that lets Lua **take direct control** of an NPC and drive its movement/sight/
animation/sound — the engine side of the `xr_logic` schemes documented in `flows/ai-squad-logic.md`.
When a scheme puts an NPC under "script control," the GOAP planner (`flows/ai-planner.md`) yields and a
**queue of script actions** drives the same execution managers as `flows/stalker-execution.md`.

**Provenance:** spine personally code-verified — `CScriptEntity` has `m_bScriptControl`,
`GetScriptControl()`, and `ProcessScripts()` (the script-control fork, consistent with the
`GetScriptControl→ProcessScripts` seam verified in `flows/ai-planner.md`). The `CScriptEntityAction`
sub-action structure, the `bfAssign*` execution bodies, `AddAction`, and the
"script control is a control-flow fork, not a GOAP world property" finding are confirmed homes but
*(agent-traced — not personally re-read)*.

## Script control = a fork, not a planner action (verified mechanism)

`CScriptEntity` (mixed into NPCs) holds `m_bScriptControl` + `GetScriptControl()`/`SetScriptControl
(bool, name)`/`ProcessScripts()` (verified). The per-frame scheduler forks on it: **`GetScriptControl()
? ProcessScripts() : Think()`** — i.e. under script control the **GOAP planner is skipped entirely** and
the script action queue runs instead; releasing control (`SetScriptControl(false)` or the queue
empties) returns to the planner next frame. Notably, script control is **not** modeled as a GOAP world
property — it is a straight control-flow branch in the update *(agent-traced; the fork's existence is
verified via the `ProcessScripts`/`GetScriptControl` members + the `ai-planner` seam)*. The
`script_action_planner_wrapper` lets Lua optionally override a whole planner, but the common path is
this queue.

## Script action queue (agent-traced)

Under control, the NPC runs a deque of **`CScriptEntityAction`** (`m_tpActionQueue`). Each action
bundles independent **sub-actions**, each with its own completion flag:
- **movement** (`CScriptMovementAction`: dest object/position/patrol/path-node, body state, movement
  type, detail-path type, speed),
- **watch** (`CScriptWatchAction`: look at object/direction/sight-type, bone),
- **animation** (`CScriptAnimationAction`: animation name + mental state),
- **sound** (`CScriptSoundAction`), **particle**, **object** (`CScriptObjectAction`: weapon fire/reload/
  aim via `CObjectHandler`), and a **condition** (which sub-actions + lifetime define "done").

Lua builds an action and enqueues it with **`AddAction`** (Lua `obj:command(action)`); a high-priority
add jumps the queue; an empty queue runs immediately.

## Processing — drive the execution managers (agent-traced; managers verified elsewhere)

`ProcessScripts()` each frame: take the front action, `initialize` it once, check completion (remove +
fire callback when done), and for the current action dispatch its sub-parts to the **same execution
managers as normal AI** (`flows/stalker-execution.md`):
- `bfAssignMovement` → `movement().set_level_dest_vertex`/`set_path_type`/`patrol().set_path`/… (marks
  done when the path completes);
- `bfAssignWatch` (stalker override) → `sight().setup(...)` (done when the head aligns);
- `bfAssignAnimation`/`bfScriptAnimation` → `LL_PlayCycle` (done on blend-end callback);
- `bfAssignSound`/`bfAssignParticles` → play + bone-track; `bfAssignObject` → `CObjectHandler::set_goal`
  (fire/reload/aim, done on `goal_reached`).
When all required sub-actions of the front action complete, it pops and the next runs; an empty queue
(or `SetScriptControl(false)`) hands control back to the planner.

So script control reuses the *execution* layer (movement/sight/animation/fire) but replaces the
*decision* layer (planner) with a Lua-fed action queue. Both stalkers and monsters can be script-driven
(`CCustomMonster`/`CAI_Stalker` both fork on `GetScriptControl`).

## Lua takes control → queue → execute → return (sequence)

1. an `xr_logic` scheme (`flows/ai-squad-logic.md`) calls `obj:script(true, name)` → `SetScriptControl`.
2. Lua builds a `CScriptEntityAction` (movement+watch+animation+…) and `obj:command(action)` →
   `AddAction`.
3. each frame the scheduler runs `ProcessScripts` (planner skipped): the front action's sub-parts set
   targets on movement/sight/animation/object; completion flags advance the queue.
4. when the queue empties or `obj:script(false)` is called, control returns to the GOAP planner.

## Isolation ledger (the seams)

- **Lua / xr_logic:** `obj:script(true,name)`/`get_script`/`command(action)`/`action()`; plus direct
  manager calls (`set_dest_level_vertex_id`, `set_body_state`, `set_movement_type`, `set_path_type`,
  `add_animation`) that bypass the queue. The scheme layer is `flows/ai-squad-logic.md`.
- **Planner:** mutually exclusive with `ProcessScripts` — `GetScriptControl()?ProcessScripts():Think()`
  (`flows/ai-planner.md`); script control is a fork, not a world property.
- **Execution managers:** the same `movement`/`sight`/`animation`/`CObjectHandler` as
  `flows/stalker-execution.md`; animation via `LL_PlayCycle` (`flows/animation-callbacks.md`).
- **CScriptGameObject:** the `command`/`script`/`set_dest_*` methods are on the `game_object` wrapper
  (`flows/cscript-game-object.md`).
- **Config:** none directly — behavior is Lua-authored (xr_logic action scripts); the C++ side is the
  queue + assign machinery.

## Threading

Main thread (scheduled NPC update + render-frame sound/particle position callbacks). Lua is
single-threaded.

## Failure & edge cases

- **Planner is fully off under script control:** an NPC under `obj:script(true)` ignores GOAP entirely
  until released — a "stuck" scripted NPC is usually a sub-action whose completion flag never sets
  (e.g. an unreachable movement dest, or a missing animation that never ends).
- **Completion is per-sub-action:** an action is done only when all its *required* sub-actions
  (per the condition flags) complete; a watch that never aligns or a sound that loops can hold the queue.
- **Direct vs queued:** Lua can drive movement/animation directly (outside the queue) or via a queued
  action; only queued actions participate in completion/sequencing.
- **Release returns to planner:** forgetting `obj:script(false)` (or never emptying the queue) leaves
  the NPC permanently scripted; the planner resumes only when control is released.
- **Both NPC kinds:** stalkers and monsters share this fork (`flows/monster-ai-control.md`), but the
  stalker `bfAssign*` overrides add sight/object handling.

## Deeper

- Control + queue: `src/xrGame/script_entity.cpp`/`.h` (`SetScriptControl`, `GetScriptControl`,
  `ProcessScripts`, `AddAction`, `bfAssignMovement`/`Watch`/`Animation`/`Sound`/`Object`,
  `m_tpActionQueue`), `CustomMonster.cpp` (`shedule_Update` fork)
- Action structs: `src/xrGame/script_entity_action.h` (`CScriptEntityAction` + sub-actions),
  `script_movement_action.h`, `script_watch_action.h`, `script_animation_action.h`,
  `script_sound_action.h`
- Stalker overrides: `src/xrGame/ai/stalker/ai_stalker_script_entity.cpp` (`bfAssignWatch`/
  `bfAssignObject`)
- Planner wrapper: `src/xrGame/script_action_planner_wrapper.h`
- Lua scheme side: `flows/ai-squad-logic.md`; execution: `flows/stalker-execution.md`; wrapper API:
  `flows/cscript-game-object.md`

## Maintenance

- Keep the "script control is a control-flow fork (`GetScriptControl?ProcessScripts:Think`), planner
  fully yields, reuses the execution managers" mechanism accurate — it is the core of the C++↔xr_logic
  seam.
- Re-grep `CScriptEntity::ProcessScripts` / the `shedule_Update` fork if script control changes.
- Promote the *(agent-traced)* `CScriptEntityAction`/`bfAssign*` bodies to verified when a scheme-
  authoring or NPC-control task needs the exact sub-action/completion logic.
