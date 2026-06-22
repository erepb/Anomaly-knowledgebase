---
domain: flows/renderer-resources-shaders
tags: [engine, renderer, shaders, r4, dx11, blender, flow]
sources: [src/Layers/xrRender/ResourceManager.cpp, src/Layers/xrRender/ResourceManager.h, src/Layers/xrRender/ResourceManager_Loader.cpp, src/Layers/xrRender/ResourceManager_Resources.cpp, src/Layers/xrRender/ResourceManager_Reset.cpp, src/Layers/xrRender/ResourceManager_Scripting.cpp, src/Layers/xrRender/Shader.cpp, src/Layers/xrRender/Shader.h, src/Layers/xrRender/ShaderResourceTraits.h, src/Layers/xrRenderDX10/dx10ResourceManager_Resources.cpp]
related: [engine/renderer.md, flows/render-frame.md, flows/renderer-blender-compile.md, flows/renderer-backend.md, flows/renderer-scene-graph.md, modlayer/assets.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: renderer resource manager — shader/texture/constant load & dedup

How a *blender* (a C++/Lua pass description) plus HLSL shader files and texture names become a usable
`Shader` object, and how every render resource is **deduplicated** (created once, shared by pointer).
`CResourceManager` is the single owner of all render resources; the renderer reaches it as
`RImplementation.Resources` / the global `DEV`. This is the supply side of `flows/render-frame.md`
(the frame consumes the `ShaderElement`s built here) and the deeper version of `engine/renderer.md`'s
"blenders vs shaders" section.

**Provenance:** researched with a read-only agent, then spot-verified against code. The create/dedup
spine (`Create` fork, `_cpp_Create` 6-element loop, `_CreateElement`/`_CreateShader`,
`_CreateVS`/`_CreatePS`/`_CreateGS` file-load+compile, the ownership maps/vectors, `creationGuard`) is
personally code-verified. Blender `_cpp_Compile` internals, constant-table reflection, and the
DX10/DX11 texture surface load are *(agent-traced — not personally re-read)*.

## Trigger / entry

A model/material/full-screen pass needs a shader. Game/engine code calls
`CResourceManager::Create(s_shader, s_textures, s_constants, s_matrices)` (string form) or the
`Create(IBlender*, …)` form (precompiled C++ blender). `s_shader` is a blender name (e.g.
`"models\\selflight"`); `s_textures` is a comma-separated texture list.

## Sequence (verified)

1. **`CResourceManager::Create(LPCSTR …)`** forks: if `_lua_HasShader(s_shader)` → `_lua_Create`
   (the `.s` Lua-scripted blender path), else `_cpp_Create` (the C++ blender path). If `_cpp_Create`
   returns null it falls back to the `stub_default` Lua shader; if that is missing too it `FATAL`s
   ("Can't find stub_default.s"). Dedicated-server builds return null (no shaders).
2. **`_cpp_Create(LPCSTR …)`** resolves the blender template by name — `_GetBlender(s_shader)` (from
   the `m_blenders` map loaded at device create) — then calls the `IBlender*` overload.
3. **`_cpp_Create(IBlender* B, …)`** is the spine: takes `creationGuard`, builds a `CBlender_Compile C`
   with `C.BT = B`, parses the three name lists (`_ParseList` → `C.L_textures/L_constants/L_matrices`),
   then **compiles 6 shader elements** (`C.iElement` 0..5 — the LOD/lighting-mode variants; element 0/1
   fetch a detail texture via `m_textures_description.GetDetailTexture`). For each: `C._cpp_Compile(&E)`
   fills a temp `ShaderElement E` with passes, then `_CreateElement(E)` dedups it into `S.E[i]`.
4. **`_CreateElement(ShaderElement&)`** — returns null if no passes; else linear-searches `v_elements`
   for an `equal` element and returns the cached one, otherwise `xr_new`s it, flags `RF_REGISTERED`,
   pushes to `v_elements`. (Same find-or-create shape as every `_Create*`.)
5. Back in `_cpp_Create`, after the 6 elements are built into a temp `Shader S`, it linear-searches
   `v_shaders` for an `equal` shader; if none, **`_CreateShader(&S)`** clones it, flags `RF_REGISTERED`,
   pushes to `v_shaders`. The returned `Shader*` is shared — identical material requests get the same
   pointer.
6. **Blender compile → passes → atomic resources** *(agent-traced for the `CBlender_Compile` body)*:
   each blender's `Compile(CBlender_Compile&)` declares passes (`r_Pass(vs, ps, …)`), texture/sampler
   bindings, and state; pass finalization dedups every leaf via the `_Create*` family:
   - **`_CreateVS`/`_CreatePS`/`_CreateGS`** (`dx10ResourceManager_Resources.cpp`) — dedup by name in
     `m_vs`/`m_ps`/the GS map; on miss build the path `::Render->getShaderPath() + name + ".vs"/".ps"/".gs"`,
     `FS.r_open` it (falling back to `stub_default` if absent), detect entry/target, then
     `::Render->shader_compile(name, data, size, entry, target, flags, handle)` → the D3D
     `Create*Shader`. HS/DS/CS go through the `ShaderResourceTraits` template (`.hs/.ds/.cs`,
     `*_5_0` targets).
   - **`_CreatePass`/`_CreateConstantTable`/`_CreateTextureList`/`_CreateConstantList`/`_CreateState`/
     `_CreateDecl`** — composite dedups (by value); **`_CreateTexture`** dedups by name in `m_textures`,
     `Preload`s `.thm` metadata then `Load`s the surface (async task group). *(bodies agent-traced.)*

So the model is a **bottom-up find-or-create tree**: atomic (textures, compiled VS/PS/GS, states) →
composite (passes, constant/texture lists, constant tables) → `ShaderElement` (6 per shader) →
`Shader`. Everything is shared by pointer; `equal()` value-comparison is what collapses duplicates.

## Function digests (spine)

- **`ResourceManager.cpp:CResourceManager::Create(LPCSTR…)`** — *Does:* top entry; lua-vs-cpp fork +
  stub_default fallback + FATAL. *Calls:* `_lua_HasShader`, `_lua_Create`, `_cpp_Create`. *Called by:*
  model/material binding, full-screen pass setup.
- **`ResourceManager.cpp:CResourceManager::_cpp_Create(IBlender*,…)`** — *Does:* the spine; compiles 6
  elements under `creationGuard`, dedups into a `Shader`. *Reads:* `m_textures_description`. *Writes:*
  `v_elements`, `v_shaders` (via `_Create*`). *Calls:* `_ParseList`, `CBlender_Compile::_cpp_Compile`
  (×6), `_CreateElement` (×6), `_CreateShader`.
- **`ResourceManager.cpp:CResourceManager::_CreateElement` / `_CreateShader`** — *Does:* find-or-create
  by `equal()`; flag `RF_REGISTERED`; push to `v_elements`/`v_shaders`. *Gotcha:* linear search — dedup
  cost is O(n) in the table size, paid at load not per-frame.
- **`dx10ResourceManager_Resources.cpp:CResourceManager::_CreateVS/_CreatePS/_CreateGS`** — *Does:*
  name-dedup then file-load+compile a shader stage. *Reads:* `::Render->getShaderPath()`, FS.
  *Calls:* `FS.r_open`, `::Render->shader_compile`. *Gotcha:* missing file silently falls back to
  `stub_default` — a "wrong shader" symptom can be a missing `.ps`/`.vs`, not a logic bug.
- **`ResourceManager_Loader.cpp:CResourceManager::OnDeviceCreate`** — *Does:* loads the blender library
  + constants + matrices from `shader.xr`, inits the Lua VM (`LS_Load`) and texture descriptions.
  *Writes:* `m_blenders`, `m_textures_description`. *Called by:* device create.
- **`ResourceManager_Reset.cpp:reset_begin/reset_end`** — *Does:* release/recreate device-bound
  resources (states, render targets, dynamic streams) around a device reset/resize.

## Isolation ledger (the seams)

- **Owned state:** `m_blenders`, `m_textures`, `m_vs`, `m_ps` (+ GS/HS/DS/CS maps); vectors
  `v_states`, `v_constant_tables`, `v_passes`, `v_elements`, `v_shaders`; `m_textures_description`
  (`CTextureDescrMngr`). All find-or-create, all dedup by `equal()`/name.
- **Thread safety:** every create path takes `creationGuard` (`xrCriticalSection`); texture surface
  loads are deferred to an async task group.
- **Blender seam:** `IBlender::Compile(CBlender_Compile&)` — the per-material contract; the full
  blender→pass mechanism is `flows/renderer-blender-compile.md` (C++ blenders in
  `xrRenderPC_R4/blender_*.cpp`). `.s` Lua blenders go through `ResourceManager_Scripting.cpp`
  (`adopt_compiler`/`adopt_sampler` luabind surface).
- **Backend seam:** `::Render->shader_compile` → D3D `Create{Vertex,Pixel,Geometry,…}Shader`; the
  resulting handles are consumed by `CBackend`/`RCache` (`flows/render-frame.md`, and a future
  R_Backend flow).
- **File system:** `shader.xr` (blender lib), `$game_shaders$/*.{vs,ps,gs,hs,ds,cs,h}` (HLSL +
  includes), `$game_textures$/*.{dds,tga,…}` + `.thm` metadata. Shader-path folder = `getShaderPath()`
  (`r3\` for R3+R4 — see `engine/renderer.md`).
- **Config:** `RImplementation.o.ssfx_core` + `::Render->hud_loading` gate the HUD-element compile
  variant; `g_dedicated_server` skips shader creation entirely.

## Threading

Creation is serialized by `creationGuard`; most shader creation happens during level load (HUD
loading, model load), not per-frame. Texture pixel data loads asynchronously.

## Failure & edge cases

- **Missing `.s`/`stub_default.s` → FATAL**; missing `.ps`/`.vs`/`.gs` → silent `stub_default`
  fallback (wrong-looking material, no crash).
- **Dedup is by value** — two materials with identical passes/textures share one `Shader*`; editing a
  texture name changes the dedup key and forks a new shader.
- **Dedicated server** has no resources — guard every renderer-resource assumption against it.

## Deeper

- Create/dedup spine: `xrRender/ResourceManager.cpp` (`Create`, `_cpp_Create`, `_CreateElement`,
  `_CreateShader`); ownership in `ResourceManager.h`
- Shader stage load/compile: `xrRenderDX10/dx10ResourceManager_Resources.cpp`
  (`_CreateVS`/`_CreatePS`/`_CreateGS`), `xrRender/ShaderResourceTraits.h` (HS/DS/CS template)
- Object model: `xrRender/Shader.h` (`Shader`/`ShaderElement`/`SPass`), `SH_*.cpp` (atomic resources)
- Blender library load: `xrRender/ResourceManager_Loader.cpp` (`OnDeviceCreate`); Lua bind:
  `ResourceManager_Scripting.cpp`
- Device reset: `xrRender/ResourceManager_Reset.cpp`
- Renderer mechanism (blender/shader-path): `engine/renderer.md`; on-disk shaders: `modlayer/assets.md`

## Maintenance

- The create fork is the source of truth — re-grep `CResourceManager::Create` and `_cpp_Create` if
  shader creation is restructured.
- The blender `_cpp_Compile`/`r_Pass` mechanism is now its own flow (`flows/renderer-blender-compile.md`);
  promote its remaining *(agent-traced)* parts + constant-table reflection (`r_constants.cpp`
  `R_constant_table::parse`) to verified there when a shader-binding task needs it.
- If `getShaderPath()` or the file extensions change, update here + `engine/renderer.md`.
