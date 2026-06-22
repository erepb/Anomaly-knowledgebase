---
domain: flows/renderer-lighting
tags: [engine, renderer, r4, dx11, deferred, flow]
sources: [src/Layers/xrRender/light.cpp, src/Layers/xrRender/light.h, src/Layers/xrRender/Light_DB.cpp, src/Layers/xrRender/Light_DB.h, src/Layers/xrRender/Light_Package.cpp, src/Layers/xrRender/Light_Package.h, src/Layers/xrRender/lights_render.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_accum_point.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_accum_spot.cpp, src/Layers/xrRenderPC_R4/r4_rendertarget_phase_smap_S.cpp, src/Layers/xrRenderPC_R4/SMAP_Allocator.h]
related: [flows/render-frame.md, flows/renderer-scene-graph.md, flows/renderer-visibility.md, flows/renderer-sun-shadows.md, flows/renderer-blender-compile.md, flows/renderer-post-process.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: dynamic light accumulation & shadow maps (R4 deferred)

How point/spot lights are collected, how their shadow maps are allocated and rendered, and how they
accumulate into the deferred lighting buffer. This is the expansion of `flows/render-frame.md` step 5
(light accumulation) for dynamic lights. **The sun/directional cascades are a separate flow (F55)** —
mentioned here but not deep-dived. Each light reuses the per-view scene-graph instance (`GMLight`) from
`flows/renderer-scene-graph.md` to collect its shadow casters.

**Provenance:** researched with a read-only agent, then spot-verified against code. The light object
(`class light` with its embedded `CDSGraphManager GMLight` at flags `0`, `export_`), the DB
(`CLight_DB::add_light`, `sun_original`/`sun_adapted`), the package split, and the entry points
(`CRender::render_lights(light_Package&)`, `accum_point(light*)`, `accum_spot(light*)`,
`phase_smap_spot(light*)`) are personally code-verified. The accumulation pass internals (stencil
masking, shadow-matrix build), `SMAP_Allocator` bin-packing, and GI/indirect are *(agent-traced — not
personally re-read)*.

## Trigger / entry

Each frame the light DB exports active lights into packages; the frame loop
(`flows/render-frame.md`) calls `CRender::render_lights(LP_normal)` then `render_lights(LP_pending)`
during the lighting step, after the G-buffer is filled.

## Sequence (verified structure)

### Light model & DB

`class light` carries type (point/spot/omnipart/direct/reflected), a shadow flag, color/range/cone, an
occlusion-query visibility record, and **its own `CDSGraphManager GMLight`** — a per-light scene-graph
instance (flags `0`: no HOM/SSA/fade) used to collect that light's shadow casters from the light's
point of view (same machinery as the main view, `flows/renderer-scene-graph.md`). `CLight_DB` holds the
static/hemi lights and the two sun lights (`sun_original`, `sun_adapted`); **`add_light(light*)`** runs
each frame per active light and calls `light::export_()`, which buckets the light (via its visibility
state) into `RImplementation.LP_normal` or `LP_pending` (`light_Package` = point / spot / shadowed
vectors). The **normal vs pending split** lets lights with an in-flight occlusion query
(`flows/renderer-visibility.md`) be processed second so the GPU result can land.

### render_lights (the spine)

`CRender::render_lights(light_Package& LP)`:
1. **Visibility filter** — `vis_update()` each shadowed light; drop the invisible; `optimize_smap_size`
   shrinks a light's shadow-map resolution by screen size/angle.
2. **Atlas allocation** — pack the surviving lights' shadow maps into a shared depth texture via
   `SMAP_Allocator` (bin-packing); lights that don't fit roll to the next batch. *(allocator
   agent-traced.)*
3. **Per-batch shadow render** — for each shadowed light in the batch: set phase `PHASE_SMAP`, build
   the light's caster queue (`L->GMLight.traverse` + `r_dsgraph_capture_static/dynamic`), set the
   atlas rect as viewport via **`phase_smap_spot(L)`** (front-face cull, depth-only), and render the
   casters depth-only (`L->GMLight.r_dsgraph_render_static/dynamic`).
4. **Accumulate** — for each light call **`accum_spot(L)`** (shadowed/spot) or **`accum_point(L)`**
   (point), sampling the G-buffer + that light's shadow-map region and additively blending into the
   lighting accumulator. Volumetric/indirect passes follow if enabled.
5. Unshadowed point/spot lights skip steps 2–3 and go straight to accumulation.

### Accumulation pass

`accum_point`/`accum_spot` (`r4_rendertarget_accum_*`): stencil-mark the light's bounding volume (so
only lit pixels run the expensive pass), bind the deferred light shader element (`blender_light_point`/
`_spot` from `flows/renderer-blender-compile.md`), set light constants + (for spot) the shadow-matrix
that projects each fragment's world position into the shadow-map atlas rect, then render the light
volume additively into the accumulator RT. MSAA adds per-sample variants. *(stencil/shadow-matrix body
agent-traced.)*

## Function digests (spine)

