---
domain: flows/renderer-scene-graph
tags: [engine, renderer, r4, dx11, deferred, flow]
sources: [src/Layers/xrRender/r__dsgraph_manager.h, src/Layers/xrRender/r__dsgraph_manager.cpp, src/Layers/xrRender/r__dsgraph_build.cpp, src/Layers/xrRender/r__dsgraph_render.cpp, src/Layers/xrRender/r__dsgraph_types.h, src/Layers/xrRender/r__sector.cpp, src/Layers/xrRenderPC_R4/r4.h, src/Layers/xrRender/light.h]
related: [flows/render-frame.md, flows/renderer-resources-shaders.md, flows/renderer-visibility.md, flows/renderer-backend.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: renderer scene-graph (dsgraph) — build & phased render

How visible geometry is collected each frame into a sorted render queue and flushed to the GPU in
phases. `CDSGraphManager` is the "what gets drawn, in what order" core that `flows/render-frame.md`'s
cull and geometry steps delegate to. **Key architecture:** the *same* manager class is instantiated
**per view** — `RImplementation.GMBase` for the main camera, `GMLight` per shadow-casting light,
`GMCascade` per sun cascade — so one body of code builds both the main render queue and every
shadow-map queue, differing only in cull-quality flags.

**Provenance:** researched with a read-only agent, then spot-verified against code. The manager
structure (`CDSGraphManager`, `RGraph`, the per-view instances `GMBase`/`GMLight`/`GMCascade`), the
build chain (`traverse` → capture → `add_Static`/`add_Dynamic` → `r_dsgraph_insert_*` →
`AddToRenderQueue`), and the render core (`r_dsgraph_render_graph` sort + state-batch + `Render`) are
personally code-verified. SSA/fade math and the sector-portal traversal body are *(agent-traced)*.

## Trigger / entry

Per frame, the R4 renderer calls `RImplementation.GMBase.traverse(pLastSector, ViewBase,
cameraPos, fullTransform)` to build the main scene graph; shadow generation calls the same on the
per-light/per-cascade instances. Render happens later, phase by phase, via the
`r_dsgraph_render_*` family.

## Sequence (verified)

### Build (cull → insert)

1. **`CDSGraphManager::traverse(CSector* start, CFrustum&, …)`** seeds the walk from the camera's
   sector and frustum.
2. **Sector-portal traversal** (`r__sector.cpp:CSector::traverse`, *agent-traced*) recursively walks
   portals, accumulating per-sector frustums (HOM/SSA/frustum-clipped) into the manager's sector map.
3. **Capture** (`r_dsgraph_capture_static`/`r_dsgraph_capture_dynamic`) extracts visible static BSP
   geometry and frustum-tested dynamic spatials, calling `add_Static`/`add_Dynamic` per renderable.
4. **`add_Static`/`add_Dynamic`** (`r__dsgraph_build.cpp`) recursively expand hierarchical/LOD/skeletal
   visuals, cull each leaf, then call **`r_dsgraph_insert_static`/`r_dsgraph_insert_dynamic`**.
5. **Insert** computes screen-space area (SSA), applies a fade-in gradient (anti-popin), runs the HOM
   test, and **routes by shader-element flag**: distort → `RGraph.map*Sorted.Distort`; strict
   back-to-front → `.Sorted`; emissive → `.Emissive`; wallmark → `.Wmark`; water (R4) → `mapWater`;
   HUD/scope/cam-attached → their own maps; otherwise opaque → `AddToRenderQueue` per shader pass.
6. **`AddToRenderQueue`** builds a `RenderPacket` carrying the precomputed pass resources (VS, PS, GS,
   HS, DS, constants, state, textures) plus a `RenderPacketSortKey`, and appends it to
   `mapStaticPasses[priority][pass]` (or the dynamic equivalent).

### Sort key (verified shape)

`RenderPacketSortKey` packs resource-pointer fragments into two 64-bit words. The **high** word holds
state / GS / HS / DS (things that change rarely → grouped first); the **low** word holds VS / PS /
constants / textures (change often → last). Sorting by (high, low) clusters draws so redundant GPU
state sets are skipped during render.

### Render (sort → batch → submit)

7. **`r_dsgraph_render_static/dynamic(priority, clear)`** forward to **`r_dsgraph_render_graph(queues,
   priority, clear, static_geometry)`** (the core, verified):
   - For each of `SHADER_PASSES_MAX` passes: sort the packet vector (`std::sort` if <4096 packets,
     else `xr_parallel_sort`).
   - Walk packets caching last-set resources: when the **high** key changes, set state/GS/HS/DS; per
     packet compare the **low** members and set VS/PS/constants/textures (+ `apply_lmaterial`) only on
     change.
   - For dynamic geometry, set the world transform (`set_xform_world(*item.pMatrix)`) and
     `apply_object`. Then `calcLOD(ssa, radius)` and `item.pVisual->Render(LOD)` (the per-visual draw
     that hits `CBackend`/`RCache`). Clear the queue if asked.
8. **Sorted/translucent lists** go through `r_dsgraph_render_graph_sorted` (per-item, no shader
   batching — back-to-front order matters): `r_dsgraph_render_sorted` (glass/translucent), `_emissive`
   (self-illum → accumulator), `_wmarks`, `_distort`, plus the HUD/scope variants.

## Phases (order within the R4 frame)

Opaque G-buffer (static prio 0 → dynamic prio 0) → LODs → HUD → wallmarks → emissive (to
accumulator) → lighting (`flows/render-frame.md`) → forward pass (static/dynamic prio 1) → sorted
(translucent + sorted-HUD) → water → distort (in `phase_combine`). Each maps to a `r_dsgraph_render_*`
call invoked from `r4_R_render.cpp` / `r4_rendertarget_phase_combine.cpp` (*invocation sites
agent-traced; the render functions verified*).

## Function digests (spine)

- **`r__dsgraph_manager.cpp:CDSGraphManager::traverse`** — *Does:* seed + run sector-portal walk for
  this view. *Writes:* the sector-frustum set, dedup set. *Calls:* `CSector::traverse`. *Called by:*
  `r4_R_render.cpp` (main) + shadow generation (per light/cascade).
- **`r__dsgraph_build.cpp:r_dsgraph_insert_static/_dynamic`** — *Does:* SSA + fade + HOM cull, then
  route a visual to the correct queue/sorted map by shader flag. *Writes:* `RGraph.map*`. *Calls:*
  `CalcSSA`, `AddToRenderQueue`. *Gotcha:* dynamic keeps a world-matrix pointer; static is identity.
- **`r__dsgraph_build.cpp:AddToRenderQueue`** — *Does:* build a `RenderPacket` (precached pass
  resources + sort key) and append. *Why:* precaching at insert makes render a tight pointer-compare
  loop.
- **`r__dsgraph_render.cpp:r_dsgraph_render_graph`** — *Does:* per-pass sort + state-batched submit.
  *Reads:* `mapStaticPasses`/`mapDynamicPasses`. *Writes:* `RCache` state. *Calls:* `RCache.set_*`,
  `item.pVisual->Render`. *Gotcha:* the high/low key split is what avoids redundant state sets — the
  whole point of the sort.
- **`r__dsgraph_render.cpp:r_dsgraph_render_graph_sorted`** — *Does:* per-item submit in sorted order
  (no batching). *Used by:* emissive/sorted/wmark/distort/HUD.

## Isolation ledger (the seams)

- **Owned state:** `RGraph` (`R_dsgraph::DynamicSceneRgraph`) — the per-priority/per-pass render
  queues + the sorted maps (Sorted/Emissive/Wmark/Distort) + HUD/scope/cam/water maps. One `RGraph`
  per manager instance.
- **Per-view instances:** `r4.h` declares `CRender::GMBase` (main view: `VQ_HOM|VQ_SSA|VQ_FADE`);
  `light.h` declares per-light `GMLight` (flags `0` — shadow views skip HOM/SSA/fade); sun cascades
  use `GMCascade` (`R_sun.cpp`). Same code, different cull quality.
- **Input seam:** sectors/portals + HOM occlusion (`r__sector.cpp`, `HOM.cpp`) and the engine spatial
  DB (`g_SpatialSpace`) feed `traverse`/capture — the cull machinery is `flows/renderer-visibility.md`.
- **Output seam:** `RCache`/`CBackend` (`set_VS/PS/GS/HS/DS/States/Constants/Textures`,
  `set_xform_world`) and `dxRender_Visual::Render` — the per-visual draw. Shaders come from
  `flows/renderer-resources-shaders.md`.
- **Lighting seam:** shadow passes read `RGraph.mapStaticPasses[..]`/`mapDynamicSorted` sizes to decide
  deferred-vs-forward shadow rendering (`lights_render.cpp`, `R_sun.cpp`).
- **Links:** built/consumed by `flows/render-frame.md`; LOD impostors via `r_dsgraph_render_lods`.

## Threading

Build + render run on the main render thread inside the frame; the only parallelism is
`xr_parallel_sort` for very large (>4096-packet) queues. Culling/bone prep happen earlier in the
frame's secondary tasks (`flows/engine-boot-frame.md`).

## Failure & edge cases

- **Deferred ⇒ opaque (prio 0) vs forward (prio 1)** split: translucent/glass go to the sorted/forward
  path, not the G-buffer — a "missing in SSAO/SSR" symptom is usually forward-rendered geometry.
- **SSA/fade gating** can drop or fade distant objects (pop-in tuning); LOD swap is decided at insert.
- **Per-view reuse**: a bug in the shared graph code affects shadows *and* the main view at once.

## Deeper

- Manager + queues: `xrRender/r__dsgraph_manager.h` (`CDSGraphManager`, `RGraph`,
  `r_dsgraph_render_static/dynamic`), `r__dsgraph_types.h` (`RenderPacket`, sort key,
  `DynamicSceneRgraph`)
- Build/insert: `xrRender/r__dsgraph_build.cpp` (`r_dsgraph_insert_static/_dynamic`, `AddToRenderQueue`)
- Render: `xrRender/r__dsgraph_render.cpp` (`r_dsgraph_render_graph`, `_graph_sorted`, the phase fns)
- Traversal: `xrRender/r__sector.cpp` (`CSector::traverse`); per-view instances: `xrRenderPC_R4/r4.h`
  (`GMBase`), `xrRender/light.h` (`GMLight`), `xrRender/R_sun.cpp` (`GMCascade`)
- Frame caller: `flows/render-frame.md`

## Maintenance

- The render core is `r_dsgraph_render_graph` — re-grep it if the sort key or state-batching changes.
- Promote the *(agent-traced)* SSA/fade math and `CSector::traverse` body to verified by reading
  `r__dsgraph_build.cpp` (`CalcSSA`, the insert routing) + `r__sector.cpp` when a culling task needs
  it. HOM/sectors/occlusion is now `flows/renderer-visibility.md` (F50); the draw seam is
  `flows/renderer-backend.md` (F48).
- Keep the "per-view instances" fact — it explains why shadow code and scene code share this file.
