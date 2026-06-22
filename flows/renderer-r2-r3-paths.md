---
domain: flows/renderer-r2-r3-paths
tags: [engine, renderer, r2, r3, dx9, dx10, build, flow]
sources: [src/Layers/xrRenderPC_R2/r2.h, src/Layers/xrRenderPC_R2/r2_R_render.cpp, src/Layers/xrRenderPC_R2/stdafx.h, src/Layers/xrRenderPC_R3/r3.h, src/Layers/xrRenderPC_R3/r3_R_render.cpp, src/Layers/xrRenderPC_R3/stdafx.h]
related: [flows/render-frame.md, flows/renderer-lighting.md, flows/renderer-sun-shadows.md, flows/renderer-post-process.md, flows/renderer-r1-path.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: R2 / R3 render generations (comparative — deltas vs R4)

R2 (`xrRenderPC_R2/`) and R3 (`xrRenderPC_R3/`) are the deferred render paths that precede the active
R4 build: R2 on DX9, R3 on DX10. They share R4's deferred architecture and file layout, so the
R4 flows (`flows/render-frame.md`, `-lighting.md`, `-sun-shadows.md`, `-post-process.md`) describe the
**shape** of R2/R3 too. This flow is **comparative/reference** (OOS — the active build is R4); the value
is the deltas. Document deeper only if a task targets R2/R3.

**Provenance:** researched with a read-only agent, then spot-verified against code. The shader paths
(`getShaderPath()` = `"r2\\"` for R2, `"r3\\"` for R3 — and R4 also `"r3\\"`), the **shared generation
constant** (R2, R3, **and** R4 all return `GENERATION_R2` from `get_generation()` — verified), the
backend split (R2 `stdafx.h` includes only `<d3dx9.h>` = DX9; R3 `stdafx.h` adds `<d3d10_1.h>` +
`<d3d11.h>`), and the per-generation `CRender::Render` entries are personally code-verified. The
feature-delta list and per-pass details are *(agent-traced / SKIMMED — expected for a comparative flow)*.

## What R2/R3 are (verified)

Both are **deferred** renderers (G-buffer → light accumulation → combine/post → forward → present) — the
same family as R4, just older backends:
- **R2** — DX9 (Shader Model 3); `getShaderPath()=="r2\\"`; `stdafx.h` includes only `<d3dx9.h>`.
- **R3** — DX10/10.1 (also pulls `<d3d11.h>`); `getShaderPath()=="r3\\"`.
- **R4** — DX10/11; `getShaderPath()=="r3\\"` (shares R3's on-disk shader folder, see
  `engine/renderer.md`).
- All three report `get_generation()==GENERATION_R2` (a lineage constant — *not* an R3/R4-specific
  value; don't infer the active path from it). Selection is **compile-time** (`engine/renderer.md`).

## Structural parallel (verified shape)

`r2_R_render.cpp:CRender::Render` and `r3_R_render.cpp:CRender::Render` follow the same deferred frame as
R4's `flows/render-frame.md`: cull (`GMBase.traverse`) → scene/G-buffer → occlusion → geometry/LODs/HUD
→ wallmarks → (R3: rain + MSAA edge-mark) → sun cascades → emissive → dynamic light accumulation
(`render_lights`) → `phase_combine` post → forward. The `r{2,3}_rendertarget_accum_{point,spot,direct}`
+ `blender_light_*` set mirrors R4's. So the documented R4 flows (`-lighting.md`, `-sun-shadows.md`,
`-post-process.md`) apply to R2/R3 as the same *shape*, with fewer/older passes. *(accum bodies SKIMMED.)*

## Deltas vs R4 (the comparative value — agent-traced)

- **R4 adds (vs R3):** the SSFX screen-space stage suite (`o.ssfx_*` — SSR, water, AO/IL, TAA, motion
  blur, fog, bloom… see `flows/renderer-post-process.md`), DX11 tessellation, HDR10, ultra/HDAO SSAO,
  minmax shadow-map refinements.
- **R3 adds (vs R2):** DX10 geometry shaders, full MSAA path (`o.dx10_msaa*`), minmax shadow maps,
  DX10 rain (`r3_R_rain.cpp`), volumetric fog option.
- **R2 lacks (vs R3):** no geometry shaders, no MSAA, DX9 buffer types, simpler post — the SM3 floor.

So feature richness grows R2 → R3 → R4 on the *same* deferred skeleton; R4's distinctive layer is SSFX +
DX11 (tessellation/compute/HDR10).

## Function digests (comparative entries)

- **`r2_R_render.cpp:CRender::Render` / `r3_R_render.cpp:CRender::Render`** — *Does:* the deferred frame
  for that generation (same shape as `flows/render-frame.md`; R3 adds MSAA edge-mark + rain).
  *(entries verified; bodies agent-traced.)*
- **`r2.h`/`r3.h:CRender::getShaderPath` / `get_generation`** — *Does:* `"r2\\"`/`"r3\\"`; both
  `GENERATION_R2`. *(verified.)*

## Isolation ledger (the seams)

- **Compile-time selection:** R2/R3/R4 are separate projects; one is linked per build. **This build is
  R4** — R2/R3 are reference only (`engine/renderer.md`).
- **Shared interface:** all implement `IRender_interface`; `flows/render-frame.md` is the R4 counterpart.
- **Shared infrastructure:** the scene graph (`flows/renderer-scene-graph.md`), backend
  (`flows/renderer-backend.md`), light DB, and blender system are shared `xrRender/` code; R2/R3/R4
  differ in the per-generation `r*_*` files.
- **Shader paths:** `gamedata/shaders/r2` (R2), `gamedata/shaders/r3` (R3 **and** R4).
- **Config:** option sets grow by generation (R2 base → R3 `dx10_*` → R4 `ssfx_*`/`dx11_*`).

## Threading / failure / edge cases

- Older paths predate some of R4's MT work; treat their threading as simpler.
- **Don't trust R2/R3 code pointers for the active build** — the live frame is R4
  (`xrRenderPC_R4/r4_R_render.cpp`).
- **`GENERATION_R2` is shared** — never use `get_generation()` to tell R2/R3/R4 apart; use the linked
  project / `getShaderPath` / `get_dx_level`.

## Deeper

- R2: `xrRenderPC_R2/r2.h`, `r2_R_render.cpp`, `r2_rendertarget_phase_combine.cpp`, `r2_rendertarget_accum_*`
- R3: `xrRenderPC_R3/r3.h`, `r3_R_render.cpp`, `r3_R_rain.cpp`, `r3_rendertarget_*`
- Active R4 counterparts: `flows/render-frame.md`, `-lighting.md`, `-sun-shadows.md`, `-post-process.md`;
  path selection: `engine/renderer.md`; R1 (forward): `flows/renderer-r1-path.md`

## Maintenance

- Comparative flow — keep it short. If a task targets R2/R3, deepen the *(agent-traced)* deltas + the
  specific `r{2,3}_*` bodies then.
- Keep the verified facts: R2=DX9/`r2\`, R3=DX10/`r3\`, all generations report `GENERATION_R2`, selection
  is compile-time, active build = R4.
