---
domain: flows/renderer-backend
tags: [engine, renderer, r4, dx11, flow]
sources: [src/Layers/xrRender/R_Backend.cpp, src/Layers/xrRender/R_Backend.h, src/Layers/xrRender/R_Backend_Runtime.h, src/Layers/xrRender/R_Backend_Runtime.cpp, src/Layers/xrRender/R_Backend_xform.cpp, src/Layers/xrRender/R_DStreams.cpp, src/Layers/xrRender/r_constants_cache.h, src/Layers/xrRenderDX10/dx10R_Backend_Runtime.h, src/Layers/xrRenderDX10/dx10r_constants_cache.cpp, src/Layers/xrRenderDX10/dx10ConstantBuffer.cpp, src/Layers/xrRenderDX10/StateManager/dx10StateManager.cpp]
related: [flows/renderer-scene-graph.md, flows/renderer-resources-shaders.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: render backend (CBackend / the global RCache)

The state-caching draw submitter. `CBackend` (the single global `RCache`) is a software mirror of the
GPU pipeline: it remembers the currently-bound shaders / textures / state / constants / geometry and
**skips the D3D call when the requested value already matches**, deferring constant-buffer and
render-target binds until just before a draw. This is what `flows/renderer-scene-graph.md`'s
`RCache.set_*` + `pVisual->Render` calls and every full-screen post pass actually drive; it consumes
the `ShaderElement`/`SPass` resources from `flows/renderer-resources-shaders.md`.

**Provenance:** researched with a read-only agent, then spot-verified against code. The global
instance (`CBackend RCache`), the `set_Element` apply order, the `Render`→`DrawIndexed` pre-draw flush,
and the `R_DStreams` discard/no-overwrite lock model are personally code-verified. The per-stage
constant-buffer redistribution, the `dx10StateManager` object caches, and the matrix-constant upload in
`R_Backend_xform` are *(agent-traced — not personally re-read)*.

## Trigger / entry

Anything that draws: the scene-graph (`r_dsgraph_render_graph` → `set_States`/`set_VS/PS/…`/
`set_Constants`/`set_Textures` then `pVisual->Render`), full-screen post passes
(`flows/renderer-post-process.md`), and the engine bridges (UI/font/console/debug — future F61). All
go through the one global `RCache`.

## Sequence (verified)

### Apply a shader pass

**`CBackend::set_Element(ShaderElement*, pass)`** unpacks the `SPass` and applies, in order:
`set_States(P.state)` → `set_PS(P.ps)` → `set_VS(P.vs)` → (DX10/11) `set_GS(P.gs)` → (DX11)
`set_HS/set_DS/set_CS` → `set_Constants(P.constants)` → `set_Textures(P.T)`. `set_Shader(Shader*)` is
just `set_Element(S->E[0], pass)`.

### Dedup pattern (every setter)

Each `set_X` compares the requested resource to the cached one and **returns early if unchanged**;
otherwise it updates the cache, bumps a stat counter, and issues the D3D context call
(`PSSetShader`/`VSSetShader`/`IASetVertexBuffers`/`IASetIndexBuffer`/…). `set_Geometry(SGeometry*)`
fans out to `set_Format`(declaration) + `set_Vertices`(VB+stride) + `set_Indices`(IB). The matrix
setters (`set_xform_world/view/project`) recompute derived matrices (WV, WVP) and, if the corresponding
constant is bound, push them via `set_c` — i.e. transforms become shader constants
(`R_Backend_xform`, *agent-traced*).

### Deferred uploads + draw

Constants and render-target binds are **not** applied immediately — `set_c` marks the constant's
buffer dirty; `set_RT`/`set_ZB` only set a "changed" flag. The actual GPU sync happens inside
**`CBackend::Render(primitive, …)`** just before the draw:
`ApplyRTandZB` (one `OMSetRenderTargets` if RT/ZB changed) → `ApplyVertexLayout` (find-or-create the
input layout for the current VS signature + declaration) → `StateManager.Apply` (rasterizer/blend/
depth-stencil objects) → **`constants.flush`** (map-discard + copy each *dirty* constant buffer, per
shader stage) → `HW.pContext->DrawIndexed(...)` (or `Draw`, or `Dispatch` for compute). Topology is
forced to a patch list when hull/domain shaders are bound (tessellation).

### Dynamic geometry streams

Transient geometry (UI, particles, wallmarks, LOD batches) uses the `R_DStreams` ring buffers:
`_VertexStream::Lock`/`_IndexStream::Lock` map the dynamic VB/IB with **`D3D_MAP_WRITE_NO_OVERWRITE`
(append)** while space remains, and **`D3D_MAP_WRITE_DISCARD` (wrap, new discard-id)** when the write
won't fit — so the GPU can keep reading already-submitted regions without a stall. `Unlock` advances
the write position.

## Function digests (spine)

- **`R_Backend.cpp:CBackend RCache`** (+ `R_Backend.h:class CBackend`) — *Does:* the single global
  pipeline-state mirror + submitter. *Reads/Writes:* its own cached state. *Used by:* all draw code.
- **`R_Backend_Runtime.h:CBackend::set_Element`** — *Does:* apply a pass (state→PS→VS→GS→HS/DS/CS→
  constants→textures). *Reads:* `SPass`. *Calls:* the `set_*` family. *Called by:* scene-graph + post
  passes. *(verified.)*
- **`dx10R_Backend_Runtime.h:CBackend::set_PS/set_VS/set_GS/set_Vertices/set_Indices`** — *Does:*
  pointer-dedup then the matching `HW.pContext->*Set*`. *Gotcha:* `set_VS` also captures the input
  signature for layout creation. *(verified signatures.)*
- **`dx10R_Backend_Runtime.h:CBackend::set_Constants`** — *Does:* switch the active constant table,
  redistribute its constant buffers to each shader stage, run named-constant handlers. *Gotcha:*
  early-returns if the table pointer is unchanged. *(agent-traced body.)*
- **`dx10R_Backend_Runtime.h:CBackend::Render`** — *Does:* pre-draw flush (RT/ZB, vertex layout, state
  manager, constants) then `DrawIndexed`/`Draw`. *Gotcha:* this is where deferred constants/RT actually
  reach the GPU — the dedup elsewhere pays off here. *(verified.)*
- **`R_DStreams.cpp:_VertexStream::Lock/_IndexStream::Lock`** — *Does:* ring-buffer alloc; append
  (`NO_OVERWRITE`) until wrap, then discard (`DISCARD`). *Called by:* particles/wallmarks/UI/LOD.
  *(verified.)*
- **`R_Backend_xform.cpp:R_xforms::set_W/set_V/set_P`** — *Does:* update matrix + derived (WV/WVP) and
  upload bound matrix constants via `set_c`. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** the cached pipeline mirror — current VS/PS/GS/HS/DS/CS, textures/samplers, render
  state, constant registers/buffers, geometry+declaration, world/view/proj. The single `RCache`
  instance.
- **Callers (in):** `flows/renderer-scene-graph.md` (`r_dsgraph_render_graph`), full-screen post
  (`flows/renderer-post-process.md`), per-visual `dxRender_Visual::Render`, and the engine
  `dx*Render` bridges (UI/font/console/debug — future F61).
- **Resource seam:** `ShaderElement`/`SPass`/`R_constant_table`/`STextureList` come from
  `flows/renderer-resources-shaders.md`.
- **HW seam:** `HW.pContext` (`ID3D11DeviceContext`) for every bind/draw; `HW.pDevice` for creating
  input layouts + state objects.
- **Sub-caches:** `dx10StateManager` (dedups rasterizer/blend/depth-stencil objects + `Apply`),
  the SRV/sampler state cache, and the DX11 constant-buffer cache (`dx10ConstantBuffer` +
  `dx10r_constants_cache`) — `RCache` holds high-level state, delegates object dedup to these.
- **Threading:** main render thread (the device context is single-threaded here).

## Threading

All submission is on the main render thread inside the frame; `RCache` is not thread-safe and is not
meant to be touched off-thread.

## Failure & edge cases

- **Deferred uploads** mean a constant/RT set has no effect until the next `Render` — debugging "my
  constant didn't apply" usually means no draw happened, or a later `set_Constants` swapped the table.
- **Pointer-identity dedup** — two distinct-but-equal resources won't dedup at the backend (they were
  already pooled by the resource manager; see `flows/renderer-resources-shaders.md`).