- **`light.cpp:light::export_`** — *Does:* per-frame xform calc + build `GMLight` caster queue + bucket
  into `LP_normal`/`LP_pending`. *Calls:* `xform_calc`, `vis_prepare`, `GMLight.traverse`/capture.
  *Called by:* `CLight_DB::add_light`. *(verified entry.)*
- **`Light_DB.cpp:CLight_DB::add_light`** — *Does:* per-frame light intake → `export_`. *Reads:* the
  light list + sun. *(verified.)*
- **`lights_render.cpp:CRender::render_lights`** — *Does:* visibility filter → SMAP atlas alloc →
  per-light shadow-map render (via `GMLight`) → accumulate. *Reads:* a `light_Package`. *Writes:* the
  shadow atlas + accumulator. *Calls:* `vis_update`, `SMAP_Allocator::push`, `phase_smap_spot`,
  `GMLight` render, `accum_spot`/`accum_point`. *Called by:* `r4_R_render.cpp` (frame step 5).
  *(verified entry.)*
- **`r4_rendertarget_phase_smap_S.cpp:CRenderTarget::phase_smap_spot`** — *Does:* set the light's atlas
  rect as a depth-only render target/viewport (front-face cull). *(verified entry.)*
- **`r4_rendertarget_accum_point.cpp:accum_point` / `accum_spot.cpp:accum_spot`** — *Does:* stencil-mask
  the light volume + sample G-buffer (+ shadow map for spot) + additive blend into the accumulator.
  *Calls:* `draw_volume`, the `blender_light_*` element. *(verified entries; bodies agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `CLight_DB` (light list + sun), `LP_normal`/`LP_pending` packages, the per-light
  `GMLight` dsgraph, the shadow-map atlas + `SMAP_Allocator`.
- **Input seam:** game creates lights via the render factory (`IRender_Light`) and configures
  type/pos/range/color/shadow; `add_light` runs them each frame.
- **Shadow-caster seam:** each light's `GMLight` is a `flows/renderer-scene-graph.md` instance (flags
  `0`) — the same insert/render code, run from the light's POV, depth-only.
- **G-buffer/accumulator seam:** reads the deferred G-buffer (`flows/render-frame.md`) and writes the
  HDR lighting accumulator that `flows/renderer-post-process.md`'s combine resolves.
- **Blender seam:** `blender_light_point/spot/mask` + MSAA variants (`flows/renderer-blender-compile.md`).
- **Occlusion seam:** `vis_prepare`/`vis_update` use the GPU occlusion-query pool
  (`flows/renderer-visibility.md`); the LP_normal/LP_pending split hides the one-frame query latency.
- **Sun:** the directional cascades are the separate F55 flow (`R_sun.cpp`, `phase_smap_direct`,
  `accum_direct`) — not this path.
- **Config:** `ps_r2_ls_flags` (shadow/sun/GI bits), shadow-map size/quality, depth bias/scale.

## Threading

Main render thread, in the frame's lighting step; the only cross-frame dependency is the deferred
occlusion-query read-back (handled by the package split).

## Failure & edge cases

- **Shadow atlas overflow** spills lights into extra batches (more render-target switches) — many
  large shadowed lights are expensive; `optimize_smap_size` mitigates by shrinking distant ones.
- **One-frame query latency** (LP_pending) can pop a light in/out on fast camera motion — by design.
- **Deferred ⇒ stencil-masked volume** — accumulation only touches lit pixels; a light "not affecting"
  forward/translucent geometry is expected (those aren't in the G-buffer).
- **Point shadows** use omnipart sub-lights — a single point light can cost several shadow renders.

## Deeper

- Light object + DB + package: `xrRender/light.cpp`/`.h` (`light`, `export_`, `GMLight`),
  `Light_DB.cpp`/`.h` (`CLight_DB::add_light`, sun), `Light_Package.cpp`/`.h` (`LP_normal`/`LP_pending`)
- Spine: `xrRender/lights_render.cpp` (`CRender::render_lights`)
- Shadow map + accum + atlas: `xrRenderPC_R4/r4_rendertarget_phase_smap_S.cpp` (`phase_smap_spot`),
  `r4_rendertarget_accum_point.cpp`/`accum_spot.cpp`, `SMAP_Allocator.h`
- Occlusion: `flows/renderer-visibility.md` (`light_vis.cpp`); casters: `flows/renderer-scene-graph.md`;
  frame: `flows/render-frame.md`; combine: `flows/renderer-post-process.md`; sun: F55 (`R_sun.cpp`)

## Maintenance

- `render_lights` is the spine — re-grep it (and the `accum_*`/`phase_smap_spot` entries) if the
  shadow-atlas or accumulation model changes.
- Promote the *(agent-traced)* `accum_*` stencil/shadow-matrix bodies, `SMAP_Allocator::push`, and the
  GI/indirect path to verified when a lighting/shadow task needs exact behavior.
- The sun/directional cascades are now `flows/renderer-sun-shadows.md` (F55); both flows together
  promote `engine/renderer.md`'s inferred deferred-lighting passes.
- Keep the "per-light GMLight = scene-graph reused for shadow casters" fact — it ties this to F49.
