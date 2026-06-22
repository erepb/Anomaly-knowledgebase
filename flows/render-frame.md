---
domain: flows/render-frame
tags: [engine, renderer, r4, dx11, deferred, shaders, flow]
sources: [src/Layers/xrRenderPC_R4/r4_R_render.cpp, src/Layers/xrRenderPC_R4/r4.h, src/Layers/xrRenderPC_R4/r4_rendertarget.h, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_scene.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_combine.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_accum_direct.cpp, src/Layers/xrRender/lights_render.cpp, src/Layers/xrRender/R_sun.cpp]
related: [flows/engine-boot-frame.md, engine/renderer.md, features/spom.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: R4 (DX11) render frame

The deferred render pipeline for the R4 (DX11) backend: cull → G-buffer → shadows → light
accumulation → combine/post → forward → present. This is the render step the engine frame loop calls
(`flows/engine-boot-frame.md` → `seqRender` → backend `CRender::Render`). It resolves the "R2+
deferred dynamic-lighting *passes* left inferred" gap noted in `engine/renderer.md` (which has the
`CRender`/`IBlender`/shader-path/blender-Compile mechanism).

**Provenance:** researched with a read-only agent, then spot-verified against code. The entry point
(`CRender::Render` in `r4_R_render.cpp`) and the phase homes (`phase_scene_begin`/`phase_combine`/
`phase_combine_volumetric` in `r4_rendertarget*`, `render_lights` in `r4.h`) are personally
code-verified. The **ordered pass list, render-target roles, and the SSFX post stages are
*(agent-traced — not personally re-read)*** — this is a large, heavily-modified (Anomaly + SSFX) R4;
treat the pass *order* as a working map to verify per-task, not a line-checked guarantee.

## Entry (verified)

`CRender::Render()` (`r4_R_render.cpp`, verified) is the top-level per-frame render invoked by the
device via the renderer interface. It runs the deferred pipeline below, then returns so the device's
`End` presents (`flows/engine-boot-frame.md`).

## Pass sequence (agent-traced order; phase homes verified)

1. **Cull / scene-graph capture** — hierarchical occlusion (HOM) + sector/portal visibility collect
   visible static/dynamic geometry and the light lists into the render graph (`r_dsgraph_*`).
2. **G-buffer fill** — `phase_scene_begin` (verified) binds the MRT set and renders opaque geometry
   into the G-buffer (position, color/spec, plus motion vectors / heat in this build), marking lit
   pixels in stencil; `phase_scene_end` closes it. Wallmarks follow.
3. **Occlusion queries** (`phase_occq`) — test light-volume visibility against the depth/stencil.
4. **Sun shadow cascades** — `render_sun_cascades`/`R_sun.cpp` render depth-only shadow maps per
   cascade (near/mid/far).
5. **Light accumulation** — `accum_direct` (`r4_rendertarget_accum_direct.cpp`) does the directional
   (sun) full-screen pass sampling the cascades into the accumulator; **`render_lights`** (verified,
   `lights_render.cpp`) accumulates dynamic point/spot lights (rendering a per-light shadow map when
   shadowed) into the accumulator.
6. **Combine + post** — **`phase_combine`** (verified) resolves the deferred result (G-buffer +
   accumulator) and runs the post chain: SSAO/SSR/water, bloom, motion blur, DOF, tonemap/LUT, AA
   (SMAA/TAA), with `phase_combine_volumetric` (verified present) for volumetric lighting. Many of
   these are SSFX (`o.ssfx_*`) stages in this build.
7. **Forward / blended** — particles, glass, sorted/translucent geometry, environment (rain, bolts)
   render forward over the combined image.
8. **HUD / UI**, then **present** (the device's swap-chain present at `End`).

So R4 is **deferred**: opaque → G-buffer, then lighting is screen-space accumulation from the
G-buffer + shadow maps, then post, then a forward pass for things deferred can't do (blending,
particles), then present.

## Isolation ledger (the seams)