- **Dynamic-stream wrap** discards the whole buffer — never hold a pointer across a `Lock`.
- **Tessellation** forces patch-list topology when HS/DS are set — a wrong primitive type with a hull
  shader bound is a mismatch symptom.

## Deeper

- Class + global: `xrRender/R_Backend.h`/`.cpp` (`CBackend`, `RCache`)
- Apply + draw: `xrRender/R_Backend_Runtime.h` (`set_Element`, matrix setters),
  `xrRenderDX10/dx10R_Backend_Runtime.h` (DX11 `set_*`, `Render`, `set_Constants`, `ApplyRTandZB`,
  `ApplyVertexLayout`)
- Constants: `xrRender/r_constants_cache.h`, `xrRenderDX10/dx10r_constants_cache.cpp`,
  `dx10ConstantBuffer.cpp`; transforms: `xrRender/R_Backend_xform.cpp`
- Dynamic streams: `xrRender/R_DStreams.cpp`; state objects: `xrRenderDX10/StateManager/dx10StateManager.cpp`
- Callers: `flows/renderer-scene-graph.md`, `flows/renderer-post-process.md`; resources:
  `flows/renderer-resources-shaders.md`

## Maintenance

- `set_Element` + `Render` are the spine — re-grep them if the apply order or pre-draw flush changes.
- Promote the *(agent-traced)* `set_Constants` cbuffer redistribution, `dx10StateManager::Apply`, and
  `R_Backend_xform` matrix upload to verified by reading those bodies when a constant/state task needs
  exact behavior.
- Keep the "deferred until Render" + "pointer-identity dedup" facts — they explain the most common
  backend confusions.
