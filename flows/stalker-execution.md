---
domain: flows/stalker-execution
tags: [xrgame, ai, stalker, movement, sight, animation, fire, flow]
sources: [src/xrGame/ai/stalker/ai_stalker.h, src/xrGame/ai/stalker/ai_stalker.cpp, src/xrGame/ai/stalker/ai_stalker_fire.cpp, src/xrGame/sight_manager.cpp, src/xrGame/stalker_animation_manager.cpp, src/xrGame/movement_manager.cpp, src/xrGame/detail_path_manager.cpp]
related: [flows/ai-planner.md, flows/ai-perception.md, flows/smart-cover.md, flows/weapon-fire-callbacks.md, flows/animation-callbacks.md, flows/monster-ai-control.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: stalker execution layer

How a GOAP-chosen action becomes real **motion, aim, and gunfire** for a stalker NPC. The planner
(decide *what*, `flows/ai-planner.md`) sits above; this is the layer below — the execution managers
(`sight`, `movement`, `animation`) and the fire control that turn an action's intent into a moving,
aiming, shooting NPC. Contrast `flows/monster-ai-control.md` (monsters use FSM+controls instead).

**Provenance:** spine personally code-verified — `CAI_Stalker` owns `CStalkerAnimationManager*
m_animation_manager`, `stalker_movement_manager_smart_cover* m_movement_manager` (the movement manager
*is* the smart-cover variant — ties to `flows/smart-cover.md`), and `animation()`/`sight()`/`movement()`
accessors. The per-frame update order, the `movement`/`sight`/`animation` bodies, the fire path
(aim-then-shoot, skill dispersion, burst), and "NPC weapon is a world object not HUD" are confirmed
homes but *(agent-traced — not personally re-read)*.

## The execution managers (ownership verified)

`CAI_Stalker` owns four pieces (verified): the **brain** (`CStalkerPlanner` — GOAP,
`flows/ai-planner.md`), **`movement()`** (`stalker_movement_manager_smart_cover` ← `CMovementManager`),
**`sight()`** (`CSightManager`), and **`animation()`** (`CStalkerAnimationManager`). A GOAP action's
`execute()` doesn't move the NPC directly — it **sets targets on these managers** *(agent-traced)*:
- `movement().set_level_dest_vertex(v)` / `set_path_type` / `set_movement_type(walk/run/stand)` /
  `set_body_state(stand/crouch)` / `set_mental_state(danger/free/panic)` — where + how to go;
- `sight().setup(CSightAction(enemy…))` — what to look/aim at;
- `set_goal(eObjectActionFire1, weapon, queue)` (via `CObjectHandler`) — fire.
Each frame `shedule_Update`/`Think` updates the brain then movement; `UpdateCL` updates sight (aim);
animation is selected from the resulting state.

## Movement (agent-traced)

`CMovementManager` runs a path state machine: pick game vertex → build game-graph path → build level
path (A*) → build **detail path** (`CDetailPathManager`, smooth Dubins-style curves) → follow it
(velocity/turn into the physics character). `EMovementType` (walk/run/stand) × `EBodyState`
(stand/crouch) × `EMentalState` (danger/free/panic) are the state inputs both for speed and for which
animations the animation manager picks. The stalker variant is the **smart-cover** movement manager,
so cover occupation is integrated here (`flows/smart-cover.md`).

## Sight & fire (agent-traced; aim-gates-fire is the key rule)

`CSightManager` turns the head/spine toward a sight target (`ESightType`: enemy/point/direction/cover/
fire-object), slerping within angle limits. **Fire is gated on aim:** the combat action checks the
head's yaw vs the angle to the enemy; if not aligned it calls `aim_ready()` (keep turning) and only
when aligned issues `eObjectActionFire1`. So an NPC must *face* its target before it shoots.

`ai_stalker_fire.cpp`: `g_fireParams` returns the fire position/direction from the eye matrix (or head
orientation while moving); `GetWeaponAccuracy` builds the dispersion cone scaled by **rank**
(`m_fRankDisperison`) and state (stand/crouch × walk/run/stand × zoom), with a Lua override
`_g.CAI_Stalker__GetWeaponAccuracy`. Burst length/interval (`select_queue_params`) varies by weapon
class and distance. **NPC weapons are world objects on a skeleton bone** (not the actor's HUD item) —
`g_fireParams` reads the bone; the weapon's own `FireStart`/bullet pipeline
(`flows/weapon-fire-callbacks.md`) does the rest. `fire_make_sense` raycasts LOS before bothering.

## Animation (agent-traced)

`CStalkerAnimationManager` selects global/head/torso/legs animation **parts** from {movement type, body
state, mental state, equipped weapon}. The torso animation reflects the weapon (aim idle, fire), legs
the locomotion; the head follows sight. It plays via `IKinematicsAnimated::LL_PlayCycle` and rides the
animation-callback timeline (`flows/animation-callbacks.md`).

## GOAP action → execution (sequence)

1. brain picks an action (`flows/ai-planner.md`); the action's `initialize`/`execute` **set targets**:
   movement dest + type + mental state, sight target, and (combat) a fire goal.
2. `movement().update` builds/follows the path (game→level→detail) into the physics character.
3. `sight().update` aims the head toward the target.
4. fire control: if aimed, `eObjectActionFire1` → the weapon fires (skill-scaled dispersion, burst);
   else keep aiming.
5. `animation().update` picks torso/legs/head animation from the current movement/mental/weapon state.

## Isolation ledger (the seams)

- **Planner (above):** the brain sets the action; under script control the planner yields
  (`flows/ai-planner.md`, `flows/script-npc-actions.md`).
- **Perception:** enemy/cover targets come from memory/enemy managers (`flows/ai-perception.md`).
- **Weapons:** NPC fires its **world** weapon via `CObjectHandler`→`FireStart`→bullet pipeline
  (`flows/weapon-fire-callbacks.md`); `g_fireParams` supplies position/dispersion.
- **Animation/skeleton:** `CStalkerAnimationManager` → `LL_PlayCycle` (`flows/animation-callbacks.md`).
- **Pathfinding/physics:** game/level/detail path managers → physics character movement.
- **Smart cover:** the movement manager is the smart-cover variant (`flows/smart-cover.md`).
- **Lua:** `_g.CAI_Stalker__GetWeaponAccuracy` (accuracy override), `_G.CNPCBeforeLookAtActor`,
  `_g.CAI_Stalker__BeforeHitCallback`, `_g.update_best_weapon`.
- **Config:** per-NPC dispersion by state (`disp_*`), `fire_queue_section` (burst sizes/intervals by
  weapon class + distance), rank dispersion.

## Threading

Main thread: `shedule_Update`/`Think`/movement on the scheduled channel, `sight`/`UpdateCL` on the
render channel; bone calc on the worker thread (`flows/animation-callbacks.md`).

## Failure & edge cases

- **Must aim before firing:** an NPC that "won't shoot" is often still turning to face the enemy
  (sight angle > fire threshold) — fire is gated on aim.
- **NPC weapon ≠ HUD:** NPC guns are world objects on bones; the actor's HUD weapon path doesn't apply.
  Fire point/dispersion come from `g_fireParams`, not a HUD model.
- **Accuracy is skill+state:** rank and stand/crouch/move/zoom scale the dispersion cone — a "too
  accurate/inaccurate" NPC is the `disp_*`/rank config, overridable in Lua.
- **Detail path smoothing:** NPCs follow a smoothed detail path, not the raw level path — odd pathing is
  usually the detail-path/graph, not the action.
- **Movement manager is smart-cover-aware:** cover behavior is part of stalker movement, not a bolt-on
  (`flows/smart-cover.md`).

## Deeper

- Owner + update: `src/xrGame/ai/stalker/ai_stalker.cpp`/`.h` (`UpdateCL`, `shedule_Update`, `Think`,
  `SelectAnimation`, `animation()`/`sight()`/`movement()`)
- Movement: `src/xrGame/movement_manager.cpp` (path state machine), `detail_path_manager.cpp` (smooth
  follow)
- Sight: `src/xrGame/sight_manager.cpp` (`CSightManager`, `ESightType`)
- Animation: `src/xrGame/stalker_animation_manager.cpp` (part selection)
- Fire: `src/xrGame/ai/stalker/ai_stalker_fire.cpp` (`g_fireParams`, `GetWeaponAccuracy`,
  `select_queue_params`, `fire_make_sense`) — weapon side `flows/weapon-fire-callbacks.md`
- Planner above: `flows/ai-planner.md`; cover: `flows/smart-cover.md`; monster contrast:
  `flows/monster-ai-control.md`

## Maintenance

- Keep the "GOAP sets targets on movement/sight/animation; fire is gated on aim; NPC weapon is a world
  object" facts — they are the core of NPC combat execution.
- Re-grep `CAI_Stalker`'s manager ownership / `ai_stalker_fire.cpp` if the execution wiring or accuracy
  model changes.
- Promote the *(agent-traced)* manager bodies + fire path to verified when an NPC-combat-tuning task
  needs the exact dispersion/burst math.