- **Render targets** (`r4_rendertarget.h`): a G-buffer set (position, color/specular, plus
  motion-vector and heat targets in this build), an HDR **accumulator** for lighting, generic LDR
  working targets for post, shadow-map depth, bloom/SSAO/SSR/luminance auxiliaries. Each phase reads
  some and writes others (e.g. scene writes G-buffer; accum writes the accumulator; combine reads
  both → generic). *(exact RT set/formats agent-traced; the header is the source of truth.)*
- **Blender / shader seam:** phases pick shaders via the `IBlender`/`ShaderElement` mechanism and the
  shared shader path (`engine/renderer.md`: `getShaderPath()` = `r3\\`, R3+R4 share shaders);
  full-screen passes use dedicated `s_*` elements (combine, accum_direct/point/spot, ssao, bloom,
  smaa…), with MSAA variants.
- **Scene-graph seam:** geometry is collected/sorted in the render graph (`r_dsgraph_*` capture →
  phased render: static/dynamic, emissive, sorted, distort, HUD).
- **Config / flags:** `ps_r2_ls_flags` bits toggle passes (sun, DOF, AA, terrain prepass, split
  scene…) and `o.ssfx_*` options toggle the SSFX stages (bloom, SSR, water, volumetric, motion blur,
  TAA). *(specific flag names agent-traced.)*
- **Links to other flows:** called by the engine frame loop (`flows/engine-boot-frame.md`, the
  `seqRender` step). SPOM (silhouette POM) hooks into the geometry/lighting passes — see
  `features/spom.md` (**branch-specific: only on the `spom` branch, not this working branch**; don't
  trust its code pointers unless checked out there).

## Threading

The render submission runs on the main thread inside `on_idle` (after the secondary task group is
joined); culling/bone prep happen in the secondary tasks earlier in the frame
(`flows/engine-boot-frame.md`).

## Failure & edge cases

- **Deferred ⇒ transparency is a separate forward pass** — glass/particles/water aren't in the
  G-buffer; a "missing in reflections/SSAO" bug is usually because the object is forward-rendered.
- **Heavily SSFX-modified:** this R4 adds many screen-space passes (`o.ssfx_*`); the vanilla R4 pass
  list differs. Verify a specific pass against `r4_rendertarget_phase_combine.cpp` before relying on
  it.
- **Shaders are shared R3/R4** (`r3\\` path) — a shader change affects both backends.
- **MSAA has per-sample shader variants** — lighting/combine branch on the MSAA path.

## Deeper

- Entry + main passes: `src/Layers/xrRenderPC_R4/r4_R_render.cpp` (`CRender::Render`),
  `src/Layers/xrRenderPC_R4/r4.h` (`render_lights`, members)
- Render targets + phases: `src/Layers/xrRenderPC_R4/r4_rendertarget.h`,
  `r4_rendertarget_phase_scene.cpp` (`phase_scene_begin`/`_end`),
  `r4_rendertarget_phase_combine.cpp` (`phase_combine`, `phase_combine_volumetric`),
  `r4_rendertarget_accum_direct.cpp` (`accum_direct`)
- Lighting/shadows: `src/Layers/xrRender/lights_render.cpp` (`render_lights`),
  `src/Layers/xrRender/R_sun.cpp` (sun cascades)
- Renderer mechanism (blender/shader/path): `engine/renderer.md`; SPOM: `features/spom.md`

## Maintenance

- Re-grep `CRender::Render` (`r4_R_render.cpp`) and the `phase_*` homes in `r4_rendertarget.h` if the
  pipeline is restructured.
- Promote the *(agent-traced)* pass order, RT set, and SSFX stage list to verified by reading
  `r4_R_render.cpp` + `r4_rendertarget_phase_combine.cpp` end-to-end when a render task needs exact
  ordering — and update `engine/renderer.md` (which still marks the deferred passes inferred) in the
  same pass.
- Keep the "deferred + separate forward pass + SSFX-modified" caveats — they prevent wrong
  assumptions about what's in the G-buffer.
