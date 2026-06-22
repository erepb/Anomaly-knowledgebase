---
domain: engine
tags: [engine, cpp, build, mt]
sources: [src/engine-vs2022.sln, src/xrEngine/EngineAPI.cpp, src/xrGame/xrGame.vcxproj, CMakeLists.txt]
related: [INDEX.md, GLOSSARY.md, engine/xrGame.md, engine/xrCore.md, engine/renderer.md, engine/server-entities.md]
updated: 2026-06-05
verified: code
---

# Engine overview (src/)

The C++ X-Ray monolith engine. A layered set of static libs/DLLs built into the Anomaly exe. Start
here before any engine change to find the right module.

## Module map (file counts verified via `find`; roles per linked domain files)

- **xrGame/** (~2686 files) — the gameplay DLL and by far the largest. Actor, weapons, inventory,
  AI, in-game UI, level logic, and the C++↔Lua game bindings. → `engine/xrGame.md`
- **Layers/** (~613) — the renderer. Common code in `xrRender/`; per-path code in
  `xrRenderPC_R1..R4/`; D3D backends in `xrRenderDX9/`, `xrRenderDX10/`; abstraction in `xrAPI/`.
  → `engine/renderer.md`
- **xrCore/** (~178) — foundation lib: strings (`shared_str`), `smart_cast`, file system (`FS`),
  LTX/ini + DLTX parsing, math, debug. Everything depends on it. → `engine/xrCore.md`
- **xrServerEntities/** (~161) — ALife server objects + the Lua script engine host (`CScriptEngine`,
  exports). Shared between game and server. → `engine/server-entities.md`
- **xrPhysics/** (~150) — ODE-based physics. → `engine/physics-collision.md`
- **xrEngine/** (~187) — the executable core: device, main loop, scheduler, console, input, the Lua
  debug bridge. → `engine/engine-core.md`
- **xrCDB/** (~86) — collision database (CFORM build/query). → `engine/physics-collision.md`
- **xrSound/** (~34) — audio + doppler. → `engine/sound.md`
- **xrNetServer/** (~19) — networking transport (ALife/MP). → `engine/server-entities.md`
- **xrParticles/**, **xrCPU_Pipe/**, **xrXMLParser/**, **ReShadeCompat/** — small subsystems.
  → `engine/subsystems-misc.md`
- **Include/**, **3rd party/** — shared headers and vendored libs (luabind, luajit, ODE, etc.).
- **compressor/**, **sdk/** (repo root) — offline tools, not runtime.

## Module wiring

Verified: `xrEngine`'s `CEngineAPI` (`EngineAPI.cpp`) is the host that wires in the render path and
game module. In this **all-in-one build** the renderer is fixed at **compile time** — `#error` guards
enforce exactly one of R1/R2/R3/R4 (`EngineAPI.cpp`) — and the historical `LoadLibrary(rN_name)`
/ `LoadLibrary(g_name)` DLL loads are **commented out**, i.e. modules are statically linked
into one exe (hence "all-in-one"). Render path detail: `engine/renderer.md`.

Dependency edges verified from `.vcxproj` ProjectReferences: **`xrGame`** links
`xrCore`, `xrCDB`, `xrPhysics`, `xrNetServer`, `xrXMLParser`, `Layers/xrAPI` (renderer seam), plus
3rd-party `luabind`, `ode`, `crypto` (`xrGame.vcxproj`). Notably `xrServerEntities` is *not*
a separate reference — the script-engine/ALife sources compile directly into `xrGame` under
`XRGAME_EXPORTS` (`engine/server-entities.md`). So: `xrCore` is the base foundation everything links;
`xrGame` sits on top and reaches the renderer only through `xrAPI`/`Layers`. *(INFERRED — the full
edge set of the *other* modules, e.g. exactly what `xrCDB`/`xrPhysics`/`xrEngine` each link, was not
enumerated from their own vcxproj files.)*

## Build system

- **Primary**: Visual Studio 2022 solution `src/engine-vs2022.sln` (older `engine.sln` also present).
  `src/Common.props`, `src/build_config_defines.h` hold shared build config/feature defines.
- **CMake**: `CMakeLists.txt` at repo root + `cmake-build-debug/` (CLion/CMake workflow). Note: the
  local environment has no cmake/MSVC on PATH — builds happen in the IDE.
- **Hot reload**: *(per README "Working with Hot Reload" — not verified in code)* VS2022 flow for
  fast engine iteration. See `GLOSSARY.md#hot-reload`.

## Branch / variant scheme

Branch names encode the build variant (verified via `git branch`): `-wpo`, `-mt`, `-vulkan`; current
checkout/main is `all-in-one-vs2022-wpo`. *(Meaning of each variant and the MT threading feature list
— loading, grass/rain/particles, bone calc, AI feel/vision, scheduler, UI, logger — are from the
README, not verified in code.)* See `GLOSSARY.md`.

## Where to start for common changes

- Gameplay/item/AI behavior → `engine/xrGame.md`
- Visual/shader → `engine/renderer.md`
- A new value read from config → `engine/xrCore.md` (parse) + the consuming module
- A Lua-exposed function → `engine/server-entities.md` (exports)

## Deeper

- Solution/build config: `src/engine-vs2022.sln`, `src/Common.props`, `src/build_config_defines.h`,
  `src/batch_build.bat`, `CMakeLists.txt`.
- Feature/patch catalog: `README.md` "List of patches" and "Changelog".

## Maintenance

- Re-list `src/` and recount per-module file sizes (`find src/<mod> -name '*.cpp' -o -name '*.h'`) if
  modules were added/removed; update the module map.
- Confirm the solution filename and main branch name (`git branch`) are still current.
- If the MT/wpo/vulkan variant set changes, sync with `GLOSSARY.md`.
