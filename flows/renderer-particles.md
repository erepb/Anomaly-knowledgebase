---
domain: flows/renderer-particles
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/ParticleEffect.cpp, src/Layers/xrRender/ParticleEffect.h, src/Layers/xrRender/ParticleGroup.cpp, src/Layers/xrRender/ParticleGroup.h, src/Layers/xrRender/ParticleEffectDef.cpp, src/Layers/xrRender/PSLibrary.cpp, src/Layers/xrRender/PSLibrary.h, src/Layers/xrRender/dxParticleCustom.h, src/Layers/xrRender/ModelPool.cpp, src/Layers/xrRender/r__dsgraph_build.cpp]
related: [flows/wallmarks-particles.md, flows/renderer-visual-types.md, flows/renderer-scene-graph.md, flows/renderer-backend.md, engine/subsystems-misc.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: particle rendering

How particle effects/groups are defined, simulated each frame, and rendered as camera-facing billboards.
This is the render side complementing the game-side particle trigger in `flows/wallmarks-particles.md`
(F44, `CScriptParticles`/`CParticlesObject`) and the `xrParticles` simulation library noted in
`engine/subsystems-misc.md`. A particle system **is a `dxRender_Visual`**, so the scene-graph
(`flows/renderer-scene-graph.md`) renders it like any other visual.

**Provenance:** researched with a read-only agent, then spot-verified against code. The class hierarchy
(`CParticleEffect`/`CParticleGroup : dxParticleCustom : dxRender_Visual, IParticleCustom`), the
definition library (`CPSLibrary`), the sim/render entries (`CParticleEffect::OnFrame`/`Render`), the
model-pool creators (`CModelPool::CreatePE`/`CreatePG`), and the **scene-graph particle special-case**
(`if (sh->flags.bStrictB2F && !pVisual->dcast_ParticleCustom())` in `r__dsgraph_build.cpp`) are
personally code-verified. The `ParticleManager` (PAPI) sim internals, billboard fill math, and group
child-spawning are *(agent-traced — not personally re-read)*.

## Trigger / entry

Game code spawns a particle system (`CParticlesObject`/`CScriptParticles`,
`flows/wallmarks-particles.md`) which gets a renderable via the render factory and calls
`play_at_pos`/`Play`. Each frame the scheduler ticks `OnFrame` (sim) and the scene-graph calls
`Render(LOD)` (billboards).

## Sequence (verified structure)

### Object model

- **`CParticleEffect`** — a single emitter; `: dxParticleCustom : dxRender_Visual, IParticleCustom` (so
  it's a scene-graph visual). Holds opaque handles into the `xrParticles` `ParticleManager` (the PAPI
  sim lib) plus a `CPEDef` (effect definition: shader/texture, frame anim, collision, time limit).
- **`CParticleGroup`** — a composite of effects (also a `dxParticleCustom`); schedules child effects by
  time and spawns particle-tied or free children. It does not render itself; the scene-graph renders
  each child effect.
- **`CPSLibrary`** — the global definition library (`CPEDef`/`CPGDef`) loaded from `particles.xr` /
  `$game_particles$`; each `CPEDef` pre-creates its shader at load.
- **Model pool:** `CModelPool::CreatePE`/`CreatePG` instantiate a `CParticleEffect`/`CParticleGroup`
  (types `MT_PARTICLE_*`) and `Compile` it from a def (`flows/renderer-visual-types.md`).

### Simulation tick

`CParticleEffect::OnFrame(dt)` accumulates a fixed timestep (capped to a few sub-steps to avoid stalls),
calls `ParticleManager()->Update(...)` (the PAPI lib advances positions/spawns/kills via the effect's
action list — gravity, velocity, color-over-life, …), then runs engine-side post-processing
(`ExecuteAnimate` for frame animation, `ExecuteCollision` for world-ray bounce/kill), and recomputes the
bounding box (drives culling). When deferred-stopped and all particles are dead, it clears the playing
flag. `CParticleGroup::OnFrame` schedules children by time and merges their boxes. *(PAPI internals +
group child-spawning agent-traced.)*

### Render

`CParticleEffect::Render(LOD)` (called from the scene-graph) gets the live particle array, locks the
dynamic VB (`R_DStreams`, `flows/renderer-backend.md`), and per particle emits a 4-vertex quad oriented
either camera-facing or aligned to the particle's velocity/path, with color/size/UV (frame-animated),
then `set_Geometry` + `RCache.Render`. Particles use `FVF::LIT` and the def's shader
(`flows/renderer-resources-shaders.md`, `Blender_Particle` blend modes). *(fill math agent-traced.)*

### Scene-graph routing (verified special-case)

When a particle visual is inserted (`r__dsgraph_insert_dynamic`), the **strict-back-to-front sort is
bypassed for particles**: `if (sh->flags.bStrictB2F && !pVisual->dcast_ParticleCustom())` — i.e. a
particle with a B2F shader does *not* go into the per-item `Sorted` list; particles render in the
standard dynamic pass instead (distort/emissive routing still applies via shader flags). The dsgraph
also expands `CParticleGroup` children via `dcast_ParticleCustom` (`flows/renderer-scene-graph.md`).

## Function digests (spine)

- **`ParticleEffect.cpp:CParticleEffect::OnFrame`** — *Does:* fixed-step sim (PAPI `Update` + engine
  animate/collision) + bbox + stop check. *Reads:* def flags, particle array. *Calls:*
  `ParticleManager()->Update`/`GetParticles`, `ExecuteAnimate`/`ExecuteCollision`. *Called by:*
  scheduler. *(verified entry; PAPI body agent-traced.)*
- **`ParticleEffect.cpp:CParticleEffect::Render`** — *Does:* build camera/path-aligned billboards from
  live particles → dynamic VB → `RCache.Render`. *Calls:* `RCache.Vertex.Lock/Unlock`, `set_Geometry`,
  `Render`. *Called by:* scene-graph. *(verified entry; fill math agent-traced.)*
- **`ParticleGroup.cpp:CParticleGroup::OnFrame`** — *Does:* time-schedule + tick child effects, merge
  boxes. *Gotcha:* the group itself isn't rendered; children are. *(verified entry.)*
- **`ModelPool.cpp:CModelPool::CreatePE`/`CreatePG`** — *Does:* instantiate + `Compile` a particle
  effect/group from a def. *Calls:* `Instance_Create(MT_PARTICLE_*)`, `Compile`. *(verified.)*
- **`r__dsgraph_build.cpp` particle case** — *Does:* exclude particles from strict-B2F sort
  (`!dcast_ParticleCustom()`); expand group children. *(verified.)*

## Isolation ledger (the seams)

- **Owned state:** `CParticleEffect`/`CParticleGroup` (live system + def + PAPI handles); `CPSLibrary`
  (the def library).
- **Game trigger seam:** `CParticlesObject`/`CScriptParticles` (`flows/wallmarks-particles.md`) →
  factory creates the visual → `play_at_pos`/`UpdateParent`/`Play` → `add_Dynamic` into the scene-graph.
- **Sim seam:** `ParticleManager` (PAPI, the `xrParticles`/3rd-party lib — `engine/subsystems-misc.md`)
  does the particle physics; the engine adds frame-anim + collision.
- **Definition seam:** `CPSLibrary` ← `particles.xr` / `$game_particles$` (`.pe`/`.pg`); shaders
  pre-created per def.
- **Scene-graph seam:** particle visual rendered via `flows/renderer-scene-graph.md` (dynamic pass; B2F
  bypass; distort/emissive by shader flag).
- **Backend seam:** dynamic VB streaming + draw via `RCache` (`flows/renderer-backend.md`); shader from
  `flows/renderer-resources-shaders.md` (`Blender_Particle`).
- **Config:** particle-update coefficient, SSA discard, fog-distance cull.

## Threading

Sim (`OnFrame`) on the scheduler/update path; render on the render thread. Both per frame.

## Failure & edge cases

- **Deferred stop** — a stopped looping effect keeps living until its particles die (no instant
  disappear); the system frees only when empty.
- **B2F bypass** — particles intentionally skip strict sorting; a "particle sort order" bug is usually
  blend-mode/Z-write, not the sorted list.
- **Sub-step cap** — large frame gaps clamp sim sub-steps, so very low FPS slows particle time.
- **Definition-driven** — no hot reload of `particles.xr` without a reload; a missing def yields no
  visual.

## Deeper

- Effect/group: `xrRender/ParticleEffect.cpp`/`.h` (`OnFrame`, `Render`), `ParticleGroup.cpp`/`.h`,
  `ParticleEffectDef.cpp` (animate/collision), `dxParticleCustom.h` (the visual base)
- Library: `xrRender/PSLibrary.cpp`/`.h` (`CPSLibrary`)
- Pool + routing: `xrRender/ModelPool.cpp` (`CreatePE`/`CreatePG`), `r__dsgraph_build.cpp` (particle
  case)
- Game trigger: `flows/wallmarks-particles.md`; sim lib: `engine/subsystems-misc.md`; scene-graph:
  `flows/renderer-scene-graph.md`; backend: `flows/renderer-backend.md`

## Maintenance

- `CParticleEffect::OnFrame`/`Render` + the dsgraph particle case are the spine — re-grep if the sim/
  render or B2F handling changes.
- Promote the *(agent-traced)* PAPI `Update` path, billboard fill, and group child-spawning to verified
  when a particle task needs exact behavior.
- Keep "a particle system IS a visual; sim is PAPI, the engine adds anim/collision; B2F is bypassed."
