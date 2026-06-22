---
domain: flows/renderer-post-process
tags: [engine, renderer, r4, dx11, shaders, flow]
sources: [src/Layers/xrRenderPC_R4/r4_rendertarget_phase_combine.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_PP.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_bloom.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_ssao.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_luminance.cpp, src/Layers/xrRender/rendertarget_phase_blur.cpp, src/Layers/xrRender/rendertarget_phase_dof.cpp, src/Layers/xrRender/rendertarget_phase_lut.cpp, src/Layers/xrRender/rendertarget_phase_smaa.cpp, src/Layers/xrRender/rendertarget_phase_nightvision.cpp, src/Layers/xrRender/rendertarget_phase_sunshafts.cpp]
related: [flows/render-frame.md, flows/renderer-scene-graph.md, engine/renderer.md, features/spom.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: renderer post-processing chain (R4 + SSFX)

The ordered screen-space chain that runs after deferred lighting is resolved: SSAO, the deferred
combine, sky, SSR/water, forward geometry, volumetrics, bloom, sunshafts, fog, motion blur, blur,
DOF, LUT, gasmask, night/heat-vision, SMAA, TAA, HDR10, and the final post pass. This is the
expansion of `flows/render-frame.md` step 6 ("combine + post"); this build is heavily **SSFX**-modified
(Screen Space Shaders), so the chain differs substantially from vanilla R4.

**Provenance:** researched with a read-only agent, then spot-verified against code. The **entry**
(`CRenderTarget::phase_combine`, called from `CRender::Render`) and the **ordered stage sequence** are
personally code-verified by reading the actual call order in `r4_rendertarget_phase_combine.cpp`
(the in-source `// [SSFX]` comments mark the SSFX additions). The per-stage RT names, the internal
pass counts, and the SKIMMED stages are *(agent-traced — not personally re-read)*.

## Trigger / entry

`CRender::Render` (`r4_R_render.cpp`), after light accumulation, calls **`Target->phase_combine()`**
(`r4_rendertarget_phase_combine.cpp:CRenderTarget::phase_combine`). That single function drives the
whole chain below, each stage guarded by a renderer option (`RImplementation.o.*`) and/or a console
var (`ps_*`). The whole-frame map is `flows/render-frame.md`.

## Sequence (verified order, in `phase_combine`)

1. **SSAO** — one of `phase_hdao()` (if `o.ssao_hdao && o.ssao_ultra && ps_r_ssao>0`),
   `phase_downsamp()` (if `o.ssao_opt_data`), or `phase_ssao()` (if `o.ssao_blur_on`).
2. **Motion-vector setup** — save previous/current view-proj matrices + camera position (skipped on
   second-viewport frames, `Device.m_SecondViewport.IsSVPFrame()`).
3. **`phase_ssfx_ao()`** *[SSFX]* and **`phase_ssfx_il()`** *[SSFX]* — extra AO and indirect light.
4. **Deferred combine** — full-screen `s_combine` pass resolving G-buffer + accumulator into the
   generic buffer (ambient/hemisphere/fog/sun params).
5. **Sky + clouds** — `Environment().RenderSky()` / `RenderClouds()`.
6. **`phase_ssfx_ssr()`** *[SSFX]* — screen-space reflections (skipped on SVP frames).
7. **SSFX water** *[SSFX]* — `phase_ssfx_water_blur()` + `phase_ssfx_water_waves()`, then
   `GMBase.r_dsgraph_render_water()` (the water geometry from `flows/renderer-scene-graph.md`).
8. **`phase_ssfx_rain()`** *[SSFX]*, then **`Environment().RenderLast()`** (rain/thunderbolts).
9. **`RImplementation.render_forward()`** — forward-shaded geometry (particles, glass, hair) over the
   combined image (+ heat RT).
10. **`phase_combine_volumetric()`** — volumetric light combine (SSFX or active spot/point volumetrics).
11. **`phase_bloom()`** (invalidates the HDR RT; internally drives `phase_luminance` exposure) — but
    early-returns if `o.ssfx_bloom`, deferring to step 16.
12. **`phase_sunshafts()`** (if `ps_sunshafts_mode` is screen-space/combine and not menu PP).
13. **`phase_ssfx_fog_scattering()`** *[SSFX]*, **`phase_ssfx_motion_blur()`** *[SSFX]*.
14. **`phase_blur()`** — auxiliary half/quarter-res blur buffers (SVP-skipped).
15. **`phase_ssfx_bloom()`** *[SSFX]* **or** **`phase_pp_bloom()`** (vanilla) — mutually exclusive.
16. **`phase_dof()`** (if `R2FLAG_DOF`), **`phase_lut()`** (always — color grading).
17. **Gasmask** — `phase_gasmask_dudv()` then `phase_gasmask_drops()` (if mask/drops control on).
18. **`phase_nightvision()` / `phase_heatvision()` / `phase_fakescope()`** (vision modes / fake scope).
19. **`phase_smaa()`** (if `ps_smaa_quality`), **`phase_ssfx_taa()`** *[SSFX]*.
20. **Final combine 2** — second `s_combine` element (AA/distort variants) applying tonemap + motion
    blur + AA into the LDR color RT (or backbuffer).
21. **`phase_hdr10_bloom()` / `phase_hdr10_lens_flare()`** (if `o.dx11_hdr10`), **`RenderFlares()`**.
22. **`phase_pp()`** (`r4_rendertarget_phase_PP.cpp`) — final post (blur/grayscale/noise/color map)
    into the backbuffer, if `PP_Complex`.

After `phase_combine` returns, the device presents (`flows/engine-boot-frame.md`).

## SSFX vs vanilla (verified distinction)

The in-source `// [SSFX]` markers identify additions absent from vanilla R4: `phase_ssfx_ao`,
`_ssfx_il`, `_ssfx_ssr`, `_ssfx_water_*`, `_ssfx_rain`, `_ssfx_fog_scattering`, `_ssfx_motion_blur`,
`_ssfx_bloom`, `_ssfx_taa`. Each is gated by an `RImplementation.o.ssfx_*` option plus a `ps_ssfx_*`
console var. Vanilla bloom is `phase_bloom`→`phase_pp_bloom`; with `o.ssfx_bloom` set, `phase_bloom`
early-returns and `phase_ssfx_bloom` runs instead. **When documenting/modifying a specific stage,
check its guard — many stages are entirely conditional.**

## Render-target ping-pong *(agent-traced RT names)*

Stages ping-pong between generic LDR buffers (`rt_Generic_0`/`rt_Generic_1`, MSAA `_r` variants), with
auxiliaries for bloom (`rt_Bloom_*`), luminance/exposure pyramid (`rt_LUM_*` — swapped each frame to
avoid GPU sync stalls), SSAO (`rt_ssao_temp`), blur (`rt_blur_*`), SMAA (`rt_smaa_edgetex`/`blendtex`),
heat (`rt_Heat`), and the SSFX scratch/history targets (`rt_ssfx_*`, incl. `rt_ssfx_prevPos` written at
chain end when any motion-dependent stage requested it). Final LDR lands in `rt_Color` → backbuffer
via `phase_pp`. *(The exact RT member set is in `r4_rendertarget.h` — the source of truth; verify a
specific RT there before relying on it.)*

## Function digests (spine)

- **`r4_rendertarget_phase_combine.cpp:CRenderTarget::phase_combine`** — *Does:* drives the entire
  ordered post chain above; the deferred combine + final combine passes are inline, the rest are
  `phase_*` calls. *Reads:* `RImplementation.o.*`, `ps_*`, G-buffer + accumulator RTs. *Writes:* the
  generic/color RTs. *Called by:* `r4_R_render.cpp:CRender::Render`. *Gotcha:* order is
  load-bearing (luminance before tonemap, bloom before DOF/AA, SSR before water).
- **`r4_rendertarget_phase_bloom.cpp:phase_bloom` / `phase_ssfx_bloom`** — *Does:* bright-pass +
  downsample/blur pyramid; vanilla drives exposure via `phase_luminance`. *Gotcha:* `phase_bloom`
  early-returns under `o.ssfx_bloom`.
- **`r4_rendertarget_phase_ssao.cpp:phase_ssao` / `phase_ssfx_ao` / `phase_ssfx_il`** — *Does:* AO /
  SSFX-AO / SSFX indirect light at reduced res with multi-pass blur.
- **`r4_rendertarget_phase_PP.cpp:phase_pp`** — *Does:* final post (blur/grayscale/noise/colormap)
  `rt_Color` → backbuffer, only when `PP_Complex`.
- Shared post stages (`xrRender/rendertarget_phase_*.cpp`): `phase_blur`, `phase_dof`, `phase_lut`,
  `phase_smaa`, `phase_nightvision`/`phase_fakescope`, `phase_sunshafts`, gasmask — each a guarded
  full-screen pass selecting a blender element. *(SKIMMED — bodies agent-traced.)*

## Isolation ledger (the seams)

- **Driver:** the single `CRenderTarget::phase_combine`; `CRenderTarget` (a.k.a. `Target`) owns all
  render targets + the `s_*` full-screen blenders.
- **Blender/shader seam:** each pass does `RCache.set_Element(s_xxx->E[n])` — blenders compile multiple
  elements (e.g. `s_combine->E[0]` resolve, `E[1..4]` AA/distort/menu variants); MSAA uses parallel
  `*_msaa` blenders. Blenders come from `flows/renderer-resources-shaders.md`.
- **Config seam:** option flags `RImplementation.o.{ssao_*, ssfx_*, dx11_hdr10, …}` (compile/feature
  toggles) + console vars `ps_{r_ssao, sunshafts_mode, smaa_quality, r2_nightvision, ssfx_*, …}` +
  `ps_r2_ls_flags` bits (`R2FLAG_DOF`, AA, sun…). `Device.m_SecondViewport.IsSVPFrame()` skips
  several stages.
- **Backend seam:** `RCache`/`HW.pContext` (DX11 device context) for full-screen draws and
  `CopyResource`/resolves.
- **Game seam:** `g_pGamePersistent->Environment()` for sky/clouds/rain/flares; HUD render follows
  `phase_combine`.
- **Links:** caller `flows/render-frame.md`; water geometry from `flows/renderer-scene-graph.md`; SPOM
  hooks geometry/lighting upstream (`features/spom.md`, **branch-specific**).

## Threading

Main render thread, inside the frame's render submission. The luminance/exposure RT swap is the one
state that intentionally lags a frame to avoid a GPU→CPU readback stall.

## Failure & edge cases

- **Reordering breaks output** — the chain has hard dependencies (exposure→tonemap, bloom→AA, SSR→water).
- **SSFX vs vanilla** — never assume a stage runs; check its `o.ssfx_*`/`ps_*` guard. The vanilla R4
  post list is a strict subset.
- **SVP frames** skip blur/SSFX-bloom/SSR/etc. — second-viewport rendering (e.g. scopes/mirror views)
  takes a reduced chain.
- **MSAA** uses separate RTs + shader variants with resolve steps interleaved.

## Deeper

- Driver + combine + final: `xrRenderPC_R4/r4_rendertarget_phase_combine.cpp`
  (`phase_combine`, `phase_combine_volumetric`)
- R4 stages: `xrRenderPC_R4/r4_rendertarget_phase_{bloom,ssao,luminance,PP,hdr10_bloom,hdr10_lens_flare}.cpp`
- Shared stages: `xrRender/rendertarget_phase_{blur,dof,lut,smaa,nightvision,sunshafts,pp_bloom,gasmask_drops,gasmask_dudv}.cpp`
- RT members: `xrRenderPC_R4/r4_rendertarget.h`; blenders: `xrRenderPC_R4/blender_*.cpp`
  (`flows/renderer-resources-shaders.md`, `engine/renderer.md`)
- Frame context: `flows/render-frame.md`

## Maintenance

- The ordered stage list above is the high-value artifact — re-derive it by re-grepping the
  `phase_*`/`Render*` call sequence inside `phase_combine` if the chain is restructured (the
  `// [SSFX]` comments flag additions).
- Promote the *(agent-traced)* RT names + SKIMMED stage bodies to verified by reading
  `r4_rendertarget.h` + the specific `rendertarget_phase_*.cpp` when a post task needs exact RTs.
- When this lands, promote `engine/renderer.md`'s inferred post passes and add this flow to its
  `related` (done) + `flows/render-frame.md` step 6 cross-link.
