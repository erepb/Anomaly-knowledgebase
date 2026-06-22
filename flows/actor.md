---
domain: flows/actor
tags: [engine, cpp, actor, flow, lifecycle, input, camera, condition, damage, callbacks]
sources: [src/xrGame/Actor.h, src/xrGame/Actor.cpp, src/xrGame/Actor_Network.cpp, src/xrGame/Actor_Movement.cpp, src/xrGame/ActorInput.cpp, src/xrGame/ActorCameras.cpp, src/xrGame/ActorCondition.cpp, src/xrGame/ActorCondition.h]
related: [flows/object-lifecycle.md, flows/actor-condition.md, flows/weapon-fire-callbacks.md, flows/inventory.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: Actor (player) lifecycle

What the **player character** (`CActor`) adds on top of the generic object-lifecycle backbone:
spawn, the two per-frame channels (input→movement→physics on the deferred channel; HUD/camera on
the real-time channel), condition (health/stamina), damage, and death. Read
`flows/object-lifecycle.md` first — this only documents the actor-specific overrides; it does not
re-explain `net_Spawn`/`UpdateCL`/`shedule_Update`/`net_Destroy` generically.

**Provenance:** researched with a read-only agent, then spot-verified against code. The
inheritance chain, every callback site (engine `callback(GameObject::e*)` + the `_G.CActor_*` Lua
functors), and the function homes are personally code-verified. Deep function bodies (the full
`net_Spawn`/`shedule_Update`/`UpdateCondition`/`Die` internals, the exact config-key lists, camera
math) are *(agent-traced — not personally re-read)* and marked where it matters.

## Inheritance (verified)

`class CActor : public CEntityAlive, public IInputReceiver, public Feel::Touch, public
CInventoryOwner, public CPhraseDialogManager, public CStepManager, public Feel::Sound` (`Actor.h`).
So the actor is simultaneously: a living entity with condition/health
(`CEntityAlive` → `CEntity` → `CGameObject`, the backbone), an **input receiver** (`IInputReceiver`
— keyboard/mouse), a **touch/sound feeler** (`Feel::Touch`/`Feel::Sound` — zone/danger awareness),
an **inventory owner** (`CInventoryOwner` — see `flows/inventory.md`), a dialog participant, and a
footstep emitter (`CStepManager`). The condition object is a separate `CActorCondition`
(`ActorCondition.h`) reached via `conditions()`.

## Spawn (override)

**`CActor::net_Spawn`** (`Actor_Network.cpp`) — actor-side spawn. Reads the
`CSE_ALifeCreatureActor` server entity (health, position, yaw/pitch/roll, money), chains
`CInventoryOwner::net_Spawn` and `inherited::net_Spawn` (= `CEntityAlive`, which creates the
condition object), sets `g_actor` if local+player, allocates the camera manager, applies the default
outfit (`OnChangeVisual`), enables network processing, registers with the scheduler. *(agent-traced
body; the spawn-from-server-entity backbone is the verified `flows/object-lifecycle.md`.)*

## Per-frame — two channels (actor specifics)

The backbone's two channels (real-time `UpdateCL` vs deferred `shedule_Update`) carry a sharp split
of responsibilities for the actor:

- **`CActor::shedule_Update(u32 DT)`** (`Actor.cpp`) — **game logic / control.** Branches on
  whether this actor is the locally-controlled entity (`Level().CurrentControlEntity() == this`):
  - **Local path:** `g_cl_CheckControls` (read keyboard → build movement acceleration + jump) →
    `g_cl_Orientate`/`g_Orientate` (camera→torso→model rotation) → `g_Physics` (run character
    physics) → `g_cl_ValidateMState` (constrain state: ladder/landing) → `g_SetAnimation`. Plus
    `feel_touch_update`, look-at raycast, breathing/blood/danger sound loops.
  - **Remote path:** `make_Interpolation` from network states instead of input.
  - **Both:** `inherited::shedule_Update` → `conditions().UpdateCondition()` (the once-per-tick
    condition update), artifact/outfit bonuses, physics post-update.
  *(agent-traced body; the local-vs-remote branch + the `g_cl_*`/`g_Physics` call order is the
  load-bearing structure.)*
- **`CActor::UpdateCL()`** (`Actor.cpp`) — **presentation / no input.** HUD flags (`psHUD_Flags`),
  camera update (`cam_Update`), first-person legs (`m_legs_controller`), footsteps
  (`CStepManager::update`), weapon HUD render, deferred game-news messages, and shader-facing
  actor data (health/stamina/bleeding into `g_pGamePersistent`). *(agent-traced body.)*

Movement is therefore driven on the **deferred** channel, not the real-time one — the opposite
intuition trap of a typical "update everything in UpdateCL" engine.

## Input (verified homes)

`CActor` implements `IInputReceiver`: `IR_OnKeyboardPress`/`Release`/`Hold` and `IR_OnMouseMove`
(`ActorInput.cpp`). Press/release/hold route a command to inventory actions (weapon fire, item use),
the movement state machine (jump/crouch/sprint bits in `mstate_wishful`), or camera control; mouse
delta drives the active camera's `Move`. Weapon fire is gated through the Lua functor
**`_G.CActor_Fire`** (verified site in `ActorInput.cpp`) — a script can block firing.

## Movement & physics (verified homes)

`g_cl_CheckControls` (`Actor_Movement.cpp`) reads the movement bit flags + `actor.ltx` speed
coefficients, builds a normalized acceleration vector scaled by walk/run/sprint/crouch factors, and
fires the Lua jump/land functors **`_G.CActor_on_jump`** / **`_G.CActor_on_land`** (verified sites).
`g_Physics` runs the character controller (`character_physics_support()->movement()->Calculate`) and,
on a hard ground collision, generates a `GE_HIT` event against the actor. *(agent-traced bodies;
callback sites verified.)*

## Camera (verified home)

`CActor::cam_Update` (`ActorCameras.cpp`) updates one of four cameras (FirstEye / LookAt / FreeLook /
FixedLookAt), applies bobbing, zoom inertia, weapon recoil, and the first-person death cam. Detailed
camera math is *(agent-traced — SKIMMED)*; the four-camera array and effector list are the structure.

## Condition / health (verified home)

`CActorCondition::UpdateCondition` (`ActorCondition.cpp`) runs once per `shedule_Update` (via
`CEntityAlive::shedule_Update`). It drains/regenerates health, power (stamina), radiation, satiety,
alcohol, applies wounds and boosters, aggregates zone influence from the `feel_touch` list, sets the
`m_bCantWalk`/`m_bCantSprint`/`m_bLimping` gates, and spawns the low-health death effector. Config
comes from the actor's condition section (stamina drain rates, limping/cant-walk thresholds, zone max
values). *(agent-traced body; home verified.)*

