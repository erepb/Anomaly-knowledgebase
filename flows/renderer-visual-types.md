---
domain: flows/renderer-visual-types
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/FBasicVisual.cpp, src/Layers/xrRender/FBasicVisual.h, src/Layers/xrRender/FVisual.cpp, src/Layers/xrRender/FProgressive.cpp, src/Layers/xrRender/FHierrarhyVisual.cpp, src/Layers/xrRender/FSkinned.cpp, src/Layers/xrRender/FTreeVisual.cpp, src/Layers/xrRender/FLOD.cpp, src/Layers/xrRender/ModelPool.cpp, src/Layers/xrRender/ModelPool.h]
related: [flows/renderer-scene-graph.md, flows/renderer-skeleton.md, flows/renderer-backend.md, flows/renderer-resources-shaders.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: renderable visual types & the model pool

The `dxRender_Visual` class tree (how an `.ogf` model becomes a renderable), the **model pool**
(load-once / instance-many), LOD selection, and the per-visual `Render(LOD)` that submits geometry to
the backend. This is the "what object is drawn" layer between the scene-graph
(`flows/renderer-scene-graph.md`, which calls `pVisual->Render(LOD)`) and the backend
(`flows/renderer-backend.md`, which receives the draw).

**Provenance:** researched with a read-only agent, then spot-verified against code. The class
hierarchy (`dxRender_Visual` base, `Fvisual`/`FProgressive`/`FHierrarhyVisual` derivations), the model
pool two-tier instancing (`CModelPool::Create` body read in full: Pool-recycle → reference-find →
`Instance_Load` → `Instance_Duplicate`), and the `Render(LOD)` per-subclass entries are personally
code-verified. The OGF chunk parsing, FSkinned HW-skinning vertex setup, and FTreeVisual wind
constants are *(agent-traced — not personally re-read)*.

## Trigger / entry

A game object needs a visual: it calls the render factory (`::Render->model_Create(name)`) →
`CModelPool::Create(name)`. Later, each frame, the scene-graph calls `pVisual->Render(LOD)` per
collected visual.

## Sequence (verified)

### The `dxRender_Visual` tree

`dxRender_Visual : IRenderVisual` (`FBasicVisual.h`) is the base — it carries the `Type` (OGF model
type), the `ref_shader` (shared shader from `flows/renderer-resources-shaders.md`), and `vis` (bbox +
bsphere for culling). Subclasses by OGF type:
- **`Fvisual : dxRender_Visual, IRender_Mesh`** — a simple static mesh (one VB/IB).
- **`FProgressive : Fvisual`** — progressive mesh: a slide-window index table where `Render(LOD)`
  selects a triangle subset (LOD by index window, *not* mesh re-simplification).
- **`FHierrarhyVisual : dxRender_Visual`** — a container of child visuals (no geometry of its own;
  children rendered individually).
- **`CSkeletonX_ST`/`CSkeletonX_PM`** (`FSkinned.cpp`) — skinned mesh (static / progressive); the
  per-render-vertex skinning helper that uploads bone matrices. Driven by the skeleton
  (`flows/renderer-skeleton.md` — note `CKinematics` *is* an `FHierrarhyVisual`).
- **`FTreeVisual_ST`/`FTreeVisual_PM`** — foliage with wind/wave shader constants.
- **`FLOD : FHierrarhyVisual`** — far-distance impostor (8 pre-baked quads); its `Render` is currently
  disabled in this build — far LOD impostors go through `r_dsgraph_render_lods`
  (`flows/renderer-scene-graph.md`) instead.
- **Particle visuals** (`PS::CParticleEffect`/`CParticleGroup`) — for `flows/wallmarks-particles.md`
  / a future particle-render flow.

### Load → subclass dispatch

`CModelPool::Instance_Load` opens the `.ogf`, reads its header type, calls `Instance_Create(type)`
(the `xr_new<Fvisual/FProgressive/FHierrarhyVisual/CSkeletonX_*/FLOD/…>` switch), then
`visual->Load(name, reader, flags)`. `dxRender_Visual::Load` reads the common header (type, shader+
texture → `getShader`, bbox/bsphere); each subclass then reads its geometry chunks (VB/IB containers,
slide-window data, children, bone-weighted vertices). *(chunk parsing agent-traced.)*

### Two-tier instancing (verified)

`CModelPool::Create(name)`: lowercase/strip-ext the name, then —
1. **Pool** (recycle): if an unused instance of this name sits in `Pool`, `Spawn()` it and return.
2. **Reference**: `Instance_Find` the base model in the registry; if absent, `Instance_Load` it once
   (registered as the shared reference).
3. **Instance**: `Instance_Duplicate(Base)` — `Instance_Create(Base->Type)` + `Copy(Base)` (shallow:
   shares the geometry VB/IB by ref-count, copies shader/vis), `Spawn`, register in `Registry`. Return
   the per-object instance.

So **geometry/shaders load once and are shared by pointer**; each game object gets a cheap instance
wrapper. `Delete` defers during rendering, then either pools the instance for reuse or `Discard`s it
(decrement ref; free the base when it hits zero).

### Render(LOD)

The scene-graph computes `LOD ∈ [0,1]` from screen-space area and calls `pVisual->Render(LOD)`:
- `Fvisual::Render` ignores LOD — `RCache.set_Geometry(rm_geom)` + `RCache.Render(triangle list, …)`.
- `FProgressive::Render` maps LOD → a slide-window index and submits only that window's triangles.
- `CSkeletonX::Render` uploads bone matrices (`flows/renderer-skeleton.md`) then submits skinned
  geometry.
- `FTreeVisual_*::Render` sets wind/wave/xform constants, then submits (PM variant adds LOD window).
- `FHierrarhyVisual` submits nothing itself — the scene-graph renders its children.

All ultimately drive `RCache` (`flows/renderer-backend.md`).

## Function digests (spine)

- **`ModelPool.cpp:CModelPool::Create`** — *Does:* return a per-object visual instance via
  Pool-recycle / reference-find / load / duplicate. *Reads/Writes:* `Pool`, `Registry`, the reference
  models. *Calls:* `Instance_Find`, `Instance_Load`, `Instance_Duplicate`. *(verified.)*
- **`ModelPool.cpp:CModelPool::Instance_Load`** — *Does:* open OGF, dispatch type → subclass, `Load`.
  *Calls:* `Instance_Create`, `visual->Load`. *(verified entry; OGF parse agent-traced.)*
- **`ModelPool.cpp:CModelPool::Instance_Duplicate`** — *Does:* clone a reference into an instance
  (`Copy` = shared geometry, copied shader/vis). *Gotcha:* geometry is ref-counted, not duplicated.
- **`ModelPool.cpp:CModelPool::Discard`** — *Does:* unregister instance, decrement reference, free base
  at zero. *Gotcha:* `Delete` defers to a queue while `g_bRendering`.
- **`FBasicVisual.cpp:dxRender_Visual::Load`** — *Does:* read common header (type/shader/bbox).
  *Calls:* `getShader`. *Called by:* every subclass `Load`.
- **`FVisual.cpp:Fvisual::Render` / `FProgressive.cpp:FProgressive::Render`** — *Does:* submit geometry
  (FProgressive picks a slide-window by LOD). *Calls:* `RCache.set_Geometry`/`RCache.Render`. *(verified.)*

## Isolation ledger (the seams)

- **Owned state:** `CModelPool` — reference models (load-once), `Registry` (live instances), `Pool`
  (recycled instances). Each `dxRender_Visual` owns its `Type`/`shader`/`vis` + ref-counted geometry.
- **Request seam:** game ↔ pool via the render factory (`::Render->model_Create` →
  `CModelPool::Create`); game objects hold an `IRenderVisual*`.
- **Scene-graph seam:** the scene-graph collects the visual (its `vis` bbox/sphere drives culling) and
  calls `Render(LOD)` (`flows/renderer-scene-graph.md`).
- **Resource seam:** shader from `flows/renderer-resources-shaders.md`; VB/IB from the renderer geometry
  pool; the actual draw via `RCache` (`flows/renderer-backend.md`).
- **Skeleton seam:** `CKinematics`/`CKinematicsAnimated` *are* `FHierrarhyVisual`s whose skinned-mesh
  children are `CSkeletonX_*`; bone matrices come from `flows/renderer-skeleton.md`.
- **File system:** `.ogf` model files (loaded once per name).

## Threading

Model load/instancing happens on the main thread (mostly at level load / object spawn). `Render(LOD)`
runs on the render thread during the frame.

## Failure & edge cases

- **Shared geometry** — two objects of the same model share one VB/IB; do not assume per-instance
  geometry edits.
- **FLOD::Render disabled** — far impostors render via the scene-graph LOD path, not `FLOD::Render`.
- **Delete-during-render** is queued — a freed visual may persist until the frame ends.
- **Pool recycling** means a "new" visual may be a reused instance — `Spawn()`/`Depart()` bracket the
  reuse lifecycle.

## Deeper

- Base + subclasses: `xrRender/FBasicVisual.cpp`/`.h` (`dxRender_Visual`), `FVisual.cpp` (`Fvisual`),
  `FProgressive.cpp`, `FHierrarhyVisual.cpp`, `FSkinned.cpp` (`CSkeletonX_ST/PM`), `FTreeVisual.cpp`,
  `FLOD.cpp`
- Model pool: `xrRender/ModelPool.cpp`/`.h` (`Create`, `Instance_Load`, `Instance_Duplicate`, `Discard`)
- Consumers: `flows/renderer-scene-graph.md` (Render(LOD)), `flows/renderer-backend.md` (draw);
  skeleton: `flows/renderer-skeleton.md`; shaders: `flows/renderer-resources-shaders.md`

## Maintenance

- `CModelPool::Create` + `Instance_Duplicate` are the instancing spine — re-grep if the
  reference/instance/pool model changes.
- Promote the *(agent-traced)* OGF chunk parsing + FSkinned HW-skinning vertex setup + FTreeVisual wind
  constants to verified by reading the per-subclass `Load`/`Render` when a model/LOD task needs it.
- Keep the "shared geometry, two-tier instancing" fact — it explains the pool/registry/refcount model.
