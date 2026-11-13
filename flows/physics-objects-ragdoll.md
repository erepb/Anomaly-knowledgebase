---
domain: flows/physics-objects-ragdoll
tags: [xrgame, physics, ragdoll, death, shell, lamp, flow]
sources: [src/xrGame/PhysicsShellHolder.cpp, src/xrGame/PhysicsShellHolder.h, src/xrGame/PhysicObject.cpp, src/xrGame/PhysicObject.h, src/xrGame/PhysicsSkeletonObject.cpp, src/xrGame/HangingLamp.cpp, src/xrGame/CharacterPhysicsSupport.cpp, src/xrGame/CharacterPhysicsSupport.h, src/xrGame/character_shell_control.cpp]
related: [engine/physics-collision.md, flows/item-base.md, flows/damage-hit.md, flows/object-lifecycle.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: physics objects & death ragdoll

The game-side objects that own a physics shell (crates, lamps, skeleton props) and — the headline —
how a creature switches from **animated** (kinematics-driven) to a **ragdoll** (physics-driven body) on
death. The low-level solver (ODE, CFORM, `CPhysicsShell`) is `engine/physics-collision.md`; this flow
is the *game* layer that creates/activates shells and the animated→ragdoll transition. The shell
activate/deactivate on drop/pickup is shared with `flows/item-base.md`.

**Provenance:** spine personally code-verified — `CPhysicObject : CPhysicsShellHolder, CPHSkeleton`,
`CCharacterPhysicsSupport : CPHSkeleton, CPHDestroyable, …`, and the existence of
`CCharacterPhysicsSupport::in_Die`/`CreateShell` (the ragdoll entry). The `CPhysicsShellHolder`
create/activate/deactivate bodies, the full death sequence (`KillHit`/`death_anims`/`imotion`/
`EndActivateFreeShell`/friction evolution), `HangingLamp`, and `ClimableObject` are confirmed homes but
*(agent-traced — not personally re-read)*.

## Physics objects (entry verified; bodies agent-traced)

- **`CPhysicsShellHolder`** — the base for any game object with a physics body; owns `m_pPhysicsShell`
  (`PPhysicsShell()`), with `create_physic_shell`/`activate_physic_shell`/`deactivate_physics_shell`
  and `PHHit` (forward a hit's impulse to the shell). The drop/pickup activate/deactivate is
  `flows/item-base.md`'s `OnH_B_*`.
- **`CPhysicObject` : `CPhysicsShellHolder`, `CPHSkeleton`** (verified) — a generic world object (crate/
  barrel/door). `Load`/`CreateBody` build a shell of `EPOType` **box** (simple rigid body), **chain**,
  or **skeleton** (multi-bone, `P_build_Shell` with `fixed_bones` from `.ltx`). Optional
  `on_physic_contact` Lua callback (modded build) and `is_ai_obstacle`. *(agent-traced.)*
- **`CPhysicsSkeletonObject`** — a multi-bone destructible skeleton (chandelier, statue); shell drives
  bone transforms, breakable variants stay scheduled. *(agent-traced.)*
- **`CHangingLamp`** — couples a physics shell + a dynamic light (`light_render`/`light_ambient`
  tracking a bone, `color_animator`) + breakable health: a hit reduces `fHealth`, breaks/flickers, and
  the lamp can swing on its shell. *(agent-traced.)*
- **`CClimableObject`** — a static-geometry ladder (no dynamic shell); orientation axes + contact
  callback let the actor character-controller climb. *(agent-traced.)*

## Death ragdoll — animated → physics (entry verified; sequence agent-traced)

A living creature runs **kinematics animation + a character movement controller**
(`CPHMovementControl`); on death it hands its skeleton to a **physics shell** so the body falls
naturally. `CCharacterPhysicsSupport` (on every `CEntityAlive`) owns this. The sequence
*(agent-traced; `in_Die`/`CreateShell` verified to exist)*:

1. **`CEntityAlive::Die`** → **`CCharacterPhysicsSupport::in_Die(hit)`** — if killed by a saved hit,
   use it for the throw direction/impulse.
2. **`KillHit`** — `TestForWounded` (raycast pelvis to detect a crawling pose), pick a death animation
   from `death_anims` by hit angle; set up an `imotion_position` blend (animation-blended ragdoll) or
   go straight to a free ragdoll; scale the fatal impulse (`skel_fatal_impulse_factor`, full for
   explosions).
3. **`CreateShell`** — destroy the live movement/IK controllers, swap the skeleton root (`bip01` →
   `bip01_pelvis`), build the physics skeleton from the current kinematics
   (`preBuild_FromKinematics`→`Build`), move it into `m_pPhysicsShell`, `set_Kinematics` (bone↔element
   binding), `RunSimulation`; **`m_eState = esDead`**, transfer the active weapon's collision to the
   shell.
4. **`EndActivateFreeShell`** — correct penetration, apply the start velocity
   (`apply_start_velocity_factor`) and the hit impulse (`m_pPhysicsShell->applyHit`) so the body is
   thrown; set air resistance + a contact callback.
5. **continuous ragdoll** — the shell drives all bone transforms; joint resistance and skin friction
   **evolve over time** (`UpdateFrictionAndJointResistanse`, from `ph_skeleton_*` `.ltx` params) so the
   body first flops then settles; `SetRagDoll` puts it in the ragdoll collision class. On NetDestroy
   the shell is deactivated/deleted.

The reverse (a *living* hit) goes through `CPHMovementControl::ApplyHit` (a push, not ragdoll); only
death switches to `esDead`/physics. A non-fatal hit may play a `character_hit_animation_controller`
reaction.

## Isolation ledger (the seams)

- **xrPhysics:** `P_build_Shell`/`P_create_Shell`, `CPhysicsShell`/`CPhysicsElement`/`CPhysicsJoint`,
  ODE contact callbacks (`engine/physics-collision.md`); the shell interpolates the render transform
  each frame.
- **Damage/death:** `CEntityAlive::Die`→`in_Die`→`KillHit`→`CreateShell` (`flows/damage-hit.md`); the
  killing hit's impulse throws the body; `PHHit`/`applyHit` for physics pushes.
- **Animation/skeleton:** the live body is kinematics-driven; ragdoll binds kinematics to the shell
  (`set_Kinematics`); `imotion_position` blends a death animation with physics
  (`flows/animation-callbacks.md`).
- **Item-base:** generic objects activate/deactivate their shell on drop/pickup
  (`flows/item-base.md` `OnH_B_*`).
- **Lights:** `CHangingLamp` ties a dynamic light to its shell bone.
- **Lua:** `on_physic_contact` (modded `CPhysicObject` contact); no direct ragdoll Lua hook (death is
  scripted via the `eDeath` callback, `flows/damage-hit.md`).
- **Config:** physics shell `type`/`mass`/`fixed_bones`; the `ph_skeleton_*` / `ph_skel_*` ragdoll
  tuning (`fatal_impulse_factor`, `airr_*_factor`, `hinger_factor1`, `ddelay`, `skin_friction_start/end`,
  `shot_up_factor`, `after_death_velocity_factor`); lamp light params.

## Threading

Game-side create/activate and the death transition run on the main thread; the physics shell is stepped
by `xrPhysics` (its own update), and the body interpolates its render transform on the main thread.

## Failure & edge cases

- **Script-frozen shells ignore each other (2026.11.09)** — `CPHCollideValidator` gained a
  `cbClassFrozen` / `cbNCClassFrozen` class pair; shells frozen from script (`physics_shell_scripted.cpp`)
  get those bits (alongside the existing `cbNCStatic`), so frozen shells stop colliding with one another.
- **Scale is a vector** — physics shell/element scale is per-axis (`PHShell`, `PHElement`); script
  `get_scale()` returns a vector and `set_scale()` takes a number or a vector. Script attachments use it
  (`features/script-attachments.md`).

- **Ragdoll only on death:** a living hit pushes via the movement controller; the kinematics→physics
  switch (`esDead`) happens once, in `CreateShell`. A body that "T-poses" or doesn't flop usually means
  the death animation/blend (`death_anims`) or the kinematics→shell bind failed.
- **Friction evolves:** the body is floppy right after death then stiffens (`UpdateFrictionAndJoint
  Resistanse`); tuning `ph_skeleton_ddelay`/`skin_friction_*` changes how long it slides.
- **Wounded pose:** `TestForWounded` picks different death handling for a crawling NPC — separate
  friction delays.
- **Skeleton shell needs `fixed_bones` right:** a `CPhysicObject` skeleton with wrong `fixed_bones`
  collapses or floats; a box object ignores them.
- **Impulse scale:** explosions use full fatal impulse; normal hits a fraction — a body flying too far/
  not enough is the `skel_fatal_impulse_factor`/`after_death_velocity_factor`.

## Deeper

- Holder base: `src/xrGame/PhysicsShellHolder.cpp`/`.h` (`PPhysicsShell`, `create_physic_shell`,
  `activate_physic_shell`, `PHHit`)
- Generic object: `src/xrGame/PhysicObject.cpp`/`.h` (`CreateBody`, `CreateSkeleton`, `EPOType`),
  `PhysicsSkeletonObject.cpp`, `HangingLamp.cpp` (light+break), `ClimableObject.cpp`
- Ragdoll: `src/xrGame/CharacterPhysicsSupport.cpp`/`.h` (`in_Die`, `KillHit`, `CreateShell`,
  `EndActivateFreeShell`, `UpdateDeathAnims`, `m_eState`/`esDead`), `character_shell_control.cpp`
  (`set_fatal_impulse`, `apply_start_velocity_factor`, `UpdateFrictionAndJointResistanse`, `Load`)
- Solver: `engine/physics-collision.md`; death trigger: `flows/damage-hit.md`

## Maintenance

- Keep the "ragdoll only on death (kinematics→shell at `esDead`), living hits push via the movement
  controller" distinction — it is the core of the system.
- Re-grep `CCharacterPhysicsSupport::CreateShell`/`in_Die` if the death transition changes.
- Promote the *(agent-traced)* death sequence + `CPhysicObject`/`HangingLamp` bodies to verified when a
  ragdoll-tuning or destructible-object task needs the exact math/params.
