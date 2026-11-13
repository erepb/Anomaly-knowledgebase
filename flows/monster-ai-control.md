---
domain: flows/monster-ai-control
tags: [xrgame, ai, monster, control, fsm, state, animation, flow]
sources: [src/xrGame/ai/monsters/basemonster/base_monster.cpp, src/xrGame/script_monster_enemy_info.h, src/xrGame/script_monster_home_info.h, src/xrGame/ai/monsters/control_manager.cpp, src/xrGame/ai/monsters/control_manager.h, src/xrGame/ai/monsters/control_combase.h, src/xrGame/ai/monsters/control_animation.cpp, src/xrGame/ai/monsters/control_movement.cpp, src/xrGame/ai/monsters/control_jump.cpp, src/xrGame/ai/monsters/basemonster/base_monster.h, src/xrGame/ai/monsters/basemonster/base_monster_think.cpp, src/xrGame/ai/monsters/monster_enemy_manager.cpp, src/xrGame/ai/monsters/pseudogigant/pseudo_gigant.cpp, src/xrGame/ai/monsters/burer/burer_state_attack_tele.h, src/xrGame/ai/monsters/burer/burer_state_attack_tele_inline.h]
related: [flows/ai-planner.md, flows/ai-perception.md, flows/animation-callbacks.md, flows/ai-squad-logic.md, flows/options-runtime.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
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
- **Species-specific behaviour is cvar-gated in this repo.** Two examples to copy when adding one:
  `pseudogiant_can_damage_objects_on_stomp` and `pseudogiant_dodge_stomp_while_falling` (both globals in
  `ai/monsters/pseudogigant/pseudo_gigant.cpp`, registered in `console_commands.cpp`, exposed as
  options rows). The second makes `CPseudoGigant::on_threaten_execute` treat `mcFall` like the
  already-exempt `mcJump`, i.e. a falling actor dodges the earthquake stomp; default **off**, so vanilla
  behaviour is unchanged unless enabled.
- **An ability that holds a pointer to another object must implement `remove_links`.** The burer
  telekinesis crash is the worked example: `CStateBurerAttackTele` kept `selected_object` (the thing it
  is throwing) across updates, so a destroyed target left a dangling pointer. The fix overrides
  `remove_links` to null it and drop `ACTION_TELE_FIRE` back to `ACTION_TELE_CONTINUE`, plus a
  defensive `!selected_object || getDestroy()` early-out in `ExecuteTeleFire`. Any new ability caching
  an object pointer needs the same pair.

## Lua combat seam (merged 2026.10.11 / 2026.11.09, dami.sirbu)

A per-monster Lua surface over the FSM + control manager. Full signatures and caveats are in
`lua_help_ex.script` (search "CBaseMonster -"). Everything is per object, **not serialized** (re-apply on
`net_spawn`), and defaults to vanilla behaviour.

- **Combat decision callback** — `CBaseMonster::script_combat_substate` calls `_G.CMonsterCombatAction`
  (bus callback `monster_on_combat_action`) at most once per monster per frame (cached by
  `m_script_substate_frame`) with `(monster, enemy, dist, enemy_strength, can_jump)`; a returned
  `EMonsterState` id is a *proposal*. Consumed by the shared attack selectors
  (`monster_state_attack_inline.h`, `group_state_attack_inline.h`) and by **every species state
  manager** (`*_state_manager.cpp`); honored only if that species registers the state and its start
  conditions pass. Poltergeist takes none. Enum `monster_state` in `lua_help_ex`.
- **Readers** — `get_monster_state` (deepest FSM substate), rank/morale/aura influences,
  `get_monster_enemy_info` / `get_monster_home_info` (`script_monster_enemy_info.h`,
  `script_monster_home_info.h`), extended `get_monster_hit_info` / `get_sound_info`,
  `get_monster_cover_vertex`.
- **Param setters** — attack distance, jump params/flags, run-attack, attack-on-move, aura falloff,
  turn factor (`CControlDirectionBase::update_frame`), and species families (chimera, burer gravi/tele,
  controller tube/stamina, poltergeist, bloodsucker vis timing).
- **Movement override** — `set_monster_dest` / `clear_monster_dest`, re-asserted in
  `CControlPathBuilderBase::update_target_point`.
- **Maneuvers** — `try_monster_jump` / `_rotation_jump` / `_run_attack` / `_threaten`,
  `monster_capture_control` / `monster_release_control`; all defer to a scheme owner (script control).
  `set_monster_melee_strike` arms the dormant `CControlMeleeJump` rear spin-strike.
- **Fixes in the same series** — chimera attack geometry guarded against bad configs (sqrt clamp below
  jump range); the dead `set_monster_aggressive` flag removed. Dog consults the callback only when not
  under control.

Other monster changes this period: phantoms can die / switch to shoot when hit during `stBirth`
(`ai/phantom/phantom.cpp`); `CALifeMonsterBrain::process_task` guards a nil smart-terrain task.

## Deeper

- Lua seam: `basemonster/base_monster.cpp:CBaseMonster::script_combat_substate`,
  `script_game_object3.cpp` (monster readers/setters), `control_manager_custom.cpp`
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
