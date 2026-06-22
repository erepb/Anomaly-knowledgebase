---
domain: flows/renderer-3dfluid
tags: [engine, renderer, dx10, dx11, flow]
sources: [src/Layers/xrRenderDX10/3DFluid/dx103DFluidManager.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidManager.h, src/Layers/xrRenderDX10/3DFluid/dx103DFluidGrid.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidData.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidEmitters.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidRenderer.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidVolume.cpp, src/Layers/xrRenderDX10/3DFluid/dx103DFluidBlenders.cpp, src/Layers/xrRenderPC_R4/r4.cpp]
related: [flows/renderer-blender-compile.md, flows/renderer-backend.md, flows/renderer-hw-device.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: DX10 3D fluid simulation (niche / dormant)

A GPU grid-based (Eulerian) volumetric fluid sim (smoke/gas) in `xrRenderDX10/3DFluid/`: a 3D voxel
grid stepped each frame by a Navier-Stokes solver (advection → forces → divergence → pressure solve →
projection) and ray-marched as a volume. **Niche + effectively dormant in this build** (see below);
documented for completeness (OOS) — drill in only if a task targets volumetric smoke.

**Provenance:** researched with a read-only agent, then spot-verified against code. The manager spine
(`dx103DFluidManager::Initialize`/`Update`/`RenderFluid`), the R4 wiring (`r4.cpp` includes it, sets
`o.volumetricfog = ps_r2_ls_flags.test(R3FLAG_VOLUMETRIC_SMOKE)`, calls `FluidManager.Initialize(70,70,70)`
+ `SetScreenSize` + `Destroy`), and the volume visual (`Type = MT_3DFLUIDVOLUME`) are personally
code-verified. **Correction to the agent draft:** the per-volume render call
`dx103DFluidVolume::Render` has its **`FluidManager.RenderFluid(...)` commented out** (only the debug
box remains) — so although the manager is initialized at R4 startup, the fluid is **not actually
rendered per volume in this build** → effectively dormant/experimental, *not* "active". The sim-pass
ordering, ray-march, and blender details are *(agent-traced — not personally re-read)*.

## Status (verified)

- The fluid manager **is** wired at R4 init: `r4.cpp` includes `dx103DFluidManager.h`, gates it on
  `o.volumetricfog` (`R3FLAG_VOLUMETRIC_SMOKE`), and calls `FluidManager.Initialize(70,70,70)` /
  `SetScreenSize` / `Destroy`.
- But `dx103DFluidVolume::Render` (the per-`MT_3DFLUIDVOLUME` visual) has `FluidManager.RenderFluid(...)`
  **commented out** — the function only renders a debug bounding box. So in practice the fluid sim does
  **not** drive any visible output in this build. Treat it as present-but-dormant.

## Sequence (structure — agent-traced)

### Sim step — `dx103DFluidManager::Update(FluidData, timestep)`

Ordered GPU passes, each a draw into the 3D grid slices (`dx103DFluidGrid::DrawSlices`): update
obstacles → advect color/density (BFECC error-corrected, decay) → advect velocity (± buoyancy/gravity)
→ vorticity confinement → apply emitter forces (Gaussian density/velocity splats) → compute divergence
→ Jacobi pressure-Poisson solve (iterated) → project velocity to divergence-free. Velocity/pressure
volume textures ping-pong between render targets; density/temperature live in the per-volume `FluidData`
(swapped in/out around the update). *(agent-traced.)*

### Render — `dx103DFluidManager::RenderFluid(FluidData)` → `dx103DFluidRenderer::Draw`

Ray-march the density volume against its world bounding box: compute ray data (back/front faces) → edge
detect → raycast fog/fire transfer function with lighting → composite into the scene. *(agent-traced;
and recall the call is commented out at the volume level.)*

### Shaders

The sim + render passes are configured by the fluid blenders (`dx103DFluidBlenders.cpp`:
`CBlender_fluid_advect`/`_simulate`/`_emitter`/`_raydata`/`_raycast`) via the standard blender mechanism
(`flows/renderer-blender-compile.md`), with custom constant setups for grid dims/samplers. *(agent-traced.)*

## Function digests (spine)

- **`dx103DFluidManager.cpp:dx103DFluidManager::Initialize`** — *Does:* allocate the 3D volume textures
  + grid/renderer/emitters/obstacles; compile sim techniques; gated by `o.volumetricfog`. *Called by:*
  `CRender` create (`r4.cpp`). *(wiring verified; body agent-traced.)*
- **`dx103DFluidManager.cpp:dx103DFluidManager::Update`** — *Does:* the ordered NS solver passes over
  the grid. *(entry verified; body agent-traced.)*
- **`dx103DFluidManager.cpp:dx103DFluidManager::RenderFluid`** — *Does:* ray-march/volume-render the
  grid via `dx103DFluidRenderer::Draw`. *Gotcha:* the per-volume caller is commented out in this build.
  *(entry verified; body agent-traced.)*
- **`dx103DFluidVolume.cpp:dx103DFluidVolume::Render`** — *Does:* in this build, only renders a debug
  box (the `RenderFluid` call is commented out). *Type:* `MT_3DFLUIDVOLUME`. *(verified.)*

## Isolation ledger (the seams)

- **Owned state:** the global `FluidManager` (grid volume textures, emitters, obstacles, renderer); the
  per-volume `dx103DFluidData` (density/velocity/pressure volumes + emitter profile).
- **Driver seam:** `r4.cpp` initializes/destroys the manager (gated by `volumetricfog`); the
  `MT_3DFLUIDVOLUME` visual *would* drive per-frame `Update`/`RenderFluid` — but that call is commented
  out (dormant).
- **Backend seam:** DX10/11 3D textures + slice draws via `HW.pContext`/`RCache`
  (`flows/renderer-backend.md`, `flows/renderer-hw-device.md`); DX10 has no native 3D RTV, so depth
  slices are iterated.
- **Blender seam:** `dx103DFluidBlenders` (`flows/renderer-blender-compile.md`).
- **Config:** `R3FLAG_VOLUMETRIC_SMOKE` → `o.volumetricfog`; per-volume `.ini` profile (emitters,
  decay, confinement, fog/fire type); a DEBUG `fog_reload` console command.
- **Self-contained:** isolated to `3DFluid/`; minimal coupling to the rest of the engine (obstacles do
  shallow physics-shell queries only).

## Threading / failure / edge cases

- **Dormant** — the per-volume render is commented out; enabling it would require uncommenting +
  ensuring `volumetricfog` is set. Don't assume it produces visible output as shipped.
- **Heavy** — a 70³ grid with multiple full-volume passes per frame is expensive; intended as an
  optional feature.
- **DX10/11 only** — relies on 3D render targets + geometry-shader slice routing.

## Deeper

- Manager: `xrRenderDX10/3DFluid/dx103DFluidManager.cpp`/`.h` (`Initialize`/`Update`/`RenderFluid`)
- Grid/data/emitters/obstacles/renderer/volume: `dx103DFluidGrid.cpp`, `dx103DFluidData.cpp`,
  `dx103DFluidEmitters.cpp`, `dx103DFluidObstacles.cpp`, `dx103DFluidRenderer.cpp`, `dx103DFluidVolume.cpp`
- Shaders: `xrRenderDX10/3DFluid/dx103DFluidBlenders.cpp` (`flows/renderer-blender-compile.md`)
- Wiring: `xrRenderPC_R4/r4.cpp` (`FluidManager.*`); backend/device: `flows/renderer-backend.md`,
  `flows/renderer-hw-device.md`

## Maintenance

- Niche/dormant — keep it short. If volumetric smoke is revived, re-check whether
  `dx103DFluidVolume::Render`'s `RenderFluid` call is uncommented and deepen the *(agent-traced)* sim/
  render bodies then.
- Keep the verified facts: manager initialized in `r4.cpp` (gated by `volumetricfog`), but the per-volume
  `RenderFluid` is commented out → effectively dormant in this build.
