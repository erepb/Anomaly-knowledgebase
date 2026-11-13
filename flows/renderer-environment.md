---
domain: flows/renderer-environment
tags: [engine, renderer, r4, dx11, flow]
sources: [src/xrEngine/Environment_misc.cpp, src/xrEngine/Rain.cpp, src/xrEngine/thunderbolt.cpp, docs/env_mods.md, gamedata/configs/environment/env_mods/_template.ltx, src/Layers/xrRender/dxEnvironmentRender.cpp, src/Layers/xrRender/dxEnvironmentRender.h, src/Layers/xrRender/dxRainRender.cpp, src/Layers/xrRender/dxThunderboltRender.cpp, src/Layers/xrRender/dxLensFlareRender.cpp, src/Layers/xrRender/xr_effgamma.cpp, src/Layers/xrRender/xr_effgamma.h, src/Layers/xrRenderPC_R4/r4_R_rain.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_draw_rain.cpp]
related: [flows/renderer-post-process.md, flows/renderer-backend.md, flows/renderer-resources-shaders.md, flows/renderer-factory-bridges.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: environment rendering (sky / clouds / rain / thunder / flares / gamma)

How the renderer draws the atmospheric layer: sky dome + clouds, rain (streaks + R4 wet-surface), lightning
bolts, sun lens flares, and the output gamma/color correction. The environment **state** (time of day,
weather blend, densities) is game-side (`g_pGamePersistent->Environment()`); the **draw** is the Layers
`dx*Render` bridges, called by the post chain (`flows/renderer-post-process.md`) at fixed points.

**Provenance:** researched with a read-only agent, then spot-verified against code. The bridge classes
(`dxEnvironmentRender:IEnvironmentRender` with `RenderSky`/`RenderClouds`; `dxRainRender:IRainRender`;
`dxThunderboltRender:IThunderboltRender`; `dxLensFlareRender:ILensFlareRender`), the gamma path
(`CGammaControl::Update` → `SetGammaControl` on DX10/11, `SetGammaRamp` on DX9), and the R4 wet-surface
entry (`CRender::render_rain`) are personally code-verified. The geometry fills (sky dome, cloud plane,
rain streaks, bolt mesh, flare quads) and the LUT math are *(agent-traced — not personally re-read)*.

## Trigger / entry

The post chain `phase_combine` (`flows/renderer-post-process.md`) calls, in order:
`Environment().RenderSky()` + `RenderClouds()` (early, into the scene buffer) → `RenderLast()` (rain +
thunder, after water) → `RenderFlares()` (late, near the end). Gamma is applied at the display via the
swapchain output, separate from the frame draws.

## Sequence (verified structure)

### Game owns state, Layers draws

The game-side `CEnvironment` holds the blended weather descriptor (sky rotation/colors, cloud colors +
mesh, fog, rain density/wetness, lens-flare controller). It renders through a `FactoryPtr<IEnvironmentRender>`
(and sibling rain/thunder/flare render pointers), whose concrete impls are the Layers `dx*Render` classes
(created via `flows/renderer-factory-bridges.md`). The bridges are passive: they read the descriptor at
render time and emit geometry to `RCache`.

### Sky / clouds (`dxEnvironmentRender`)

`RenderSky(env, …)` draws a camera-locked sky-dome half-box with **two blended sky textures** (for
time-of-day transition, blend weight in vertex alpha), rotated by `sky_rotation`, then renders the sun
flare overlay (`eff_LensFlare->Render`). `RenderClouds(env)` draws a planar cloud layer with wind baked
into vertex colors and two blended cloud textures. Both lock dynamic geometry and submit via `RCache`.
*(geometry fills agent-traced.)*

### Rain (two layers)

- **Streaks** — `dxRainRender::Render` draws camera-following rain streak quads (count scaled by
  `rain_density`) plus splash particles from a detail model; frustum-culled.
- **R4 wet-surface** — `CRender::render_rain` renders a downward rain-light shadow depth pass and builds
  a wet-surface cull frustum; `CRenderTarget::draw_rain` then applies gloss/normal/wetness to lit
  surfaces (stencil-masked, deferred), and the SSFX `phase_ssfx_rain` adds a refraction buffer when
  enabled. *(bodies agent-traced.)*

### Thunder / lens flare

`dxThunderboltRender::Render` draws the lightning bolt detail-model mesh + two camera-facing gradient
flashes, opacity driven by the bolt's lightning phase. `dxLensFlareRender::Render` draws the sun disk +
halo flares along the light→center axis + an optional full-screen gradient, faded by an occlusion blend.
*(bodies agent-traced.)*

### Gamma / color correction (`CGammaControl`)

`CGammaControl::Update` builds a per-channel LUT from `fGamma`/`fBrightness`/`fContrast`/`cBalance` and
applies it at the **display hardware** level: DX10/11 via `IDXGIOutput::SetGammaControl`, DX9 via
`SetGammaRamp` — i.e. a post-swapchain output ramp, not an in-frame shader LUT (that's `phase_lut` in
`flows/renderer-post-process.md`). *(LUT gen agent-traced.)*

## Function digests (spine)

- **`dxEnvironmentRender.cpp:dxEnvironmentRender::RenderSky` / `RenderClouds`** — *Does:* draw the
  camera-locked sky dome (two blended textures + sun flare) / cloud plane (wind in vertex color).
  *Reads:* `env.CurrentEnv` (rotation/colors/weight). *Calls:* `RCache.*`, `eff_LensFlare->Render`.
  *Called by:* `phase_combine`. *(verified entries.)*
- **`dxRainRender.cpp:dxRainRender::Render`** — *Does:* camera-following rain streaks + splashes.
  *Reads:* `rain_density`. *Called by:* `Environment().RenderLast()`. *(verified entry.)*
- **`r4_R_rain.cpp:CRender::render_rain`** — *Does:* rain-light shadow pass + wet-surface cull frustum
  (R4 deferred wet surfaces). *Calls:* `draw_rain`, `phase_smap_direct`. *(verified entry; body
  agent-traced.)*
- **`dxThunderboltRender.cpp:Render` / `dxLensFlareRender.cpp:Render`** — *Does:* lightning bolt + flash
  / sun disk + halos. *Called by:* `RenderLast` / `RenderFlares`. *(verified entries.)*
- **`xr_effgamma.cpp:CGammaControl::Update`** — *Does:* build + apply the output gamma LUT
  (`SetGammaControl` DX10/11 / `SetGammaRamp` DX9). *(verified entry; LUT math agent-traced.)*

## Isolation ledger (the seams)

- **Game-data seam:** `g_pGamePersistent->Environment()` (`CEnvironment` weather state + the lens-flare
  controller) — owned game-side, read-only to the bridges. Game owns logic/blending; Layers owns draws.
- **Frame seam:** `phase_combine` (`flows/renderer-post-process.md`) calls `RenderSky`/`RenderClouds`
  (early) → `RenderLast` (rain/thunder) → `RenderFlares` (late). `render_rain` runs in its own R4 pass.
- **Bridge seam:** the `IEnvironmentRender`/`IRainRender`/`IThunderboltRender`/`ILensFlareRender`
  interfaces ↔ the `dx*Render` impls created by `flows/renderer-factory-bridges.md`.
- **Backend seam:** all draws stream geometry to `RCache` (`flows/renderer-backend.md`); shaders/textures
  from `flows/renderer-resources-shaders.md` (sky/cloud/rain/flare shaders).
- **SSFX link:** `phase_ssfx_rain` + the wet-surface gloss tie into the post chain
  (`flows/renderer-post-process.md`).
- **Gamma seam:** applied at the swapchain output (`CHW`/`flows/renderer-hw-device.md`), distinct from
  the in-frame `phase_lut`.
- **Config:** rain density/scale cvars, wet-surface distance/quality, flare flags (`ps_r2_anomaly_flags`),
  heat-vision (disables flares), gamma/brightness/contrast.

## Game-side environment state changes (2026-09 → 2026-11)

Not renderer code, but they change what the environment draws:

- **Env modifiers from LTX** — `CEnvironment::mods_reload` (`xrEngine/Environment_misc.cpp`) first
  tries `CEnvironment::mods_load_ltx`: `configs\environment\env_mods\<level>.ltx` replaces the binary
  `level.env_mod` entirely (an empty .ltx disables the level's modifiers). Console `env_mods_reload`.
  Format, blending math and `use_flags`: `docs/env_mods.md`; copyable section:
  `env_mods/_template.ltx`.
- **Rain wetness is frame-rate independent** — `Rain.cpp` scales wet/dry rate by `Device.fTimeDelta`
  over `WETNESS_CHANGE_TIME` (166 s at multiplier 1; `ssfx_wetness_multiplier` x = wet, y = dry).
  `level.set_rain_wetness(0..1)` writes `Environment().wetness_factor` directly.
- **Rain loop volume hook** — sampled once at rain onset via `IGame_Persistent::OnRainSound` (see
  `flows/sound-music.md`).
- **Thunderbolt flash no longer stacks while paused** — `thunderbolt.cpp` keeps base/flash colours
  separately (`flash_applied`), so the flash is applied over the base once rather than accumulating.

## Threading

Main render thread, within the frame (env draws inside `phase_combine`; gamma applied at output).

## Failure & edge cases

- **Gamma is an output ramp**, not the in-frame LUT — `phase_lut` (color grading) and `CGammaControl`
  (display gamma) are two different things; confusing them mis-diagnoses brightness issues.
- **Flares gated** by `ps_r2_anomaly_flags` (R2_AN_FLAG_FLARES) and disabled under heat-vision.
- **Rain has two layers** — streaks (`dxRainRender`) and wet surfaces (`render_rain`/`draw_rain`);
  "no wet surfaces but rain falls" means the R4 wet pass is off/unsupported.
- **Sky is two-texture blended** — a time-of-day transition seam is a descriptor-blend (weight) issue.

## Deeper

- Sky/clouds: `xrRender/dxEnvironmentRender.cpp`/`.h` (`RenderSky`/`RenderClouds`/`OnFrame`)
- Rain: `xrRender/dxRainRender.cpp` (streaks); `xrRenderPC_R4/r4_R_rain.cpp` (`render_rain`),
  `r4_rendertarget_draw_rain.cpp` (`draw_rain`)
- Thunder/flare: `xrRender/dxThunderboltRender.cpp`, `dxLensFlareRender.cpp`
- Gamma: `xrRender/xr_effgamma.cpp`/`.h` (`CGammaControl::Update`/`GenLUT`)
- Frame placement: `flows/renderer-post-process.md`; bridges: `flows/renderer-factory-bridges.md`;
  backend: `flows/renderer-backend.md`

## Maintenance

- The `Render*` bridge entries + `CGammaControl::Update` are the spine — re-grep if the env bridges or
  gamma path change.
- Promote the *(agent-traced)* geometry fills (sky dome, clouds, rain streaks, bolt/flare quads) +
  `draw_rain` + LUT math to verified when an environment/weather task needs exact behavior.
- Keep "game owns weather state, Layers draws; gamma is an output ramp, not phase_lut."
