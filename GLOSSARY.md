---
domain: meta
tags: [meta, engine, modlayer, dltx, dxml, unlocalizer, ltx, callbacks, renderer]
sources: [src/xrCore/Xr_ini.cpp, src/Layers/xrRenderPC_R4/r4.h, src/xrServerEntities/script_engine_export.cpp, gamedata/scripts/dxml_core.script, README.md, DXML.md]
related: [INDEX.md, CONVENTIONS.md, modlayer/config-ltx.md, modlayer/dxml.md, engine/renderer.md]
updated: 2026-06-05
verified: code
---

# Glossary

Concepts specific to this repo (X-Ray monolith engine + Anomaly "modded exes" mod layer). Read this
first for any task. Anchors below are linkable from other KB files (e.g. `GLOSSARY.md#dltx`).

## Project shape

*(This section is from the README + `git branch`, not code-derived.)*

- **X-Ray monolith** — the C++ engine in `src/`. A community fork of the X-Ray 1.6 engine producing
  patched Anomaly 1.5.3 executables. See `engine/_overview.md`.
- **modded exes** — the distributed product: patched engine `.exe`s + the supporting `gamedata/`
  overlay shipped in this repo. The README's "List of patches" is the authoritative feature list.
- **Anomaly 1.5.3** — the standalone S.T.A.L.K.E.R. game these exes target. The full game (base
  configs/scripts/assets) is NOT in this repo; only the mod's additions/overrides are.
- **MT / wpo / vulkan** — build variants (branches). MT = multithreaded performance build
  (`...-mt`); wpo = whole-program optimization (`...-wpo`); vulkan = experimental Vulkan backend.
  Branch names like `all-in-one-vs2022-wpo` encode the variant.

## File-layer concepts

- <a id="gamedata"></a>**gamedata / db / mods** — `gamedata` is a verified FS entry point
  (`LocatorAPI.cpp`). This repo ships only the modded-exes `gamedata/` overlay. **Override
  mechanism verified**: the file registry `m_files` dedups by path descriptor and **last registration
  wins** — `Register` overwrites an existing entry in place (`const_cast<file&>(*I) = desc`,
  `LocatorAPI.cpp`); both loose files and archive contents go through the same registry
  (`ProcessOne`/`ProcessArchive`). *(INFERRED — that loose `gamedata/` is registered
  after packed `db/` archives so it wins; the concrete scan order depends on `fs.ltx` + directory
  traversal, not traced.)* See `modlayer/_overview.md`.
