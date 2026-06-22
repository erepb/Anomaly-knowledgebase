---
domain: flows/wallmarks-particles
tags: [xrgame, wallmark, decal, particles, material, impact, flow]
sources: [src/xrGame/wallmark_manager.cpp, src/xrGame/wallmark_manager.h, src/xrGame/script_wallmarks_manager.cpp, src/xrGame/script_particles.cpp, src/xrGame/script_particles.h, src/xrGame/Level_bullet_manager_firetrace.cpp, src/xrEngine/GameMtlLib.h]
related: [flows/render-frame.md, flows/renderer-wallmark-engine.md, flows/renderer-particles.md, flows/weapon-fire-callbacks.md, flows/consumables-explosives.md, flows/level-runtime.md, modlayer/assets.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: wallmarks (decals) & gameplay particles

How bullet holes / scorch / blood **decals** and gameplay **particle effects** get spawned in response
to impacts, and the **material-pair** system that maps an impact to {wallmark, particle, sound}. The
low-level decal/particle *rendering* is engine-side (`xrRender`/`xrParticles`, `flows/render-frame.md`);
this flow is the **game-side trigger + material lookup + script API**.

**Provenance:** spine personally code-verified — the classes `CWalmarkManager`, `ScriptWallmarksManager`,
`CScriptParticles` exist, and **`GMLib.GetMaterialPair(...)`** is the impact seam (verified call sites:
`CBulletManager::FireShotmark`, poltergeist ability, `physics_game`, `material_manager`). The function
bodies (`PlaceWallmarks`/`AddWallmark`, `FireShotmark`, script place/play, `SGameMtlPair` parsing) and
the engine render interface (`IRender::add_*Wallmark`) are confirmed homes but *(agent-traced — not
personally re-read)*. Material configs are `modlayer/assets.md` (base-game).

## The material-pair seam (verified entry)

The core mapping: a `GameMtl` **pair** (material-of-A × material-of-B) carries the collision response.
`SGameMtlPair` (`GameMtlLib.h`, engine-side) holds `CollideSounds`, `CollideParticles`, and
`m_pCollideMarks` (an `IWallMarkArray` of wallmark shaders), parsed from `material_pairs.ltx`
(`collide_sounds`/`collide_particles`/`collide_marks` keys). **`GMLib.GetMaterialPair(matA, matB)`**
(verified call sites) is how every impact looks up its effects. Single materials carry only flags
(`flBloodmark`, `flSuppressWallmarks`). Collision triangle materials are remapped to `GameMtl` indices
at level load (`flows/level-runtime.md` CFORM remap).

## Bullet impact → effects (agent-traced; seam verified)

`CBulletManager::FireShotmark` (`Level_bullet_manager_firetrace.cpp`, the end of the bullet pipeline
`flows/weapon-fire-callbacks.md`): on a bullet hit, **`GMLib.GetMaterialPair(bullet_material,
target_material)`** (verified), then from the pair:
- **wallmark:** `::Render->add_StaticWallmark(pair->m_pCollideMarks, point, size, tri, verts)` for static
  geometry, or `add_SkeletonWallmark(...)` for a bone/dynamic object (so decals stick to NPCs);
- **sound:** a random `CollideSounds` clip at the impact;
- **particle:** a random `CollideParticles` effect at the impact frame.
So the same `(bullet, surface)` pair drives the hole, the dust puff, and the ricochet sound together —
authored in `material_pairs.ltx`. `flSuppressWallmarks` on a material skips the decal.

## Explosion wallmarks (agent-traced)

`CWalmarkManager::PlaceWallmarks(pos)` (called from `CExplosive::Explode`,
`flows/consumables-explosives.md`) is the explosion path — **not** material-pair driven: it loads an
`explosion_marks` config (dist/size/max_count + wallmark shaders), AABB-queries nearby triangles,
raycasts occlusion, and `add_StaticWallmark`s scorch decals around the blast. `AddWallmark` is the
per-triangle placement (honoring `flBloodmark`).

## Script API (classes verified; bodies agent-traced)

- **`ScriptWallmarksManager`** (`wallmarks_manager()` in Lua) — `place(dir, start, trace_dist, size,
  section, ignore_obj, ttl [, rotation])`: raycasts, looks up the wallmark shader array by config
  section, and `add_StaticWallmark`/`add_SkeletonWallmark`s it (honoring `flSuppressWallmarks`).
  `place_skeleton` attaches directly to a bone.
- **`CScriptParticles`** (`particles_object(name)` in Lua, wraps `CParticlesObject`) — `play`/
  `play_at_pos(pos)`/`stop`/`move_to`/`set_position`/`set_direction`/`set_orientation`, plus path
  animation (`load_path`/`start_path`). The mod way to spawn a one-shot or attached particle effect.

## Engine vs game split (agent-traced)

- **Engine:** `IRender::add_StaticWallmark`/`add_SkeletonWallmark` + `IWallMarkArray` (decal mesh/shader/
  batching, `xrRender`); `CParticlesObject`/`xrParticles` (emitter simulation); `SGameMtlPair` parsing
  (`GameMtlLib`). All render/sim detail is `flows/render-frame.md`.
- **Game:** `CWalmarkManager` (explosion wallmarks), `CBulletManager::FireShotmark` (impact effects),
  `ScriptWallmarksManager`/`CScriptParticles` (Lua APIs). These *decide what/where*; the engine *renders*.

## Isolation ledger (the seams)

- **Materials:** `SGameMtlPair` (`material_pairs.ltx`) maps impacts → {sounds, particles, wallmarks};
  `GMLib.GetMaterialPair` is the lookup (`modlayer/assets.md` for the config).
- **Weapons/bullets:** `FireShotmark` at the end of the bullet pipeline spawns impact effects
  (`flows/weapon-fire-callbacks.md`).
- **Explosives:** `CWalmarkManager::PlaceWallmarks` from `CExplosive::Explode`
  (`flows/consumables-explosives.md`).
- **Render:** `IRender::add_*Wallmark` + `CParticlesObject` (`flows/render-frame.md`); `CLevel::OnFrame`
  updates wallmarks before render (`flows/level-runtime.md`).
- **Lua:** `wallmarks_manager().place(...)`, `particles_object(name):play_at_pos(...)`.
- **Config:** `materials.ltx` (per-material flags), `material_pairs.ltx` (`collide_*`), `explosion_marks`
  section, particle `.ps` files in `gamedata/particles/`.

## Threading

Main thread (impact handling + spawn). Particle simulation + decal rendering are on the render/engine
side (`flows/render-frame.md`); the bullet pipeline that calls `FireShotmark` may run MT
(`flows/weapon-fire-callbacks.md`) but the effect spawn is marshaled to the main thread.

## Failure & edge cases

- **No decal/particle/sound = bad material pair:** if `material_pairs.ltx` lacks a `(bullet, surface)`
  pair or the keys, impacts produce nothing — the most common "no bullet holes/dust" cause.
- **`flSuppressWallmarks` / `flBloodmark`:** a surface flagged suppress gets no decals; bloodmark gates
  blood specifically — material flags, not the pair.
- **Explosion wallmarks are separate:** they use `explosion_marks` config + `CWalmarkManager`, not the
  material pair — tuning blast scorch is a different knob than bullet holes.
- **Skeleton vs static decals:** decals on NPCs use `add_SkeletonWallmark` (bone-attached, move with the
  body); world decals are static — using the wrong one detaches the decal.
- **CFORM material remap required:** collision triangles must be remapped to `GameMtl` indices at load
  (`flows/level-runtime.md`) or pair lookup fails.

## Deeper

- Explosion wallmarks: `src/xrGame/wallmark_manager.cpp`/`.h` (`CWalmarkManager`, `PlaceWallmarks`,
  `AddWallmark`)
- Bullet impact: `src/xrGame/Level_bullet_manager_firetrace.cpp` (`CBulletManager::FireShotmark`,
  `GetMaterialPair`) — `flows/weapon-fire-callbacks.md`
- Script APIs: `src/xrGame/script_wallmarks_manager.cpp` (`ScriptWallmarksManager::PlaceWallmark`/
  `FindSection`), `script_particles.cpp`/`.h` (`CScriptParticles`)
- Material pairs (engine): `src/xrEngine/GameMtlLib.h`/`.cpp` (`SGameMtlPair`, `CollideSounds`/
  `CollideParticles`/`m_pCollideMarks`) — `modlayer/assets.md`
- Render side: `IRender::add_StaticWallmark`/`add_SkeletonWallmark`, `IWallMarkArray`,
  `CParticlesObject` — `flows/render-frame.md`

## Maintenance

- Keep the "material pair (`GetMaterialPair`) maps impact → wallmark+particle+sound; explosions use
  `explosion_marks` instead" distinction — it explains both the bullet and blast effect paths.
- Re-grep `CBulletManager::FireShotmark` / `GMLib.GetMaterialPair` if the impact-effect seam changes.
- Promote the *(agent-traced)* bodies + render interface to verified when a task needs the exact decal
  placement / particle spawn / material-pair parsing.
