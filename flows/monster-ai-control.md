---
domain: flows/monster-ai-control
tags: [xrgame, ai, monster, control, fsm, state, animation, flow]
sources: [src/xrGame/ai/monsters/control_manager.cpp, src/xrGame/ai/monsters/control_manager.h, src/xrGame/ai/monsters/control_combase.h, src/xrGame/ai/monsters/control_animation.cpp, src/xrGame/ai/monsters/control_movement.cpp, src/xrGame/ai/monsters/control_jump.cpp, src/xrGame/ai/monsters/basemonster/base_monster.h, src/xrGame/ai/monsters/basemonster/base_monster_think.cpp, src/xrGame/ai/monsters/monster_enemy_manager.cpp]
related: [flows/ai-planner.md, flows/ai-perception.md, flows/animation-callbacks.md, flows/ai-squad-logic.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: monster AI (control architecture)

How monsters (dogs, bloodsuckers, burers, …) think and act — a **completely different architecture
from the stalker GOAP planner** (`flows/ai-planner.md`). A monster runs a per-species **finite state
machine** that activates **pluggable control elements** (animation, movement, direction, path, plus
ability controls like jump/charge); the controls own the low-level actuation. Perception feeds it via
a dedicated monster enemy manager.

**Provenance:** spine personally code-verified — `CControl_Manager` (owns `m_object` `CBaseMonster*` +
`m_control_elems[type]`; `capture`/`capture_pure`/`update_frame` exist), and `CBaseMonster` carrying
**`IStateManagerBase* StateMan`** + `CControl_Manager* m_control_manager` (`control()`) — i.e. **FSM +
control manager, not a GOAP planner**. The control-element base classes, the specific pure/ability
control bodies (`control_animation`/`control_jump`/`control_run_attack`/…), the `Think`/`update_fsm`
loop, and `CMonsterEnemyManager` are confirmed homes but *(agent-traced — not personally re-read)*.

## The control manager (verified core)

`CControl_Manager` (`control_manager.h`) is the heart: it owns a map of **control elements** keyed by
`ControlCom::EControlType` (`m_control_elems[type]`) plus the active set. The model:
- **`capture(com, type)`** / **`capture_pure(com)`** (verified) — an ability **exclusively grabs** one
  or more pure controls (path/animation/movement/direction), calling `on_release` on the prior owner
  and `on_capture`/`on_start_control` on the new one; **`release`** hands control back to the base.
- **`update_frame()`** / `update_schedule()` (verified) — each frame/schedule tick, dispatch
  `update_frame`/`update_schedule` to every active control (bails if the monster is dead).
- controls communicate by **events** (`notify`/listeners): `eventAnimationStart`/`End`,
  `eventPathBuilt`, `eventJumpEnd`, `eventRunAttackEnd`, … — an ability subscribes to the animation-end
  event to know its move finished.

So the manager is a small framework: pure controls are the actuators, ability controls capture them to
perform a move, and events drive completion. *(element/ability bodies agent-traced.)*

## Control elements (agent-traced)

- **Pure controls** (always active once inited): **`control_animation`** plays the current motion on
  global/legs/torso tracks via `IKinematicsAnimated::LL_PlayCycle` and fires the animation events
  (`flows/animation-callbacks.md`); **`control_movement`** smooths velocity toward a target (into the
  physics movement); **`control_direction`** smooths heading/pitch; **`control_path_builder`** (also a
  `CMovementManager`) does level+game-graph pathfinding and detail-path following, feeding movement.
- **Ability controls** (custom, capturable + controlling — `CControl_ComCustom`): **`control_jump`**
  (capture path+anim+move+dir, drive a prepare→glide→ground sequence, apply the leap impulse, fire
  `eventJumpEnd`), **`control_run_attack`** (charge: face enemy, run-attack animation, build a path
  line, `eventRunAttackEnd`), **`control_melee_jump`**, **`control_threaten`**. They `check_start_
  conditions` (distance/facing/cooldown), `activate` → capture pures, and `on_release` restore them.
  A `CControlManagerCustom` aggregates these abilities and dispatches their completion events.

## Think loop — FSM not GOAP (structure verified)

`CBaseMonster` ties it together with a **state manager** (`StateMan`, an `IStateManagerBase*` — the
per-species FSM) and the control manager (`control()`). Each tick *(loop body agent-traced)*:
- **`shedule_Update`** → `control().update_schedule()` (pathfinding/scheduled control work) + auras/
  morale/physics.
- **`Think`** → `UpdateMemory` (enemy/sound/hit memory, `flows/ai-perception.md`) → squad update →
  **`update_fsm`** → `StateMan->update()` — the FSM selects/executes a state
  (`eStateAttack`/`eStateRest`/…); a state **activates ability controls** (e.g. `com_man().jump(enemy)`
  when in range), which capture the pure controls.
- **`UpdateCL`** → `control().update_frame()` (animation playback + callbacks).

**The contrast with stalkers** (verified by structure): a stalker plans with **GOAP** (goal → A* over
world properties → action sequence, `flows/ai-planner.md`); a monster runs a hand-authored **FSM** per
species and expresses actions as **captured controls**. The controls (jump/run-attack/animation) are
generic and shared across species; the FSM states are per-monster.

## Enemy selection (agent-traced)

`CMonsterEnemyManager` is a **separate** subsystem (not the planner, not the shared `CEnemyManager`): it
picks/tracks the current enemy from the monster's enemy/sound memory, last-seen position/time, a
danger classification, and visibility flags; a Lua `force_enemy` can override it. The FSM reads it to
decide attack vs flee. *(agent-traced.)*

## Isolation ledger (the seams)

- **Animation/skeleton:** `control_animation` → `IKinematicsAnimated::LL_PlayCycle` + the animation
  events (`flows/animation-callbacks.md`); abilities sequence on `eventAnimationEnd`.
- **Movement/path:** `control_movement` + `control_path_builder` (a `CMovementManager`) drive velocity
  and pathfinding into the physics character.
- **Perception:** `UpdateMemory` + `CMonsterEnemyManager` (`flows/ai-perception.md`).
- **Damage:** hits feed the hit memory; a critical-wound control can react.
- **Lua / script control:** monsters can be put under script control (`SetScriptControl`, the C++ side
  of `xr_logic`, `flows/ai-squad-logic.md` / the planned F37), and scripts can `force_enemy`, trigger
  `jump`, or assign movement/animation/sound actions. The repo's `bind_monster.script` is a live
  override (`flows/ai-squad-logic.md`).
- **Config:** per-monster `.ltx` — ability params (`jump_*`, `Run_Attack_*`, cooldowns), animation sets,
  movement profiles.

## Threading

Main thread: `Think`/`update_fsm`/`update_frame` and `shedule_Update`/`update_schedule` run on the
scheduled object update (`flows/object-lifecycle.md`). Bone calc is the usual worker-thread path
(`flows/animation-callbacks.md`).

## Failure & edge cases

- **Don't look for a planner:** monsters have **no GOAP** — behavior is the FSM (`StateMan`) plus
  control captures. Searching for monster "actions/goals" like a stalker is a dead end.
- **Capture is exclusive:** an ability that captures path+anim+move+dir blocks others until it releases
  (on completion/timeout); a stuck monster is often an ability that never fired its end event.
- **Events drive sequencing:** abilities advance on `eventAnimationEnd`/`eventPathBuilt`/… — a missing/
  zero-length animation can strand an ability (it never gets its end event).
- **`check_start_conditions` gates abilities:** jump/charge won't fire unless distance/facing/cooldown
  pass — a monster that "won't attack" usually fails a start condition.
- **Per-species FSM:** each monster type has its own states; generic control behavior + species FSM is
  the split to keep straight.

## Deeper

- Manager: `src/xrGame/ai/monsters/control_manager.cpp`/`.h` (`capture`, `capture_pure`, `release`,
  `update_frame`/`update_schedule`, `m_control_elems`), `control_combase.h` (`CControl_Com`,
  `CControl_ComPure`, `CControl_ComCustom`, `CControl_ComBase`)
- Pure controls: `control_animation.cpp` (`LL_PlayCycle`, events), `control_movement.cpp`,
  `control_direction.cpp`, `control_path_builder*.cpp`
- Ability controls: `control_jump.cpp`, `control_run_attack.cpp`, `control_melee_jump.cpp`,
  `control_threaten.cpp`, `control_manager_custom.cpp` (ability aggregator)
- Think/FSM: `basemonster/base_monster.h` (`StateMan`, `m_control_manager`, `control()`),
  `basemonster/base_monster_think.cpp` (`Think`, `update_fsm`, `shedule_Update`),
  `monster_state_manager.h`, `state.h`
- Enemy: `monster_enemy_manager.cpp`/`.h`
- Contrast: `flows/ai-planner.md` (stalker GOAP); perception: `flows/ai-perception.md`

## Maintenance

- Keep the **monster = FSM + control-manager, stalker = GOAP** distinction front and center — it is the
  single most important fact for navigating monster AI.
- Re-grep `CControl_Manager::capture`/`update_frame` and `CBaseMonster`'s `StateMan`/`control()` if the
  architecture changes.
- Promote the *(agent-traced)* control-element/ability bodies and the `Think`/`update_fsm` loop to
  verified when a monster-behavior task needs a specific control's exact logic.
