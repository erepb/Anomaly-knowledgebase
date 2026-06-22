---
domain: flows/renderer-wallmark-engine
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/WallmarksEngine.cpp, src/Layers/xrRender/WallmarksEngine.h, src/Layers/xrRender/dxWallMarkArray.cpp, src/Layers/xrRender/dxWallMarkArray.h, src/Layers/xrRender/SkeletonCustom.cpp, src/Layers/xrRender/r__dsgraph_render.cpp]
related: [flows/wallmarks-particles.md, flows/renderer-scene-graph.md, flows/renderer-backend.md, flows/renderer-skeleton.md, flows/renderer-resources-shaders.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: wallmark / decal engine (renderer side)

How decals (bullet holes, blood, scorch, static wallmarks) are created, clipped, batched, aged, and
rendered. This is the **engine side** of the game trigger in `flows/wallmarks-particles.md` (F44): the
game does a `GMLib.GetMaterialPair` lookup to choose a wallmark shader, then calls the render interface,
which lands here in `CWallmarksEngine`.

**Provenance:** researched with a read-only agent, then spot-verified against code. The class
(`CWallmarksEngine` with `AddStaticWallmark`/`AddSkeletonWallmark` overloads, `RecurseTri`,
`BuildMatrix`, `Render`), the static-vs-skeleton split (skeleton wallmarks render via
`CKinematics::RenderWallmark` and the scene-graph `r_dsgraph_render_wmarks`), and `dxWallMarkArray`
(`AppendMark`/`GenerateWallmark`) are personally code-verified by symbol. The clip/age bodies and the
dynamic-VB streaming are *(agent-traced — not personally re-read)*.

## Trigger / entry

After a hit, game code (`flows/wallmarks-particles.md`) calls the render interface's
`add_StaticWallmark` / `add_SkeletonWallmark` (with a shader from the material pair). These dispatch to
`RImplementation.Wallmarks` (`CWallmarksEngine`).

## Sequence (verified structure)

### Two kinds of wallmark

- **Static** — projected onto static world geometry. Stored in `CWallmarksEngine`'s own per-shader
  batched arrays, aged, and drawn each frame by `CWallmarksEngine::Render`.
- **Skeleton/dynamic** — attached to a skinned visual. The decal follows the rig: `CKinematics`
  (`flows/renderer-skeleton.md`) owns the wallmark, and at render time `CKinematics::RenderWallmark`
  skins its vertices. These also go through the scene-graph's sorted **Wmark** lists
  (`r_dsgraph_render_wmarks`, `flows/renderer-scene-graph.md`).

### Static wallmark creation (verified entry)

`AddStaticWallmark(tri, verts, contact_point, normal, shader, size)` → the internal builder:
1. **Query** static geometry (CDB box query) for triangles near the contact point.
2. **`BuildMatrix`** — an orthographic projection centered at the contact, oriented to the surface
   normal, scaled by `1/size` (maps decal space → [0,1] UV).
3. **`RecurseTri`** — clip the decal quad to each nearby triangle (flood-filling across adjacent
   triangles while the surface stays roughly coplanar), emitting `FVF::LIT` vertices with projected
   UVs into the wallmark's vertex list.
4. **Store + age** — pool-allocate the wallmark (`static_wm_allocate`), stamp birth time + TTL
   (`ps_r__WallmarkTTL`), bound it, and register it per-shader/per-sector (replacing a near-duplicate
   if found). *(clip/age bodies agent-traced.)*

### Render (verified entry)

`CWallmarksEngine::Render` (called in the frame after the scene pass): per shader slot, lock a dynamic
vertex buffer (`R_DStreams`, `flows/renderer-backend.md`); for each **static** wallmark, frustum + SSA
cull, compute a **fade alpha from age**, stream its vertices; for each **skeleton** wallmark, call
`W->Parent()->RenderWallmark(W, verts)` to skin it; flush per shader (set shader/geometry, submit). It
finishes by calling `RImplementation.GMBase.r_dsgraph_render_wmarks()` for visual-attached Wmark lists.
Expired wallmarks are removed (`UpdateWallmarks`).

`dxWallMarkArray` (`IWallMarkArray`) is the game-facing holder: `AppendMark(texture)` creates a
wallmark shader (DX11 blood uses `wallmark_blood` when `o.ssfx_blood`), `GenerateWallmark` returns a
random one from the set — this is how the material-pair's mark list (`flows/wallmarks-particles.md`)
becomes a renderer shader.

## Function digests (spine)

- **`WallmarksEngine.cpp:CWallmarksEngine::AddStaticWallmark`** — *Does:* build a static decal (CDB
  query → `BuildMatrix` → `RecurseTri` → store+age). *Reads:* static geometry. *Writes:* per-shader
  wallmark pool. *Called by:* the render interface (game). *(verified entry.)*
- **`WallmarksEngine.cpp:CWallmarksEngine::RecurseTri` / `BuildMatrix`** — *Does:* clip the decal to
  triangles (flood-fill) / build the projection matrix. *(verified symbols; bodies agent-traced.)*
- **`WallmarksEngine.cpp:CWallmarksEngine::AddSkeletonWallmark`** — *Does:* register a decal on a
  skinned object (the `CKinematics` owns it). *(verified entry.)*
- **`WallmarksEngine.cpp:CWallmarksEngine::Render`** — *Does:* per-shader stream + draw static
  wallmarks (age-fade, SSA cull), skin skeleton wallmarks, then `r_dsgraph_render_wmarks`. *Calls:*
  `R_DStreams` lock/unlock, `RenderWallmark`, the sorted Wmark render. *Called by:* the R4 frame.
  *(verified entry; streaming agent-traced.)*
- **`SkeletonCustom.cpp:CKinematics::RenderWallmark`** — *Does:* skin a skeleton wallmark's vertices by
  bone matrices, emit `FVF::LIT` with age-fade. *Called by:* `CWallmarksEngine::Render`. *(verified
  symbol.)*
- **`dxWallMarkArray.cpp:dxWallMarkArray::AppendMark` / `GenerateWallmark`** — *Does:* build / pick a
  wallmark shader for a material's mark list. *(verified.)*

## Isolation ledger (the seams)

- **Owned state:** `RImplementation.Wallmarks` (`CWallmarksEngine`) — per-shader static-wallmark pools
  (per sector) + the free pool; skeleton wallmarks are owned by each `CKinematics`.
- **Game trigger seam:** `flows/wallmarks-particles.md` (F44) — game `GetMaterialPair` → mark list
  (`dxWallMarkArray`) → render interface `add_StaticWallmark`/`add_SkeletonWallmark`.
- **Static-geometry seam:** CDB query (level static model) for clipping the static decal.
- **Skeleton seam:** skeleton wallmarks ride the rig via `CKinematics::RenderWallmark`
  (`flows/renderer-skeleton.md`); the visual-attached ones also flow through the scene-graph Wmark
  lists (`r_dsgraph_render_wmarks`, `flows/renderer-scene-graph.md`).
- **Backend seam:** dynamic VB streaming via `R_DStreams` (`flows/renderer-backend.md`); each slot's
  `Shader*` from `flows/renderer-resources-shaders.md`.
- **Config:** `ps_r__WallmarkTTL` (lifetime), wallmark depth/view shift (z-fight avoidance), SSA cull
  threshold, static/dynamic enable toggles, `o.ssfx_blood`.

## Threading

Creation happens on the game/update thread (on hit); render + aging on the render thread in the frame.

## Failure & edge cases

- **Z-fighting** is avoided by a small projection depth/view shift — disabling it makes decals shimmer
  against the surface.
- **Decal cap + TTL** — old wallmarks expire and near-duplicates replace each other; heavy fire won't
  accumulate unboundedly.
- **Static vs skeleton** take different render paths — a decal "not following" a corpse means it was
  added as static, not skeleton.
- **SSA cull** drops distant/small wallmarks — expected, not a bug.

## Deeper

- Engine: `xrRender/WallmarksEngine.cpp`/`.h` (`AddStaticWallmark`, `RecurseTri`, `BuildMatrix`,
  `Render`, `AddSkeletonWallmark`)
- Mark-list holder: `xrRender/dxWallMarkArray.cpp`/`.h` (`AppendMark`, `GenerateWallmark`)
- Skeleton decals: `xrRender/SkeletonCustom.cpp` (`CKinematics::RenderWallmark`); sorted path:
  `xrRender/r__dsgraph_render.cpp` (`r_dsgraph_render_wmarks`)
- Game trigger: `flows/wallmarks-particles.md`; streaming: `flows/renderer-backend.md`

## Maintenance

- `CWallmarksEngine::Render` + `AddStaticWallmark` are the spine — re-grep if the wallmark pool or
  static-vs-skeleton split changes.
- Promote the *(agent-traced)* `RecurseTri`/`BuildMatrix` clip math + the `Render` streaming/aging to
  verified when a decal task needs exact behavior.
- Keep the "engine renders, game triggers" framing — it ties this to `flows/wallmarks-particles.md`.
