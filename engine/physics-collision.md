---
domain: engine/physics
tags: [engine, physics, collision, cpp]
sources: [src/xrPhysics/, src/xrCDB/]
related: [engine/_overview.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-04
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

## Deeper

- Collision DB: `src/xrCDB/` (grep `CDB`, `MODEL`, `COLLIDER`, `build`)
- Physics: `src/xrPhysics/` (ODE wrappers, `PhysicsShell`, joints, ragdoll)
- `ignore_collision` consumer: grep `ignore_collision` in `src/xrGame/` and `src/xrPhysics/`

## Maintenance

- If ODE is replaced or CFORM build changes, update the relevant section.
- Keep gameplay-facing physics config params cross-referenced with `modlayer/config-ltx.md`
  (`ltx_help_ex.script` is the param catalog).
