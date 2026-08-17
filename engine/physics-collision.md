---
domain: engine/physics
tags: [engine, physics, collision, cpp]
sources: [src/xrPhysics/, src/xrPhysics/PHSimpleCharacter.cpp, src/xrPhysics/PHSimpleCharacter.h, src/xrPhysics/PHSimpleCharacterInline.h, src/xrGame/PHMovementControl.cpp, src/xrCDB/]
related: [engine/_overview.md, engine/xrGame.md, flows/damage-hit.md, GLOSSARY.md]
updated: 2026-08-17
---

# Physics & collision

Two cooperating modules: `xrCDB` (static collision queries) and `xrPhysics` (dynamic rigid-body
simulation via ODE).

## xrCDB — collision database

Builds and queries **CFORM** collision geometry (level meshes). Ray/box/frustum queries against the
static world use the CDB tree (AABB tree of triangles). Used by AI line-of-sight, bullets, camera,
and the renderer's visibility/occlusion. See `GLOSSARY.md#cform`.

## xrPhysics — dynamic physics

ODE-based rigid bodies, ragdolls, joints, vehicles, and physics shells attached to game objects.
Handles object↔object and object↔world collision response. The `ignore_collision` LTX param
(documented in `gamedata/scripts/ltx_help_ex.script`) lets configs exempt specific object pairs.

## Interaction with gameplay

Physics shells are created for `xrGame` objects (corpses, thrown/physic objects, doors). Bullet hits
combine CDB world traces with physics-object hits. Most tuning is per-object via configs;
core behavior is here.

**Collision-damage callbacks are resolved by id, never cached** (2026-08 crash fix,
`CPHMovementControl::Calculate`). `CPHSimpleCharacter::SCollisionDamageInfo` used to store the colliding
object's `ICollisionHitCallback*` from `UpdateDynamicDamage`; if that object was destroyed before the
next `Calculate`, the pointer dangled. Now only `m_obj_id` is stored and
`CPHSimpleCharacter::HitCallback()` re-resolves it each time — checking the damage type is `ctObject`,
the id is valid, `net_Find` still returns the object, and `ObjectGetDestroy()` is false — while
`Calculate` also null-checks `CollisionDamageInfo()` itself before use. The general rule for this module:
**hold ids across frames, not pointers**, and re-resolve at use.

## Deeper

- Collision DB: `src/xrCDB/` (grep `CDB`, `MODEL`, `COLLIDER`, `build`)
- Physics: `src/xrPhysics/` (ODE wrappers, `PhysicsShell`, joints, ragdoll)
- `ignore_collision` consumer: grep `ignore_collision` in `src/xrGame/` and `src/xrPhysics/`

## Maintenance

- If ODE is replaced or CFORM build changes, update the relevant section.
- Keep gameplay-facing physics config params cross-referenced with `modlayer/config-ltx.md`
  (`ltx_help_ex.script` is the param catalog).
