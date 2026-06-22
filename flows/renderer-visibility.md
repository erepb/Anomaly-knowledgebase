---
domain: flows/renderer-visibility
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/HOM.cpp, src/Layers/xrRender/HOM.h, src/Layers/xrRender/occRasterizer.cpp, src/Layers/xrRender/occRasterizer_core.cpp, src/Layers/xrRender/r__occlusion.cpp, src/Layers/xrRender/r__occlusion.h, src/Layers/xrRender/r__sector.cpp, src/Layers/xrRender/r__sector.h, src/Layers/xrRender/r__sector_detect.cpp, src/Layers/xrRender/light_vis.cpp]
related: [flows/renderer-scene-graph.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: renderer visibility & occlusion (HOM + sectors/portals + occlusion queries)

The three culling mechanisms that decide what is potentially visible *before* the scene-graph collects
it: the **HOM** (hierarchical occlusion map — CPU software-rasterized occluder depth), the
**sector/portal** graph (spatial partition the camera traverses through visible portals), and
**hardware occlusion queries** (GPU fragment-count tests, mostly for lights). This is the cull *input*
that `flows/renderer-scene-graph.md`'s `traverse`/insert consume — that flow left this machinery
agent-traced; this resolves it.

**Provenance:** researched with a read-only agent, then spot-verified against code. The class
structure (`CHOM`, `CSector`/`CPortal`, `R_occlusion`), entry points (`CHOM::visible` overloads,
`CSector::traverse`, `detectSector`/`detectLastSector`, `occq_begin`/`occq_end`/`occq_get`), and the
"disabled ⇒ everything visible" + camera-inside short-circuits are personally code-verified. The
software-rasterizer internals (`occRasterizer`), the per-frame HOM build (`Render_DB`/`propagate`), and
the light-query scheduling (`light_vis.cpp`) are *(agent-traced — not personally re-read)*.

## Trigger / entry

Per frame: `CHOM::Render(frustum)` rasterizes occluders (early, possibly on a parallel task);
`CRender::Calculate` updates the camera sector (`detectLastSector`); then
`RImplementation.GMBase.traverse(pLastSector, …)` walks sectors/portals, HOM-testing each; light
visibility queries are issued during the frame and read back the next.

## Sequence (verified structure)

### A three-tier funnel

1. **HOM (occluder depth, CPU).** `CHOM::Load` reads `level.hom` occluder geometry into an AABB tree
   at level load. Each frame `CHOM::Render(frustum)` frustum-queries the occluders and software-
   rasterizes them into a small mip-pyramid depth buffer (`occRasterizer`), then `propagate`s the
   pyramid. Bounding-volume tests then reject occluded geometry/portals: **`CHOM::visible(Fbox3&)`**
   returns TRUE if HOM is disabled *or the camera is inside the box*, else delegates to `_visible`
   against the rasterized depth; `visible(Fsphere&)` boxes the sphere; `visible(sPoly&)` tests a portal
   polygon; `visible(vis_data&)` adds **adaptive re-test scheduling** (visible objects re-checked after
   a frame delay, hidden ones re-checked next frame). *(rasterizer + Render_DB agent-traced.)*
2. **Sectors/portals (topology).** The level is partitioned into `CSector` (each owns a visual root +
   a portal list) connected by `CPortal` (a polygon linking two sectors). **`CSector::traverse(CFrustum&&,
   CDSGraphManager&)`** registers the current sector's frustum, then for each portal: sphere-test vs
   the frustum, pick the far sector, optional SSA cull, clip the portal polygon to the frustum, **HOM-
   test the clipped polygon**, and if it survives, build a tighter frustum from the portal and recurse
   into the next sector. Only visited sectors contribute geometry — so the scene-graph never even
   considers geometry behind closed/occluded portals. *(traverse body agent-traced; entry verified.)*
3. **Hardware occlusion queries (GPU, lights).** `R_occlusion` is a pool of GPU queries:
   **`occq_begin(ID)`** pops a query and issues `BeginQuery`, **`occq_end(ID)`** issues `EndQuery`,
   **`occq_get(ID)`** reads the fragment count back (non-blocking with a timeout; on timeout/loss it
   conservatively reports visible) and recycles the query. Used a frame late to hide GPU latency,
   mainly for light volumes (`light_vis.cpp` `vis_prepare`/`vis_update`, *agent-traced*) — a light is
   kept only if its volume rasterized more than a small fragment threshold.

### Sector detection (camera placement)

`CRender::Calculate` calls **`detectLastSector(P)`** / `detectSector(P, dir)` (`r__sector_detect.cpp`):
ray-cast (down then up) against the portal CDB model first, else static geometry, to find the sector
containing the camera; cached in `RImplementation.pLastSector`. `pOutdoorSector` is the level's
largest sector (the outdoor fallback). The starting sector is what `traverse` seeds from.

## Function digests (spine)

- **`HOM.cpp:CHOM::visible(Fbox3&)`** — *Does:* `!bEnabled || camera-inside ⇒ TRUE`, else
  `_visible(B, m_xform_01)` against rasterized depth. *Reads:* `bEnabled`, `Device.vCameraPosition`,
  `Raster`. *Called by:* scene-graph insert, portal cull, detail/light tests. *(verified.)*
- **`HOM.cpp:CHOM::Render` / `Render_DB`** — *Does:* frustum-cull occluders, clip, software-rasterize,
  build the mip-pyramid. *(agent-traced.)*
- **`r__sector.cpp:CSector::traverse`** — *Does:* register sector frustum, then per portal
  sphere/SSA/clip/HOM-cull and recurse with a portal-derived frustum. *Writes:* the manager's
  sector-frustum set. *Called by:* `CDSGraphManager::traverse` + itself. *(entry verified; body
  agent-traced.)*
- **`r__sector_detect.cpp:CRender::detectSector/detectLastSector`** — *Does:* ray-cast portals then
  geometry to find the sector for a point. *Writes:* `pLastSector` (via caller). *(verified entry.)*
- **`r__occlusion.cpp:R_occlusion::occq_begin/occq_end/occq_get`** — *Does:* pool a GPU query, issue
  begin/end, read back fragment count (timeout ⇒ assume visible) + recycle. *Gotcha:* one-frame
  latency by design. *(verified signatures; bodies agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `RImplementation.HOM` (`CHOM` + its rasterizer), the sector/portal arrays
  (`Sectors`/`Portals`) + the portal CDB model (`rmPortals`), `RImplementation.pLastSector`/
  `pOutdoorSector`, and the `R_occlusion` query pool.
- **Output seam (to scene-graph):** `CDSGraphManager::traverse` consumes `pLastSector` + the
  sector-frustum set; `add_Static`/insert HOM-test each AABB; `capture_lights` HOM-tests light spheres
  then issues occlusion queries (`flows/renderer-scene-graph.md`).
- **Level-load source:** HOM occluders from `level.hom`; sector/portal graph + portal CDB from the
  level file; outdoor sector chosen by volume at load.
- **Threading:** HOM rasterization may run on a parallel render task (gated by an MT flag) early in the
  frame; occlusion-query read-back is deliberately deferred a frame.
- **Config:** HOM `bEnabled` (auto from load / console), occlusion-query enable (core param), SSA/LOD
  discard thresholds for portal culling, MT-calc flag for parallel HOM.
- **Links:** `flows/renderer-scene-graph.md` (consumer), `flows/render-frame.md` (frame order),
  light visibility in `light_vis.cpp` (lights → `flows/render-frame.md` lighting; future F54).

## Threading

HOM build can be a secondary-task workload (joined before traversal); sector traversal + visibility
tests run on the main render thread; occlusion-query results are read one frame after issue.

## Failure & edge cases

- **HOM disabled ⇒ `visible()` always TRUE** — no occlusion culling (only frustum/portal); a perf
  regression "everything drawn" can be a failed/missing `level.hom`.
- **Occlusion-query timeout/device-loss ⇒ conservative "visible"** — never culls on uncertainty.
- **Wrong sector detection** (camera exactly on a portal/boundary) can mis-seed traversal → flicker;
  detection validates the point is inside the sector AABB.
- **One-frame query latency** — a light popping in/out for a frame on fast camera motion is expected.

## Deeper

- HOM: `xrRender/HOM.cpp`/`.h` (`Load`/`Render`/`visible`), `xrRender/occRasterizer*.cpp` (software
  rasterizer)
- Sectors/portals: `xrRender/r__sector.cpp`/`.h` (`CSector::traverse`, `CPortal`),
  `xrRender/r__sector_detect.cpp` (`detectSector`/`detectLastSector`)
- Occlusion queries: `xrRender/r__occlusion.cpp`/`.h` (`occq_*`), `xrRender/light_vis.cpp` (light
  visibility)
- Consumer: `flows/renderer-scene-graph.md`; frame: `flows/render-frame.md`

## Maintenance
- `CHOM::visible` + `CSector::traverse` are the seams the scene-graph relies on — re-grep them if
  culling is restructured.
- Promote the *(agent-traced)* `occRasterizer`, `Render_DB`/`propagate`, and `light_vis` scheduling to
  verified by reading those bodies when a culling/perf task needs exact behavior; cross-update
  `flows/renderer-scene-graph.md` (which points here for the cull input).
- Keep the "disabled/timeout ⇒ visible" invariants — they explain conservative over-draw.
