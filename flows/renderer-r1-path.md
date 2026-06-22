---
domain: flows/renderer-r1-path
tags: [engine, renderer, r1, dx9, build, flow]
sources: [src/Layers/xrRenderPC_R1/FStaticRender.cpp, src/Layers/xrRenderPC_R1/FStaticRender.h, src/Layers/xrRenderPC_R1/FStaticRender_RenderTarget.cpp, src/Layers/xrRenderPC_R1/LightProjector.cpp, src/Layers/xrRenderPC_R1/LightProjector.h, src/Layers/xrRenderPC_R1/LightShadows.cpp, src/Layers/xrRenderPC_R1/GlowManager.cpp]
related: [flows/render-frame.md, engine/renderer.md, flows/renderer-backend.md, flows/renderer-scene-graph.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: R1 static / light-map render path (reference / contrast)

The oldest render path (`xrRenderPC_R1/`): forward rendering with **baked static light-maps** and
faked dynamic-object lighting (light projector + blob shadows). **This is NOT the build's active path —
this build is R4 (deferred, `flows/render-frame.md`).** This flow exists for *contrast*: to understand
how the deferred path differs and what "R1" means when a config/log mentions it. Document it deeper
only if a task actually targets R1.

**Provenance:** researched with a read-only agent, then spot-verified against code. The R1 `CRender :
IRender_interface, pureFrame` (`Render`/`Calculate`), `getShaderPath()=="r1\\"`, and the dynamic-light
helpers (`CLightProjector`, `CLightShadows`, `CGlowManager`) are personally code-verified. **Correction
to the agent draft:** R4's `getShaderPath()` is `"r3\\"` (verified in `r4.h`, matches
`engine/renderer.md`), NOT "r4". The R1 render-loop order, projector/shadow internals, and pass details
are *(agent-traced — not personally re-read)*.

## The key contrast (verified)

R1 is **forward + baked light-maps**: most lighting is precomputed into the level (vertex colors /
light-map textures) at authoring time, so there is **no G-buffer and no per-pixel deferred light
accumulation** (the heart of R4, `flows/render-frame.md`). Dynamic, moving objects are lit/shadowed by
two fakes instead of deferred lights:
- **`CLightProjector`** — projects nearby dynamic lights into a per-object texture (a slotted atlas)
  that's sampled when the object renders.
- **`CLightShadows`** — blob/planar shadows for dynamic objects, rendered to a shadow atlas and
  composited.
- **`CGlowManager`** — light coronas/glows with occlusion fade.

This is why R1 runs on old (DX9) hardware. R1 and R4 implement the **same `IRender_interface`** — the
engine above the renderer is path-agnostic; the path is fixed at compile time (`engine/renderer.md`).

## Sequence (R1 render loop — agent-traced; entry verified)

`CRender::Calculate` culls + builds the scene graph (shared `r__dsgraph`/`flows/renderer-scene-graph.md`)
and runs `CLightProjector::calculate` / `CLightShadows::calculate`. `CRender::Render` then forward-renders:
static geometry (with baked light-maps) → detail/grass → sky/clouds → additive dynamic lights →
wallmarks → blob shadows → sorted/translucent → glows → flares/rain. A minimal render target (color +
distortion + projector/shadow atlases) replaces the R4 G-buffer; `FStaticRender_RenderTarget::End` does
the simple post (distortion/color-map/blur). *(loop order agent-traced.)*

## Function digests (spine)

- **`FStaticRender.cpp:CRender::Render`** — *Does:* R1 forward per-frame render (baked-lit static +
  projector-lit dynamic + glows + post). *(verified entry; body agent-traced.)*
- **`FStaticRender.cpp:CRender::Calculate`** — *Does:* cull + scene-graph capture + projector/shadow
  calculate. *(verified entry.)*
- **`FStaticRender.h:CRender::getShaderPath`** — *Does:* returns `"r1\\"` (R1 shaders in
  `gamedata/shaders/r1`). *(verified — contrast: R4 = `"r3\\"`.)*
- **`LightProjector.cpp:CLightProjector::calculate`/`setup`** — *Does:* build/sample the per-object
  projected-light texture. *(verified symbols; body agent-traced.)*
- **`LightShadows.cpp:CLightShadows::calculate`/`render`** — *Does:* render + composite dynamic blob
  shadows. *(verified symbols; body agent-traced.)*
- **`GlowManager.cpp:CGlowManager::Render`** — *Does:* occlusion-tested light coronas. *(agent-traced.)*

## Isolation ledger (the seams)

- **Compile-time selection:** R1 (`xrRenderPC_R1`) vs R4 (`xrRenderPC_R4`) is chosen at build/link
  (`engine/renderer.md`); **this build is R4** — R1 is reference only.
- **Interface seam:** both implement `IRender_interface`; `flows/render-frame.md` is the R4 counterpart
  of `CRender::Render`.
- **Shader seam:** `getShaderPath()=="r1\\"` → `gamedata/shaders/r1` (vs R4's `"r3\\"`).
- **Level-data seam:** R1 relies on baked light-maps in level geometry (R4 has none — runtime deferred).
- **Backend seam:** shared `RCache` (`flows/renderer-backend.md`) and scene graph
  (`flows/renderer-scene-graph.md`).
- **Config:** R1-specific `ps_r1_*` cvars; shared LOD/detail cvars.

## Threading

R1 render runs on the main render thread (it predates the MT secondary-task model used by R4).

## Failure & edge cases

- **Reference-only caveat** — do not trust R1 code pointers for the active build; the live loop is
  `xrRenderPC_R4/r4_R_render.cpp` (`flows/render-frame.md`).
- **No deferred dynamic lighting** — R1 dynamic lighting quality is limited by the projector/blob
  approach; many R4 effects (SSAO/SSR/per-pixel shadows) don't exist in R1.
- **Baked lighting** — changing a light at runtime doesn't relight R1 static geometry (it's baked).

## Deeper

- R1 render: `xrRenderPC_R1/FStaticRender.cpp`/`.h` (`Render`/`Calculate`/`getShaderPath`),
  `FStaticRender_RenderTarget.cpp` (RTs + post)
- Dynamic lighting fakes: `xrRenderPC_R1/LightProjector.cpp`/`.h`, `LightShadows.cpp`, `GlowManager.cpp`
- Contrast: `flows/render-frame.md` (R4 deferred), `engine/renderer.md` (path selection + shader paths)

## Maintenance

- This is a reference flow — keep it short. If a task targets R1, deepen the *(agent-traced)* render
  loop + projector/shadow bodies then.
- Keep the verified facts: R1 `getShaderPath()=="r1\\"`, R4=`"r3\\"`; R1 = forward + baked light-maps +
  projector/blob; the active build is R4.