## Damage (verified callback)

**`CActor::Hit(SHit* pHDS)`** (`Actor.cpp`) — incoming-damage entry. In single-player it first
offers the hit to the Lua functor **`_G.CActor__BeforeHitCallback`** (verified site) which can cancel
it; then applies artifact/outfit reduction (`HitArtefactsOnBelt`), caches the last hitter, sets the
slow-motion stun, chains `inherited::Hit` (→ condition damage/wounds), runs the screen hit-mark, and
fires the per-object **`callback(GameObject::eHit)(lua_obj, damage, dir, attacker, bone)`** (verified
site). This is the same `eHit` the weapon-fire flow ultimately triggers on a victim
(`flows/weapon-fire-callbacks.md`).

## Death

`CActor::Die` (`Actor.cpp`) — moves the active weapon to the ruck, clears the belt, stops the looping
sounds, switches the camera to FreeLook (or FirstEye in first-person-death mode), zeroes the movement
bits, plays the death cycle, and triggers the game-over tutorial; chains `CEntityAlive::Die`.
*(agent-traced body.)*

## Isolation ledger (the seams)

- **Callbacks fired (per-object `callback(GameObject::e*)`, all sites verified in `Actor.cpp`):**
  `eHit`, `eEnterLevelBorder`/`eExitLevelBorder`, `eOnWeaponRaised`/`eOnWeaponLowered`,
  `eAttachVehicle`/`eDetachVehicle`. Plus **Lua functors** (sites verified): `_G.CActor_Fire`
  (`ActorInput.cpp`), `_G.CActor_on_jump`/`_G.CActor_on_land` (`Actor_Movement.cpp`),
  `_G.CActor__BeforeHitCallback` (`Actor.cpp`). The generic `_G.CGameObject_NetSpawn`/`NetDestroy`
  fire from the backbone. Discord/UI status uses many read-only `_g.Is*`/`ranks.*` functor *queries*
  (not callbacks). Bridge of per-object → global bus is `callbacks_gameobject.script`
  (`flows/lua-callback-lifecycle.md`).
