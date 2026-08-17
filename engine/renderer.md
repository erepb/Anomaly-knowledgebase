---
domain: engine/renderer
tags: [engine, renderer, shaders, r1, r2, r3, r4, dx9, dx10, dx11, blender, spom]
sources: [src/Layers/, src/Layers/xrAPI/xrAPI.cpp, src/Layers/xrRender/ResourceManager.cpp, src/Layers/xrRender/ResourceManager_Resources.cpp, src/Layers/xrRenderPC_R4/r4.h, src/Layers/xrRenderPC_R3/r3.h, src/Layers/xrRenderPC_R4/blender_blur.h, src/Layers/xrRenderPC_R4/blender_dof.cpp]
related: [engine/_overview.md, flows/render-frame.md, flows/renderer-resources-shaders.md, flows/renderer-blender-compile.md, flows/renderer-scene-graph.md, flows/renderer-visibility.md, flows/renderer-backend.md, flows/renderer-visual-types.md, flows/renderer-skeleton.md, flows/renderer-lighting.md, flows/renderer-sun-shadows.md, flows/renderer-wallmark-engine.md, flows/renderer-particles.md, flows/renderer-environment.md, flows/renderer-detail-grass.md, flows/renderer-hw-device.md, flows/renderer-factory-bridges.md, flows/renderer-r1-path.md, flows/renderer-r2-r3-paths.md, flows/renderer-3dfluid.md, flows/renderer-post-process.md, flows/engine-boot-frame.md, features/spom.md, modlayer/assets.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Renderer (Layers/)

The render system: a shared layer, per-generation render paths (R1–R4), and D3D backends. The
ordered **R4/DX11 per-frame pass pipeline** (cull → G-buffer → shadows → light accumulation →
combine/post → forward → present) is the flow **`flows/render-frame.md`**. Structure,
the blender concept + its `Compile`/pass pattern, and shader loading are verified below; the
per-blender *pass behavior* is shown by one worked example (`blender_dof`), with the rest left to
read on demand via the same pattern.

**Subsystem deep-dives** (this map delegates to verified flows) — the full data path is now covered:
resource manager + shader/texture load & dedup tree → `flows/renderer-resources-shaders.md`; how a
blender emits passes (CLSID factory + `CBlender_Compile` recorder) → `flows/renderer-blender-compile.md`;
visibility/culling (HOM + sectors/portals + occlusion queries) → `flows/renderer-visibility.md`; the
per-frame scene-graph (sorted render queue, per-view `GMBase`/`GMLight`/`GMCascade` instances that also
build shadow queues) → `flows/renderer-scene-graph.md`; the renderable visual types + model pool
(`dxRender_Visual` tree, two-tier instancing) → `flows/renderer-visual-types.md`; skeleton/kinematics
(bone calc, animation blending, skinning) → `flows/renderer-skeleton.md`; dynamic point/spot light
accumulation + shadow maps → `flows/renderer-lighting.md`; sun + cascaded shadow maps →
`flows/renderer-sun-shadows.md`; the wallmark/decal engine → `flows/renderer-wallmark-engine.md`; particle rendering →
`flows/renderer-particles.md`; environment (sky/clouds/rain/thunder/flares/gamma) →
`flows/renderer-environment.md`; detail/grass → `flows/renderer-detail-grass.md`; the D3D11
device/swapchain/caps → `flows/renderer-hw-device.md`; the engine↔Layers seam (render factory + UI/font/
console/debug bridges) → `flows/renderer-factory-bridges.md`; the draw-submission backend
(`CBackend`/`RCache` state caching) → `flows/renderer-backend.md`; the R4+SSFX post-processing chain
(`phase_combine` ordered stages) → `flows/renderer-post-process.md`. The legacy **R1** forward/light-map
path (not this build) is `flows/renderer-r1-path.md`; the **R2/R3** deferred generations are
`flows/renderer-r2-r3-paths.md`; the dormant **DX10 3D fluid** smoke sim is `flows/renderer-3dfluid.md`
(all reference/contrast — the active build is R4).

## Structure (verified by listing + headers)

- **xrRender/** — shared code used by all paths (blender base classes, resource manager).
- **xrRenderPC_R1 / R2 / R3 / R4/** — per-generation paths. Each defines a `CRender` implementing
  `IRender_interface` (verified: `class CRender : public IRender_interface` (`r4.h`)).
- **xrRenderDX9 / xrRenderDX10/** — D3D backends. `xrRenderDX10` mixes D3D10 and D3D11 types
  (`DXCommonTypes.h` typedefs both), i.e. it backs both R3 and R4.
- **Texture lifetime** is owned by `xrRender/ResourceManager.cpp` + the DX10/11 `CTexture`: besides the
  level-unload sweep, the MT build can **evict idle textures mid-session** (opt-in
  `r__tex_evict_enabled`, plus `_age_frames` / `_batch_size` / `_interval`, called once every N frames
  from `CRender::Render` in R4). Details and the async re-load contract:
  `flows/renderer-resources-shaders.md`.
- **xrAPI/** — holds the global render interface the engine renders through: `xrAPI.cpp` declares
  `IRender_interface* Render` and `dxRenderFactory* RenderFactory`. The active path is chosen by
  `CEngineAPI` (`src/xrEngine/EngineAPI.cpp`): in this all-in-one build the renderer is fixed at
  **compile time** — `#error` guards enforce exactly one of R1/R2/R3/R4 — with
  `psDeviceFlags`/`g_current_renderer` set in `InitializeNotDedicated()`; the historical
  `LoadLibrary(rN_name)` DLL loads are commented out, i.e. statically linked.

## Blenders vs shaders (verified)

- A **blender** is a C++ class deriving `IBlender` with a `Compile(CBlender_Compile&)` method that
  configures a render pass — verified: `CBlender_blur : public IBlender` (`blender_blur.h`),
  `getComment()=="Blur generation"`. (Not the 3D app.) Blenders are the `blender_*.cpp`/`Blender_*.cpp`
  files.
- **Shaders** (HLSL) are loaded at runtime via `::Render->getShaderPath()` prepended to the shader
  name + extension — verified in `ResourceManager_Resources.cpp` (`.ps`), `ShaderResourceTraits.h`
  (generic, `GetShaderExt()`), `CSCompiler.cpp` (`.cs`).
- **R3 and R4 both load from the `r3\` folder** — verified: `getShaderPath()` returns `"r3\\"` in
  **both** `r3.h` and `r4.h`. R4 also explicitly probes `r3\screenspace_common.h`
  (`r4.cpp`). On disk: `gamedata/shaders/{r1,r2,r3}` (verified by listing — no `r4` folder).
- Extensions: `.vs.ps.gs` handled as literals in the DX10 resource manager; `.cs` via `CSCompiler`;
  hull/domain shaders exist (`HullShader`/`DomainShader` classes in `xrRenderDX10`). `.h` = shared
  includes; `.s` = script-side blender config.

## DX-version mapping

Verified: R1 is the static path (`FStaticRender.cpp`, and `xrCPU_Pipe` does CPU point-light *vertex*
lighting — `engine/subsystems-misc.md` — consistent with R1's static/vertex model); R2 includes
`<d3dx9.h>` (`xrRenderPC_R2/stdafx.h`) → DX9; R3 has DX10 MSAA modes (`r3.h`); R4 has DX10/DX11
modes (`r4.h`); `xrRenderDX10` backs DX10+DX11. The R4 **deferred dynamic-lighting passes are now
verified** in `flows/renderer-lighting.md` (point/spot accumulation + shadow maps) and
`flows/renderer-sun-shadows.md` (sun cascades); the post chain in `flows/renderer-post-process.md`. The
legacy R2 (DX9) and R3 (DX10) deferred generations — same deferred *shape*, deltas-vs-R4 — are
`flows/renderer-r2-r3-paths.md`.

## R4 feature files & the blender pattern (mechanism verified)

R4 contains (files confirmed by listing): deferred shading (`blender_deffer_flat/model/aref`),
lighting (`blender_light_*`), post (`blender_bloom_build`, `blender_blur`, `blender_dof`,
`blender_combine`, HDR10 `blender_hdr10_*`), gasmask effects, sun effects, compute shaders
(`ComputeShader.*`, `CSCompiler.*`), shadow-map allocation (`SMAP_Allocator.h`), `R_Backend_LOD.*`.

**Blender pattern verified** (read `blender_dof.cpp`): each `CBlender_*::Compile(CBlender_Compile& C)`
calls `IBlender::Compile(C)`, then `switch (C.iElement)` — one case per **pass**. A pass does
`C.r_Pass(vertex_stub, pixel_shader, …)`, binds inputs with `C.r_dx10Texture(slot, r2_RT_*)` /
`C.r_dx10Sampler(name)`, then `C.r_End()`. Example — `CBlender_dof`: element 0 runs the
`depth_of_field` pixel shader reading position (`r2_RT_P`), scene image (`r2_RT_generic0`) and the
half-res blur RT (`r2_RT_blur_2`); element 1 composites it back via `post_processing`. The full
blender→pass compile mechanism (CLSID factory + the `CBlender_Compile` recorder) is
`flows/renderer-blender-compile.md`; the lighting/post blenders are exercised by
`flows/renderer-lighting.md`/`-sun-shadows.md`/`-post-process.md`. *(Individual per-blender `Compile`
bodies beyond `blender_dof` are still read on demand via that same pattern.)*

## SPOM (branch-specific — not on this branch)

SPOM is on the `spom` branch, not here (no `spom`/`silhouette` in `src/Layers` on this branch). See
`features/spom.md`.

## Deeper

- Shared + resource mgr: `src/Layers/xrRender/ResourceManager_Resources.cpp`, `ShaderResourceTraits.h`
- R4 path: `src/Layers/xrRenderPC_R4/` (`r4.h`/`r4.cpp`, `blender_*.cpp`, `ComputeShader.*`,
  `CSCompiler.*`)
- D3D backend: `src/Layers/xrRenderDX10/` (`DXCommonTypes.h`)
- Global interface: `src/Layers/xrAPI/xrAPI.cpp`
- HLSL on disk: `gamedata/shaders/r3/` (DX10/DX11), `r2/`, `r1/`

## Maintenance

- `getShaderPath()` return is the source of truth for which on-disk folder a path uses (`r3.h`/`r4.h`).
- Promote an *(INFERRED)* pass description to verified only after reading the blender's `Compile`.
- If a `r4` shader folder or new path appears, re-check `getShaderPath()` and update here + `GLOSSARY.md#r1-r4`.
