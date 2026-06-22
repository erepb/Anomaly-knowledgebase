---
domain: flows/renderer-sun-shadows
tags: [engine, renderer, r4, dx11, deferred, flow]
sources: [src/Layers/xrRender/R_sun.cpp, src/Layers/xrRender/R_sun_support.cpp, src/Layers/xrRender/R_sun_support.h, src/Layers/xrRender/r_sun_cascades.h, src/Layers/xrRenderPC_R4/r4_rendertarget_accum_direct.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_smap_D.cpp]
related: [flows/renderer-lighting.md, flows/renderer-scene-graph.md, flows/render-frame.md, flows/renderer-blender-compile.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: sun / directional light with cascaded shadow maps

How the sun is split into depth cascades, each cascade's shadow map rendered, and the directional light
accumulated into the deferred lighting buffer. This is the directional counterpart to
`flows/renderer-lighting.md` (F54, point/spot) — they share the lighting accumulator but take separate
code paths. Like dynamic lights, the sun reuses the per-view scene-graph: each cascade owns a
`GMCascade` (a `CDSGraphManager`) to collect shadow casters from the sun's point of view
(`flows/renderer-scene-graph.md`).

**Provenance:** researched with a read-only agent, then spot-verified against code. The cascade model
(`init_cacades` → `m_sun_cascades` with sizes from `ps_ssfx_shadow_cascades.x/y/z` = 20/40/160; the
`cascade` struct with its `GMCascade` at flags `0`), the render entry (`render_sun_cascades` →
`render_sun_cascade` per cascade, called from `r4_R_render.cpp` then `accum_direct_blend`), and the
accumulation entries (`accum_direct`/`accum_direct_cascade`/`accum_direct_f`, `phase_smap_direct`) are
personally code-verified. The orthographic frustum-fit math (`R_sun_support`), texel stabilization,
and the per-cascade pixel-shader sampling are *(agent-traced — not personally re-read)*.

## Trigger / entry

The R4 frame (`flows/render-frame.md`) calls `render_sun_cascades()` during the lighting step (after the
G-buffer), then `Target->accum_direct_blend()` to fold the result into the accumulator.

## Sequence (verified structure)

### Cascade model

`CRender::init_cacades()` builds a fixed set of `m_sun_cascades` (3 in this build), each a `cascade`
struct with: a shadow-map world size (`ps_ssfx_shadow_cascades.x/y/z`, default ~20/40/160 → near/mid/
far), a depth bias, a cull frustum + light view-proj, and **its own `CDSGraphManager GMCascade`**
(flags `0` — no HOM/SSA/fade, like `GMLight`). The sun direction/color comes from
`RImplementation.Lights.sun_adapted` (the environment-adapted sun, `flows/renderer-lighting.md`'s DB).

### Per-cascade render

`render_sun_cascades()` loops cascades calling **`render_sun_cascade(i)`**, which:
1. **Fit** — from the camera view frustum's depth slice for this cascade, build an orthographic light
   view/projection that bounds the cascade's caster volume (`R_sun_support` `FixedConvexVolume`),
   with texel-snapping stabilization to reduce shadow shimmer. *(fit/stabilization agent-traced.)*
2. **Collect** — set phase `PHASE_SMAP`, `cascade.GMCascade.traverse(pOutdoorSector, cull_frustum, …)`
   + capture, gathering shadow casters from the sun's POV (`flows/renderer-scene-graph.md`).
3. **Depth render** — `Target->phase_smap_direct(sun, …)` sets the shadow-map depth target/region,
   then `cascade.GMCascade.r_dsgraph_render_graph(0/1)` renders the casters depth-only (grass shadows
   optionally for near cascades).
4. **Accumulate** — `Target->accum_direct_cascade(sub_phase, cascade.xform, …, bias)`: a full-screen
   directional pass that, per pixel, samples this cascade's shadow-map region (PCF), reads the
   G-buffer, computes sun lighting, and writes the accumulator. Z-test bounds the pass to the
   cascade's depth slice. After all cascades, `accum_direct_blend` composites.

`accum_direct` has a **filtered variant `accum_direct_f`** (softer sun shadows) selected by option, and
MSAA variants (`s_accum_direct_msaa`). The directional shader is `blender_light_direct` with per-cascade
elements (near/mid/far, ± minmax-SM optimization). *(shader sampling agent-traced.)*

## Function digests (spine)

- **`R_sun.cpp:CRender::init_cacades`** — *Does:* allocate `m_sun_cascades`, set per-cascade size/bias
  from `ps_ssfx_shadow_cascades`. *Called by:* renderer create. *(verified.)*
- **`R_sun.cpp:CRender::render_sun_cascades`** — *Does:* loop cascades → `render_sun_cascade`; manage
  the sunshaft ray chain. *Called by:* `r4_R_render.cpp` (lighting step). *(verified.)*
- **`R_sun.cpp:CRender::render_sun_cascade`** — *Does:* fit ortho light frustum, collect casters via
  `GMCascade`, render shadow-map depth (`phase_smap_direct`), accumulate (`accum_direct_cascade`).
  *Reads:* `Lights.sun_adapted`, camera frustum. *Writes:* the cascade's shadow map + the accumulator.
  *(verified entry; fit/stabilization agent-traced.)*
- **`r4_rendertarget_accum_direct.cpp:CRenderTarget::accum_direct_cascade` / `accum_direct` /
  `accum_direct_f`** — *Does:* full-screen directional accumulation per cascade (filtered variant for
  soft shadows). *Calls:* `blender_light_direct` element, `phase_accumulator`. *(verified entries;
  body agent-traced.)*
- **`r4_rendertarget_phase_smap_D.cpp:CRenderTarget::phase_smap_direct`** — *Does:* set the sun
  shadow-map depth target/region. *(verified entry.)*

## Isolation ledger (the seams)

- **Owned state:** `m_sun_cascades` (each with size/bias/frustum/xform + a `GMCascade` dsgraph), the
  shared sun shadow-map texture.
- **Sun source:** `RImplementation.Lights.sun_adapted` (env-adapted) — `flows/renderer-lighting.md`'s
  `CLight_DB`. The `rain_light` reuses a directional setup for rain shadows.
- **Caster seam:** each cascade's `GMCascade` is a `flows/renderer-scene-graph.md` instance run from the
  sun's POV (depth-only) — same machinery as `GMBase`/`GMLight`, flags `0`.
- **Accumulator seam:** writes the same HDR lighting accumulator as point/spot (F54) and reads the
  G-buffer (`flows/render-frame.md`); resolved by `flows/renderer-post-process.md`'s combine.
- **Blender seam:** `blender_light_direct` (+ MSAA/minmax variants) — `flows/renderer-blender-compile.md`.
- **Config:** `ps_ssfx_shadow_cascades` (cascade sizes), `ps_r2_ls_flags` sun bit, sun depth bias/scale,
  the filtered-sun option (`accum_direct_f`), grass-shadow cascade limit.
- **Relation to F54:** separate path, shared accumulator + shadow infrastructure; this is the directional
  case, F54 is point/spot.

## Threading

Main render thread, in the frame's lighting step (cascades rendered sequentially before the post chain).

## Failure & edge cases

- **Shadow shimmer/crawl** on camera motion is countered by texel-snapping; disabling it (or cascade
  size mismatch) reintroduces it.
- **Cascade seams** — a visible band where cascades meet is a split-distance/bias tuning issue
  (`ps_ssfx_shadow_cascades`, depth bias).
- **One sun shadow texture shared by cascades** — each cascade renders into its region; a region/size
  change affects neighbors.
- **Sun off** (`R2FLAG_SUN`) skips the whole path — scenes go flat-ambient.

## Deeper

- Cascade model + render: `xrRender/R_sun.cpp` (`init_cacades`, `render_sun_cascades`,
  `render_sun_cascade`), `R_sun_support.cpp`/`.h` (frustum fit), `r_sun_cascades.h` (`cascade` struct)
- Accumulation + depth: `xrRenderPC_R4/r4_rendertarget_accum_direct.cpp` (`accum_direct_cascade`,
  `accum_direct`, `accum_direct_f`), `r4_rendertarget_phase_smap_D.cpp` (`phase_smap_direct`)
- Casters: `flows/renderer-scene-graph.md`; sun DB: `flows/renderer-lighting.md`; frame:
  `flows/render-frame.md`; blender: `flows/renderer-blender-compile.md`

## Maintenance

- `render_sun_cascade` is the spine — re-grep it (+ `init_cacades` for the cascade count/sizes) if the
  CSM model changes.
- Promote the *(agent-traced)* ortho-fit (`R_sun_support` `FixedConvexVolume`), texel stabilization, and
  the per-cascade pixel-shader sampling to verified when a shadow-quality task needs exact behavior.
- This + F54 together promote `engine/renderer.md`'s inferred deferred-lighting passes — keep both
  cross-linked.