- **Config read:** `actor.ltx` movement coefficients (walk/run/sprint/crouch/strafe), dispersion,
  pickup radii, default outfit; the condition section (stamina drains, limping/cant-walk/zone
  thresholds); per-camera `.ltx` files; global `psActorFlags`/`psHUD_Flags`/mouse sens. *(key names
  agent-traced; mechanism = `pSettings` reads.)*
- **Scheduler seam:** registered at `net_Spawn`, unregistered at `net_Destroy`; movement/control on
  the **deferred** channel, HUD/camera on the **real-time** channel (serial scheduler —
  `engine/engine-core.md`). `AlwaysTheCrow()` returns TRUE → the actor always gets a real-time tick.
- **ALife / server seam:** server entity is `CSE_ALifeCreatureActor`; `net_Export`/`net_Import`
  carry health, position, rotation, movement state, active slot, and physics (or dead-body bones).
  In MP a non-controlled actor takes the remote interpolation path.
- **Lua seam:** `actor:conditions()` → `CActorCondition`; inventory via `CInventoryOwner`
  (`flows/inventory.md`); actor wrapper methods in `actor_script.cpp`.
- **Other xrGame domains:** inventory (active item, weight, belt/ruck — `flows/inventory.md`),
  weapons (active weapon zoom/recoil/fire — `flows/weapon-fire-callbacks.md`), condition, camera,
  character physics, `Feel::Touch`/`Sound` perception, in-game HUD/UI.

## Threading

`UpdateCL` and `shedule_Update` both run on the main thread (serial scheduler). The actor offloads
nothing itself; bullets it fires may offload via the bullet manager (its own flow).

## Failure & edge cases

- **Control entity check is the branch point:** logic that only runs for the locally-controlled
  actor (input, control) is gated on `Level().CurrentControlEntity() == this`. A remote/AI-driven or
  demo-playback actor takes the interpolation path — don't assume input is read every tick.
- **Movement lives on the deferred channel**, not `UpdateCL`; looking for control logic in `UpdateCL`
  misleads.
- **Hit is cancelable from Lua** (single-player `_G.CActor__BeforeHitCallback`) before any damage
  applies — a mod returning false makes the actor invulnerable to that hit.

## Deeper

- Class + spawn/destroy: `src/xrGame/Actor.h`, `src/xrGame/Actor_Network.cpp` (`net_Spawn`,
  `net_Destroy`, `net_Export`/`net_Import`)
- Per-frame: `src/xrGame/Actor.cpp` (`UpdateCL`, `shedule_Update`, `Hit`, `Die`; callback sites grep
  `callback(GameObject::` )
- Input: `src/xrGame/ActorInput.cpp` (`IR_On*`, `_G.CActor_Fire`)
- Movement/physics: `src/xrGame/Actor_Movement.cpp` (`g_cl_CheckControls`, `g_Physics`,
  `_G.CActor_on_jump`/`on_land`)
- Camera: `src/xrGame/ActorCameras.cpp` (`cam_Update`)
- Condition: `src/xrGame/ActorCondition.cpp/.h` (`CActorCondition::UpdateCondition`)

## Maintenance

- Re-grep `callback(GameObject::` in `Actor*.cpp` and the `_G.CActor_*` functor strings to confirm
  the callback set if behavior changes.
- Promote the *(agent-traced)* bodies (`net_Spawn`, `shedule_Update` local/remote branch,
  `UpdateCondition`, `Die`, exact config keys) to verified by reading them when a task needs the
  detail.
- Keep the "movement on the deferred channel" note — it's the non-obvious split that the backbone's
  two-channel model makes possible.
