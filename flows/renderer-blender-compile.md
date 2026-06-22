---
domain: flows/renderer-blender-compile
tags: [engine, renderer, shaders, blender, r4, dx11, flow]
sources: [src/Layers/xrRender/blenders/Blender.cpp, src/Layers/xrRender/blenders/Blender.h, src/Layers/xrRender/blenders/Blender_CLSID.h, src/Layers/xrRender/blenders/Blender_Recorder.cpp, src/Layers/xrRender/Blender_Recorder_R2.cpp, src/Layers/xrRenderDX10/Blender_Recorder_R3.cpp, src/Layers/xrRender/r2_blenders.cpp, src/Layers/xrRenderPC_R4/blender_deffer_flat.cpp]
related: [flows/renderer-resources-shaders.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: blender → pass compilation

How a *blender* (a C++ class describing a render technique) emits the concrete passes of a
`ShaderElement`: the `Compile(CBlender_Compile&)` contract, the `CBlender_Compile` recorder
(`r_Pass`/`r_End`/binders), and the CLSID registry that maps a blender name to its class. This is the
producer side of `flows/renderer-resources-shaders.md` — `CResourceManager::_cpp_Create` calls
`CBlender_Compile::_cpp_Compile(&E)` per element, and each blender's `Compile` fills `E`. It deepens
`engine/renderer.md`'s "blenders vs shaders" section from one worked example to the full mechanism.

**Provenance:** researched with a read-only agent, then spot-verified against code. The CLSID factory
(`CRender::blender_create`), name lookup (`_GetBlender`), recorder spine (`_cpp_Compile`, `r_Pass`
R2-vs-DX10 overloads, `r_End`, `r_dx10Texture`/`r_dx10Sampler`), and the `Compile`/`iElement` pattern
are personally code-verified. The per-pass render-state setters (`PassSET_*`/`StageSET_*`) and the
`uber_deffer`/`uber_shadow` helper bodies are *(agent-traced — not personally re-read)*.

## Trigger / entry

`CResourceManager::_cpp_Create` (`flows/renderer-resources-shaders.md`) needs a blender by name
(`_GetBlender(name)` → the `m_blenders` map) and runs its `Compile` six times (one per `iElement`
LOD/lighting variant), once per `ShaderElement`.

## Sequence (verified)

1. **Name → blender:** `CResourceManager::_GetBlender(LPCSTR Name)` returns the `IBlender*` registered
   under that name in `m_blenders`. The map was populated at device create from `shader.xr`: each
   serialized `CBlender_DESC` carries a `CLASS_ID`; `IBlender::Create(cls)` →
   `CRender::blender_create(cls)` (`r2_blenders.cpp`) is the **factory switch** mapping CLSID → concrete
   class (e.g. `B_DEFAULT`→`CBlender_deffer_flat`, `B_MODEL`→`CBlender_deffer_model`,
   `B_DEFAULT_AREF`→`CBlender_deffer_aref`). **Many legacy CLSIDs return `0`** (`B_LIGHT`,
   `B_SHADOW_*`, `B_BLUR`, …) — those are R1/R2 techniques with no R4 implementation.
2. **`CBlender_Compile::_cpp_Compile(ShaderElement*)`** sets the target element, resolves detail-texture
   applicability (`m_textures_description.GetDetailTexture` + the blender's `canBeDetailed`/
   `canUseSteepParallax`), then calls **`BT->Compile(*this)`** — the blender now records its passes
   into this recorder. It does not finalize the element; the blender's `r_End` calls do.
3. **Inside `Compile`** the blender switches on `C.iElement` (the 6 variants — e.g. normal-HQ,
   normal-LQ, shadow-map) and for each builds one or two passes:
   - **`C.r_Pass(vs, [gs,] ps, fog, ztest, zwrite, ablend, src, dst, atest, aref)`** begins a pass:
     resets per-pass lists, configures render state (`PassSET_ZB`/`PassSET_Blend`/`PassSET_LightFog`
     into the state recorder `RS`), creates the stage shaders (`_CreateVS`/`_CreatePS`/`_CreateGS` from
     `flows/renderer-resources-shaders.md`), and merges their reflected constants into `ctable`. The
     **R2 overload has no geometry shader**; the **DX10/DX11 overload (`Blender_Recorder_R3.cpp`) adds
     the GS** parameter.
   - **Bind textures/samplers:** `C.r_dx10Texture(resourceName, texturePath)` resolves the HLSL
     resource name to a stage via `ctable` and appends `{stage, _CreateTexture(path)}`;
     `C.r_dx10Sampler(resourceName)` resolves the sampler stage and sets filter/address from the
     sampler-name convention (`smp_linear`/`smp_base`/`smp_smap`/…).
   - **`C.r_End()`** finalizes the pass: builds the dedup'd state (`_CreateState`), texture list
     (`_CreateTextureList`), constant table (`_CreateConstantTable`), assembles the `SPass` and dedups
     it (`_CreatePass`), then appends to `SH->passes` (max 2 per element).
4. Back in the resource manager, the filled `ShaderElement` is dedup'd (`_CreateElement`) and stored as
   `Shader.E[iElement]` (`flows/renderer-resources-shaders.md`).

So a blender is a **small declarative program**: `switch(iElement) → r_Pass … bind … r_End`, and the
recorder turns those calls into deduplicated render resources. The `.s` Lua-blender path is the
scripted alternative (`ResourceManager_Scripting.cpp` `adopt_compiler`), driving the same recorder.

## Worked example (verified entry, helper bodies agent-traced)

`CBlender_deffer_flat::Compile` (`blender_deffer_flat.cpp`, the `B_DEFAULT`/world-geometry blender):
calls `IBlender::Compile(C)` (sets priority/sorting), sets `C.TessMethod`, then per `iElement` —
normal-HQ/LQ elements run the `uber_deffer` helper (G-buffer fill: base diffuse/normal + stencil mark)
and the shadow element runs `uber_shadow` (depth-only, color-write disabled). Each helper internally
does `r_Pass … r_dx10Texture/r_dx10Sampler … r_End`. *(helper bodies agent-traced.)*

## Function digests (spine)

- **`r2_blenders.cpp:CRender::blender_create(CLASS_ID)`** — *Does:* CLSID→concrete-blender factory
  switch (`xr_new<CBlender_*>`); returns `0` for unimplemented (R1/R2-only) techniques. *Called by:*
  `IBlender::Create` during `shader.xr` load.
- **`ResourceManager.cpp:CResourceManager::_GetBlender(LPCSTR)`** — *Does:* name→`IBlender*` lookup in
  `m_blenders`. *Called by:* `_cpp_Create`.
- **`Blender_Recorder.cpp:CBlender_Compile::_cpp_Compile(ShaderElement*)`** — *Does:* set target +
  detail-texture resolution, then invoke `BT->Compile(*this)`. *Reads:* `m_textures_description`,
  `RImplementation.o.*`. *Writes:* the `ShaderElement` (via the blender). *Called by:* `_cpp_Create`
  (×6).
- **`Blender_Recorder_R2.cpp` / `Blender_Recorder_R3.cpp:CBlender_Compile::r_Pass`** — *Does:* begin a
  pass; create stage shaders + merge constants + set render state. *Calls:* `_CreateVS/_CreatePS/
  _CreateGS`. *Gotcha:* two overloads (R2 = vs+ps, DX10+ = vs+gs+ps).
- **`Blender_Recorder_R2.cpp:CBlender_Compile::r_End`** — *Does:* finalize pass → dedup state/texlist/
  ctable/pass → append to element. *Calls:* `_CreateState`, `_CreateTextureList`,
  `_CreateConstantTable`, `_CreatePass`.
- **`Blender_Recorder_R3.cpp:CBlender_Compile::r_dx10Texture/r_dx10Sampler`** — *Does:* resolve HLSL
  resource name → stage via `ctable`; bind texture / set sampler state.

## Isolation ledger (the seams)

- **Up to `CResourceManager`** (`flows/renderer-resources-shaders.md`): `r_End`/`r_Pass` call the
  `_Create*` dedup family; `_cpp_Create` drives the 6-element loop and dedups the element + shader.
- **Blender library:** `shader.xr` (loaded at `OnDeviceCreate`) supplies `CBlender_DESC` records;
  CLSIDs are defined in `Blender_CLSID.h`; the factory is `r2_blenders.cpp` (per render path — R4 here).
- **Recorder state:** `CBlender_Compile` holds `BT` (template), `iElement`, the name lists,
  `ctable` (accumulating reflected constants), the render-state recorder `RS`, and the per-pass
  texture/sampler/constant lists.
- **Config read during compile:** `RImplementation.o.*` (ssfx/detail/msaa toggles), `::Render->hud_loading`
  + `o.ssfx_core` (HUD element variant), `TessMethod` (DX11 tessellation).
- **Lua alternative:** `.s` blenders via `ResourceManager_Scripting.cpp` (`adopt_compiler`/`adopt_sampler`).
- **Down to backend:** the `SPass`/`ShaderElement` produced here is what `CBackend::set_Element`
  applies at draw time (`flows/renderer-backend.md`).

## Threading

Compilation runs under `CResourceManager::creationGuard` (serialized), mostly at level/HUD load.

## Failure & edge cases

- **CLSID returns 0** for legacy techniques in R4 — a `null` blender from `blender_create` means the
  technique isn't implemented on this path, not an error per se.
- **R2 vs DX10 `r_Pass` overload** — geometry-shader passes only exist on the DX10/DX11 recorder.
- **Sampler/texture name must exist in `ctable`** (i.e. the HLSL must declare it) or the bind resolves
  to no stage — a "texture not bound" symptom is usually a name mismatch with the shader.

## Deeper

- Factory + CLSIDs: `xrRender/r2_blenders.cpp` (`blender_create`), `xrRender/blenders/Blender_CLSID.h`
- Base + name lookup: `xrRender/blenders/Blender.cpp`/`.h` (`IBlender::Compile`/`Create`),
  `xrRender/ResourceManager.cpp` (`_GetBlender`)
- Recorder: `xrRender/blenders/Blender_Recorder.cpp` (`_cpp_Compile`), `xrRender/Blender_Recorder_R2.cpp`
  (`r_Pass`/`r_End` R2), `xrRenderDX10/Blender_Recorder_R3.cpp` (`r_Pass`/`r_End`/`r_dx10Texture`/
  `r_dx10Sampler` DX10+)
- Example blenders: `xrRenderPC_R4/blender_*.cpp` (`blender_deffer_flat` worked above; post blenders in
  `flows/renderer-post-process.md`)
- Consumers: `flows/renderer-resources-shaders.md` (dedup), `flows/renderer-backend.md` (apply)

## Maintenance

- The factory switch (`r2_blenders.cpp`) is the source of truth for which technique a CLSID maps to —
  re-grep it if blenders are added/removed.
- Promote the *(agent-traced)* `uber_deffer`/`uber_shadow` + `PassSET_*` bodies to verified by reading
  `uber_deffer.cpp` + `Blender_Recorder_R2.cpp` (`r_Pass`/`r_End` in full) when a pass-construction
  task needs it.
- If the recorder API (`r_Pass`/`r_End`/binders) changes, update here + `engine/renderer.md`.
