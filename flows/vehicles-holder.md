---
domain: flows/vehicles-holder
tags: [xrgame, vehicle, car, helicopter, holder, mount, input, flow]
sources: [src/xrGame/holder_custom.h, src/xrGame/holder_custom.cpp, src/xrGame/Car.cpp, src/xrGame/CarNew.cpp, src/xrGame/Car.h, src/xrGame/CarDrone.cpp, src/xrGame/CarDrone.h, src/xrGame/CarInput.cpp, src/xrGame/CarCameras.cpp, src/xrGame/CarScript.cpp, src/xrGame/Helicopter.cpp, src/xrGame/helicopter.h, src/xrGame/HolderEntityObject.cpp, src/xrGame/Actor.cpp, src/build_config_defines.h, gamedata/scripts/ltx_help_ex.script, gamedata/scripts/lua_help_ex.script]
related: [flows/actor.md, flows/item-base.md, flows/weapon-fire-callbacks.md, flows/cscript-game-object.md, engine/physics-collision.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: vehicles & the holder seam

How the actor mounts a controllable object (car, turret) and how control — input, camera, physics —
is handed off, plus the (script-controlled, *non-mountable*) helicopter. The unifying abstraction is
**`CHolderCustom`**: anything the actor can "get into" implements it, and while mounted the actor
forwards its input/camera to the holder. A car is a physics-driven `CHolderCustom`; a helicopter is
**not** a holder at all — it's an AI combat aircraft.

**Provenance:** spine personally code-verified — `CCar : … CHolderCustom …` (mountable),
`CHelicopter : CEntity, CShootingObject, CRocketLauncher, CPHSkeleton, CPHDestroyable, CHitImmunity,
CExplosive` (**no `CHolderCustom`** — not mountable), `CActor::attach_Vehicle(CHolderCustom*, bForce)`,
and the `eAttachVehicle` callback site. The `CHolderCustom` interface bodies, the `CCar` subcomponents
(wheels/doors/weapon/cameras/engine), the input-forward in `ActorInput`, the helicopter movement/script,
and `HolderEntityObject` are confirmed homes but *(agent-traced — not personally re-read)*.

## Holder seam (interface verified at use; bodies agent-traced)

`CHolderCustom` (`holder_custom.h`) is the thin interface for a mountable object: `attach_Actor` /
`detach_Actor` (store/clear the owner actor), and the pure-virtual input relays `OnKeyboardPress`/
`OnKeyboardRelease`/`OnKeyboardHold` plus a `Camera()`. While the actor holds one, **its input is
forwarded** to the holder. *(agent-traced; the actor side is verified below.)*

## Actor enter → drive → exit (actor seam verified; bodies agent-traced)

- **Enter:** `kUSE` on a `CHolderCustom` → `CActor::use_HolderEx` → **`CActor::attach_Vehicle(holder,
  bForce)`** (verified): checks `EnterLocked()`, calls the holder's accept/`attach_Actor`, sets
  `m_holder`, **blocks the actor's weapons/inventory** (`INV_STATE_BLOCK_ALL`) and destroys the actor's
  character physics, switches to the holder camera, and fires **`callback(GameObject::eAttachVehicle)`**
  (the Lua hook). *(verified entry + callback; body agent-traced.)*
- **Drive:** while `m_holder` is set, `ActorInput::IR_OnKeyboardPress` forwards every key **except
  `kUSE`** to `m_holder->OnKeyboardPress` (the input handoff); `CActor::UpdateCL` ticks the holder and
  uses its camera. *(agent-traced.)*
- **Exit:** `kUSE` again → `use_HolderEx(NULL)` → `detach_Vehicle` → holder `detach_Actor`, recreate
  actor character physics, restore inventory/camera, fire `eDetachVehicle`, clear `m_holder`.
  *(agent-traced.)*

So mounting is a clean three-way handoff — **input, camera, physics all move to the holder** until
dismount; `kUSE` is reserved for exit.

## Car (mountable, physics-driven) — agent-traced

`CCar : … CHolderCustom, CPHSkeleton, CPHUpdateObject …` (verified mountable). It is a real physics
vehicle: `PhDataUpdate` (physics step) drives wheels via the engine/transmission, the components
*(agent-traced)*:
- **wheels** (`CarWheels`: `SWheelDrive`/`SWheelSteer`/`SWheelBreak`) — drive torque / steer angle /
  brake on wheel joints with suspension;
- **doors** (`CarDoors`: `SDoor`) — hinge joints, open/close/break;
- **engine** (`Drive`/`StartEngine`/`Transmission`/`UpdatePower`) — RPM/torque/gears/fuel;
- **mounted weapon** (`CCarWeapon`) — barrel-aim + fire (uses the bullet pipeline,
  `flows/weapon-fire-callbacks.md`);
- **cameras** (`CarCameras`: first/chase/free) swapped on mount.
Input from the mounted actor is routed in `CarInput::OnKeyboardPress` (`kFWD`→forward, `kJUMP`→brake,
`kDETECTOR`→toggle engine, …), with an optional `on_key_board` Lua pre-hook. `HolderEntityObject` is a
generic mountable physics object (turret) on the same seam.

## CAR_NEW: drone, visual camera, camera scopes (repo extension, verified surface)

`CAR_NEW` (`src/build_config_defines.h`) is on, so `CarNew.cpp` — not the legacy `Car.cpp` path, both
whole-file `#ifdef`s — is what runs, and three feature blocks hang off `CCar`. Each is **opt-in by
config**: a block exists only if the visual's `userdata` declares its section, so an unmodified car is
unaffected. Tunables split across two places — plain floats live in the car's **LTX section**, bone
wiring in the visual's **userdata**. Keys are catalogued in `gamedata/scripts/ltx_help_ex.script`, the
Lua surface in `lua_help_ex.script`.

- **Drone subclass** (`CCarDrone`, `CarDrone.cpp/.h`) — built when the userdata declares
  `[drone_definition]`. A drone is not a wheeled car: `PhDataUpdate` applies per-axis forces to the
  physics elements of the bones named by `drive_bones` — `control_ele/pit/rol/yaw_force` for
  up-down / forward-back / strafe / turn (yaw and the rotation damping are read in degrees and
  converted) — with `damping_velocity` and per-axis `damping_rotation` to stop wobble, plus
  `m_power_efficiency` as a payload-style responsiveness scalar (`CCarDrone_Get/SetPowerEfficiency`).
  A userdata `[rotors]` section maps each rotor bone to a `clock:true/false` spin direction, spun by
  `rotor_force` up to `rotor_speed` RPM in `RotorUpdate`; `Load` **`R_ASSERT3`s that at least one rotor
  bone resolved**, so a drone section with no matching bones is a hard error, not a silent no-op.
  Keyboard press/release/hold arrive through `CCar::CCarDrone_OnKeyboardPress/Release/Hold` and set the
  `eControlEle/Yaw/Pit/Rol` intents; `ControlReset` clears them.
- **Visual camera** (`CCar::SVisualCamera`) — `[visual_camera_definition]` with `enable`,
  `rotate_x_bone`/`rotate_y_bone` and their speeds: a camera mounted on model bones that turns with the
  view. Read back from Lua as `VisualCamera_GetDesireAngle` / `_GetRotXCur` / `_GetRotYCur`.
- **Camera scopes** (`m_scopes`, `m_scope_active`) — `[scopes]` maps a level to `bone_camera:zoom`
  (`S0 = bone_camera:1.0`, `S1 = …:2.0`, …), giving the mounted view discrete zoom steps.
  `IsScopeEnable`/`GetScopeSize`/`GetScopeActive`/`SetScopeActive` plus `GetZoomFactor` and
  `Get/SetViewportNear` are exported; `SetUseAction` overrides the use prompt. Input: `CActor`'s
  mouse-wheel handler (`ActorInput.cpp`) checks `m_holder` for a scope-enabled car first and routes the
  notch to `CCar::ScopeOnMouseWheel`, consuming it — so wheel-driven actor behaviour is suppressed while
  mounted in a scoped vehicle.

## Helicopter (NOT a holder — AI aircraft) — agent-traced

`CHelicopter` is **not** a `CHolderCustom` (verified inheritance) — the actor cannot mount it. It is a
scripted-path combat aircraft (`CEntity` + `CShootingObject` + `CRocketLauncher` + `CExplosive`): a
movement state machine follows patrol/round paths or flies to a point, a body-state pitches/banks
toward the target, and it attacks with a rocket/MGun. Scripts drive it via its Lua surface (`SetEnemy`,
`goPatrolByPatrolPath`, `SetDestPosition`, `LookAtPoint`, `Die`/`Explode`) and react via `on_point`
(reached waypoint) / `on_hit` callbacks. *(agent-traced; the "not a holder" inheritance is verified.)*

## Isolation ledger (the seams)

- **Actor:** `attach_Vehicle`/`detach_Vehicle`/`use_HolderEx` move input/camera/physics to the holder
  and block actor weapons; fire `eAttachVehicle`/`eDetachVehicle` (`flows/actor.md`).
- **Input/camera:** `ActorInput` forwards keys (except `kUSE`) to `m_holder->OnKeyboardPress`; holder
  `Camera()` replaces the actor camera.
- **Physics:** car is a `CPHSkeleton`/physics vehicle stepped in `PhDataUpdate`
  (`engine/physics-collision.md`); the actor's own character physics is destroyed while mounted.
- **Weapons:** car mounted weapon + helicopter rocket/MGun use the shooting/bullet pipeline
  (`flows/weapon-fire-callbacks.md`).
- **Lua:** actor `eAttachVehicle`/`eDetachVehicle`; car `on_key_board`/`on_before_*` hooks; helicopter
  `SetEnemy`/`goPatrolByPatrolPath`/`on_point`/`on_hit` (helicopter is fully script-driven).
- **Config:** car definition (`driver_place` bone, driving/steering/breaking wheels, doors,
  transmission ratios, RPM/power/torque, mounted weapon section, `use_action_hint`); holder
  (`allow_weapon`/`lock_enter`/`exit_pos`/`camera_pos`).

## Threading

Main thread (actor mount/input/camera). Car physics is stepped by `xrPhysics`; helicopter movement runs
on the AI/scheduled update.

## Failure & edge cases

- **`kUSE` is exit, not forwarded:** all other keys go to the holder; `kUSE` always returns control to
  the actor — a holder that "eats" the use key can trap the player.
- **Helicopter is not mountable:** trying to treat `CHelicopter` as a holder is wrong — it has no
  `CHolderCustom`; it's an NPC aircraft controlled entirely from Lua.
- **Mount blocks weapons/physics:** while driving, the actor's weapons/inventory are blocked and its
  character physics destroyed — restored on exit; a botched detach can leave the actor without physics.
- **Enter/exit locks:** `EnterLocked`/`ExitLocked` (or `bForce`) gate mounting — a scripted locked
  vehicle won't accept the actor without force.
- **Bad `driver_place`/wheel bones:** a car with wrong bone names won't seat the driver or drive.

## Deeper

- Holder: `src/xrGame/holder_custom.h`/`.cpp` (`CHolderCustom`, `attach_Actor`/`detach_Actor`,
  `OnKeyboardPress`)
- Actor seam: `src/xrGame/Actor.cpp` (`use_HolderEx`, `attach_Vehicle`, `detach_Vehicle`,
  `eAttachVehicle`/`eDetachVehicle`), `ActorInput.cpp` (`IR_OnKeyboardPress` forward)
- Car: `src/xrGame/Car.cpp`/`.h` (`PhDataUpdate`, `Drive`, `attach_Actor`), `CarInput.cpp`,
  `CarWheels.cpp`, `CarDoors.cpp`, `CarWeapon.h`, `CarCameras.cpp`
- Helicopter: `src/xrGame/Helicopter.cpp`/`helicopter.h`, `helicopter_script.cpp` (Lua surface),
  `HelicopterMovementManager.cpp`
- Generic holder: `src/xrGame/HolderEntityObject.cpp`/`.h`

## Maintenance

- Keep the "car = mountable `CHolderCustom`; helicopter = non-mountable AI aircraft" distinction — the
  inheritance is verified and a common confusion.
- Re-grep `CActor::attach_Vehicle` / the `ActorInput` forward if the mount/input handoff changes.
- Promote the *(agent-traced)* `CHolderCustom` interface, car subcomponents, and helicopter movement to
  verified when a vehicle task needs the exact physics/Lua surface.
