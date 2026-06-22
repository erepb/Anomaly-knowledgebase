---
domain: flows/renderer-detail-grass
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/DetailManager.cpp, src/Layers/xrRender/DetailManager.h, src/Layers/xrRender/DetailManager_CACHE.cpp, src/Layers/xrRender/DetailManager_Decompress.cpp, src/Layers/xrRender/DetailManager_VS.cpp, src/Layers/xrRender/DetailManager_soft.cpp, src/Layers/xrRender/DetailModel.cpp, src/Layers/xrRender/DetailModel.h]
related: [flows/renderer-scene-graph.md, flows/renderer-sun-shadows.md, flows/renderer-lighting.md, flows/renderer-backend.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: detail / grass manager

How detail objects (grass, ground clutter, small foliage) are placed on terrain, cached in a
camera-centered grid, decompressed from the baked level data, and rendered as massively-instanced,
wind-swayed, distance-faded geometry. Unlike `flows/renderer-visual-types.md` objects, detail objects
are baked into a per-level slot grid and streamed around the camera.

**Provenance:** researched with a read-only agent, then spot-verified against code. The class
(`CDetailManager` with `Load`/`UpdateVisibleM`/`Render`/`cache_Update`/`hw_Render(light*)`), the detail
model (`CDetail : IRender_DetailModel`), and the global (`CRender::Details`) are personally
code-verified. The slot-cache ring-buffer, terrain-trace decompression, instanced VS render, and wind
math are *(agent-traced — not personally re-read)*.

## Trigger / entry

`CDetailManager::Load` runs at level load (reads the baked detail grid). Per frame, a calc task updates
the camera-centered cache + visibility, and `Details->Render()` draws visible detail — in the main pass
and, when grass shadows are enabled, in the sun/light shadow passes.

## Sequence (verified structure)

### What detail objects are

Grass/clutter scattered on terrain, baked into a per-level **slot grid** (each slot a small terrain
footprint holding a few detail-object *types*). A `CDetail` (`: IRender_DetailModel`) is one type's
geometry + wind-sway params. They differ from regular visuals: massively instanced, camera-cached,
distance-faded, and wind-animated — not individually scene-graph-managed.

### Camera-centered slot cache

`CDetailManager` keeps a grid of slots around the camera. As the camera moves, `cache_Update` shifts
the grid ring-buffer-style: new slots ahead are queried from the baked DB and queued for decompress
(budgeted per frame), slots behind are evicted. **Decompress** (`DetailManager_Decompress`) ray-casts
down onto terrain to place each grass item at the right height/normal, dithers the per-type palette to
choose which types appear, applies random scale/rotation + slope/sector culling, and bakes per-item
hemi/sun color. *(cache + decompress agent-traced.)*

### Visibility + render

`UpdateVisibleM` (a calc task) frustum/HOM-tests slots, computes per-item screen-space area, fades/culls
by distance (dither-stable, anti-popping), and bins items into still / wave1 / wave2 lists per type.
`CDetailManager::Render` then dispatches to the **VS-instanced path** (`hw_Render`): set wind/wave sway
constants (two animated sine offsets blended by wind strength), and per detail-type stream per-instance
matrices+colors into a constant array, drawing batched against a pre-baked VB/IB via `RCache.Render`.
A **soft path** (`DetailManager_soft`) is the CPU fallback for no-VS hardware. *(render/wind agent-traced.)*

### Grass in shadow passes (verified seam)

`hw_Render(light* L)` takes a light — detail is also rendered into shadow maps: the sun cascades
(`flows/renderer-sun-shadows.md`) and dynamic light shadows (`flows/renderer-lighting.md`) call detail
render with a fade distance derived from the light, gated by `ps_ssfx_grass_shadows`. That's why grass
casts shadows.

## Function digests (spine)

- **`DetailManager.cpp:CDetailManager::Load`** — *Does:* open the level detail grid, load `CDetail`
  models, init the cache. *Called by:* level load. *(verified entry.)*
- **`DetailManager_CACHE.cpp:CDetailManager::cache_Update`** — *Does:* shift the camera grid, queue +
  budget slot decompress. *(verified entry; body agent-traced.)*
- **`DetailManager.cpp:CDetailManager::UpdateVisibleM`** — *Does:* frustum/HOM cull slots + SSA-fade
  items into still/wave1/wave2 lists. *(verified entry.)*
- **`DetailManager.cpp:CDetailManager::Render`** — *Does:* prep wind, dispatch to `hw_Render`/soft.
  *Called by:* main frame + shadow passes. *(verified entry.)*
- **`DetailManager_VS.cpp:CDetailManager::hw_Render(light*)`** — *Does:* VS-instanced render of visible
  detail (batched per-instance constants), with light-derived fade for shadow passes. *Calls:*
  `RCache.Render`. *(verified entry; body agent-traced.)*
- **`DetailModel.cpp:CDetail::Load`** — *Does:* load one detail type's geometry + sway/scale params.
  *(verified symbol.)*

## Isolation ledger (the seams)

- **Owned state:** `CRender::Details` (`CDetailManager`) — the slot grid cache, the visible item lists,
  the baked `CDetail` model set, the pre-baked instancing VB/IB.
- **Level-data seam:** the baked detail grid (per-level details data) loaded at level load.
- **Terrain seam:** decompression ray-casts the static collision model to place items on the ground.
- **Frame seam:** `Details->Render()` in the main scene pass (`flows/render-frame.md`) and the shadow
  passes (`flows/renderer-sun-shadows.md`, `flows/renderer-lighting.md` via `hw_Render(light*)`).
- **Backend seam:** instanced draws via `RCache` (`flows/renderer-backend.md`); detail shader + wind
  constants from `flows/renderer-resources-shaders.md`.
- **Config:** grass density/height/radius (`ps_r__Detail_*`), grass-shadow cascade limit + fade
  (`ps_ssfx_grass_shadows`), terrain align/slope (`ps_ssfx_terrain_grass_*`), the `rsDetails`/
  `rsGrassShadow` device flags.

## Threading

The cache + visibility update can run as a parallel calc task (MT flag); render is on the render thread
(main pass + each shadow pass).

## Failure & edge cases

- **Pop-in/out** of grass is tuned by the dither-stable fade + cache budget; a too-small cache or
  budget shows streaming pop as the camera moves fast.
- **Grass shadows are separate work** — each shadow pass re-renders detail; disabling
  `ps_ssfx_grass_shadows` is a big perf lever.
- **Baked grid** — grass placement is per-level baked; runtime density/height only scale/cull it.
- **No-VS fallback** (soft path) is much slower — modern hardware uses `hw_Render`.

## Deeper

- Manager + visibility: `xrRender/DetailManager.cpp`/`.h` (`Load`, `UpdateVisibleM`, `Render`)
- Cache + decompress: `xrRender/DetailManager_CACHE.cpp` (`cache_Update`), `DetailManager_Decompress.cpp`
- Render: `xrRender/DetailManager_VS.cpp` (`hw_Render`), `DetailManager_soft.cpp` (fallback)
- Detail model: `xrRender/DetailModel.cpp`/`.h` (`CDetail`)
- Frame/shadows: `flows/render-frame.md`, `flows/renderer-sun-shadows.md`, `flows/renderer-lighting.md`;
  backend: `flows/renderer-backend.md`

## Maintenance

- `Render` + `hw_Render` + `cache_Update` are the spine — re-grep if the cache or instancing model
  changes.
- Promote the *(agent-traced)* cache ring-buffer, terrain-trace decompress, and wind/instancing render
  to verified when a grass task needs exact behavior.
- Keep the "grass renders in shadow passes via `hw_Render(light*)`" fact — it ties detail to F54/F55.