- <a id="unpacked"></a>**unpacked/** — read-only reference copy of base-game assets/scripts, kept for
  lookup. **Never edit.** Editable runtime content is `gamedata/`.
- <a id="ltx"></a>**LTX / ini** — the engine's config format (`.ltx`). Sectioned key=value, with
  `#include`, section inheritance via `:parent`. Parsed in `src/xrCore/Xr_ini.cpp`. See
  `modlayer/config-ltx.md`.
- <a id="dltx"></a>**DLTX** — Delta-LTX, a non-destructive config patching layer (by MerelyMezz,
  edited by demonized). Mods drop `mod_<file>_<name>.ltx` files that the engine merges into the base
  `.ltx` at load. Section operators **merge key-by-key onto the base, not replace it**: `![section]`
  = override fragment for an existing section, `@[section]` = same but also creates the section if
  missing; listed keys replace the base's value, unmentioned base keys are kept, new keys added.
  Value operators: `>key = a,b` append to a list, `<key = a` remove from a list, `key = DLTX_DELETE`
  delete the key. Implemented in `src/xrCore/Xr_ini.cpp` (`MergeSections`/`EvaluateSection`; grep
  `DLTX`, `FS.file_list(... "mod_"...)`). Full flow: `flows/dltx-config-load.md`.
- <a id="dxml"></a>**DXML** — Lua-side XML manipulation (by demonized). Lets mods transform XML files
  before the engine parses them, via `modxml_<name>.script` files registering the `on_xml_read`
  callback. The engine hands Lua an `xml_obj` DOM. Full guide: `DXML.md`. See `modlayer/dxml.md`.
- <a id="xml_obj"></a>**xml_obj / DOM** — the DOM-like Lua object DXML exposes; methods include
  `query`, `getText`, `setText`, `setElementAttr`, `add`, `insertFromXMLString`,
  `insertFromXMLFile`. Defined in `gamedata/scripts/dxml_core.script` (uses `slaxml.script`).
- <a id="unlocalizer"></a>**unlocalizer** — preprocessor that promotes selected `local` variables in
  a `.script` to script-global scope before load, so other code can reach them. Configured via
  `gamedata/configs/unlocalizers/unlocalizer_*.ltx` (section = script filename, keys = locals to
  unlocalize). See `modlayer/unlocalizer.md`.

## Scripting concepts

- <a id="callbacks"></a>**callbacks / RegisterScriptCallback** — Anomaly's Lua event system. Modules
  register handlers with `RegisterScriptCallback("on_event", fn)` and the engine/scripts fire them
  via `SendScriptCallback`. Game-object-specific callbacks live in
  `gamedata/scripts/callbacks_gameobject.script`. The manager is `axr_main.script`. See
  `modlayer/scripting.md`.
- <a id="axr_main"></a>**axr_main** — the module/callback manager (by Alundaio). Auto-runs any script
  exposing `on_game_start`, wiring the callback system. `gamedata/scripts/axr_main.script`.
- <a id="hot-reload"></a>**hot reload** — two distinct things. (1) *Engine* hot-reload: a VS2022
  Edit-and-Continue build workflow for C++ (README "Working with Hot Reload"; not a single traceable
  symbol). (2) *Lua* script reload: verified engine path — `CScriptEngine::process_file(name,
  reload_modules=true)` (`script_engine.cpp`) forces `load_file_into_namespace` even for an
  already-loaded namespace, re-executing the chunk. See `modlayer/scripting.md`.
- <a id="options"></a>**options framework** — the modded-exes settings UI. `options_builder.script`
  builds option trees; `options_modded_exes_*.script` (28 files) declare per-feature options;
  `ui_options_modded_exes.script` renders them. See `modlayer/options-ui.md`.
- <a id="luabind"></a>**luabind** — C++↔Lua binding library. Verified: `export_classes()` in
  `script_engine_export.cpp` registers classes via per-class `script_register(L)`; the engine is
  `CScriptEngine` (`src/xrServerEntities/script_engine.cpp`). *(INFERRED — README: "updated luabind".)*
- <a id="ssrs"></a>**SSRS** — a popular AI/combat mod; some engine params (e.g. `ai_rpm`) exist
  specifically to support it. Referenced in `ltx_help_ex.script`.

## Renderer concepts

- <a id="r1-r4"></a>**R1 / R2 / R3 / R4** — render path generations, code in
  `src/Layers/xrRenderPC_R1..R4/` (class `CRender` per path, e.g. `r4.h`), shared in
  `src/Layers/xrRender/`. Verified: R1 is the static path (`FStaticRender.cpp`, uses `r1\` shader
  folder); R3 has DX10 MSAA modes (`r3.h`); R4 has DX10/DX11 modes (`r4.h`); the D3D backend
  `xrRenderDX10` mixes D3D10/D3D11 types (`DXCommonTypes.h`), i.e. it serves both R3 and R4. R2
  includes `<d3dx9.h>` (`xrRenderPC_R2/stdafx.h`) → DX9. R1's static/vertex-lighting model is
  corroborated by `xrCPU_Pipe`'s CPU point-light vertex calc (`PLC.cpp`). *(INFERRED — that R2+ do
  per-pixel *deferred dynamic* lighting is the conventional description; those passes weren't read.)*
  See `engine/renderer.md`.
- <a id="dxn"></a>**DX9 / DX10 / DX11** — Direct3D backends in `src/Layers/xrRenderDX9`,
  `xrRenderDX10` (verified: `DXCommonTypes.h` mixes D3D10/D3D11 types → backs both R3 and R4). The
  active path is fixed at **compile time** in this all-in-one build (`CEngineAPI` `#error` guards,
  `EngineAPI.cpp`), not selected at runtime.
- <a id="blender"></a>**Blender (shader)** — verified: a C++ class deriving `IBlender` with a
  `Compile(CBlender_Compile&)` method that configures a render pass (`CBlender_blur` in
  `blender_blur.h`), not the 3D app. Files: `blender_*.cpp`/`Blender_*.cpp`.
- <a id="shaders"></a>**shaders (HLSL)** — sources under `gamedata/shaders/{r1,r2,r3}`. Verified: R3
  and R4 both load from `r3\` (`getShaderPath()` in `r3.h`/`r4.h`). This repo's `r3` ships only
  `.ps/.vs/.s/.h` (verified by listing); the engine also supports `.gs/.cs` + hull/domain. `.h` =
  shared includes; `.s` = script-side blender config. See `engine/renderer.md`.
- <a id="cform"></a>**CFORM** — collision geometry format for level meshes. Built/queried via
  `src/xrCDB/` (collision database). See `engine/physics-collision.md`.
- <a id="spom"></a>**SPOM** — Silhouette Parallax Occlusion Mapping, an in-progress rendering feature
  in this repo (cross-Fvisual buffer + manifold check + hybrid shader path). Design docs in
  `claudedocs/`; state in the SPOM auto-memory. See `features/spom.md`.

## Engine core concepts

- <a id="shared_str"></a>**shared_str** — interned, ref-counted, pooled string type. Verified
  (`xrstring.h` + `xrstring.cpp` `str_container::dock`): `intrusive_ptr<str_container>`, global
  intern pool `g_pStringContainer`, `pool_block`/`alloc_in_pool`; interning hashes with `xr_hash`
  (= `robin_hood::hash`) into a custom SRW-locked bucket table (shared lock for lookup, exclusive +
  double-check for insert — the MT-safe rework). The intern table is custom (not a robin_hood map);
  `xr_unordered_map/set` elsewhere *are* `robin_hood::unordered_*` (`_stl_extensions.h`).
- <a id="smart_cast"></a>**smart_cast** — downcast used instead of `dynamic_cast`. Verified:
  `src/xrServerEntities/smart_cast.h` — template `has_dcast` specializations (e.g.
  `IRenderVisual`↔`IKinematics` via `dcast_*` methods) with a `fast_dynamic_cast` library fallback.
- <a id="alife"></a>**ALife / simulation** — server-side entities live in `src/xrServerEntities/`
  (`CSE_*` classes, `alife_*_brain.h`); scripted squads via
  `gamedata/scripts/sim_squad_scripted.script`. Offline simulation verified: `CALifeMonsterBrain::update`
  (`alife_monster_brain.cpp`) runs `select_task` → `process_task` (when assigned a smart_terrain,
  `m_smart_terrain_id != 0xffff`) else `default_behaviour`, then `movement().update()`;
  `on_switch_online`/`on_switch_offline` flip the representation. The switch is driven by actor
  proximity with a hysteresis band (`switch_distance`/`switch_factor` from the `"alife"` section) —
  verified in `flows/alife-online-offline.md`.
- <a id="smart_terrain"></a>**smart_terrain** — ALife location object that controls NPC jobs/spawns
  in an area (gameplay-layer concept, mostly base-game scripts not in this repo).
- <a id="scheduler"></a>**scheduler** — `class ENGINE_API CSheduler` (`xrEngine/xrSheduler.h`).
  Verified: objects implement `ISheduled` and `Register(ISheduled*, BOOL RT)` with a real-time flag;
  `Update` splits into `UpdateRT()` (every RT item each frame) vs `UpdateDeferred()` (time-budgeted).
  Verified in `xrSheduler.cpp`: `UpdateDeferred`→`ProcessStep` collects a batch from the
  priority queue under an SRW lock (`ItemsLock`) and runs it **serially** with a time check every 8th
  item — the lock guards registration, execution is single-threaded (no worker-thread batch dispatch).
  Driven per frame from `device.cpp` (`seqFrame.Process`).

## Misc

- <a id="hud_offset"></a>**hud_offset** — weapon first-person model position/rotation tuning params
  (`base_hud_offset_pos/rot`, `_16x9`). Extra params documented in
  `gamedata/scripts/ltx_help_ex.script`.
- <a id="gamma-efp"></a>**GAMMA / EFP** — large Anomaly modpacks the exes are tested against /
  support. Not in this repo.
- <a id="sdk"></a>**SDK** — level/asset authoring tools under `sdk/`. Separate from runtime engine.

## Maintenance

- When a new repo-specific concept appears (new operator, new callback family, new render path),
  add an anchored entry here and tag it in `CONVENTIONS.md`.
- Re-verify against `README.md` "List of patches" periodically — it is the upstream source of truth
  for features; this glossary condenses it.
- Spot-check anchors referenced by other KB files still exist after edits.
