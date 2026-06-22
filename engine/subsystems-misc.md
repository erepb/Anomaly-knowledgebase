---
domain: engine/misc
tags: [engine, cpp]
sources: [src/xrParticles/, src/xrCPU_Pipe/PLC.cpp, src/xrXMLParser/xrXMLParser.cpp, src/ReShadeCompat/addon.cpp, src/3rd party/, compressor/readme.txt]
related: [engine/_overview.md, engine/renderer.md, flows/dxml-xml-interception.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Misc subsystems & vendored libs

Smaller engine modules. Purposes verified by reading each module's code, except `xrParticles`
emitter/sim details (large; marked inferred).

## Modules

- **xrParticles/** — particle system. Verified files: `particle_actions.h`, and
  `ParticlesAsyncManager.h` (confirms the MT async-particle work). *(INFERRED — unverified:
  emitter/simulation details; not read.)*
- **xrCPU_Pipe/** — **CPU point-light vertex-lighting calc** (verified by reading `PLC.cpp`): exposes
  `PLC_energy_*`/`PLC_calc3_*` which compute per-vertex light energy/color for a triangle from a
  `light*`, with both x86 (`_x86`) and SSE (`_SSE`) implementations. (Not a thread/job pool — earlier
  guess corrected.)
- **xrXMLParser/** — the engine XML reader. Verified: `xrXMLParser.cpp` calls
  `XMLLuaCallback(*this, xml_string)` — the DXML interception point (full flow:
  `flows/dxml-xml-interception.md`). Consumers: `src/xrGame/ui/UIXmlInit.cpp`,
  `src/xrGame/ScriptXMLInit.cpp`.
- **ReShadeCompat/** — **ReShade addon** (verified `addon.cpp`): `init_reshade` calls
  `reshade::register_addon` and subscribes to `reshade_begin_effects`; `render_reshade_effects` calls
  `runtime->render_effects(...)` into the game's RTV — i.e. it drives ReShade's effect pass from the
  engine (idea credited to "SSE ReShade Helper").

## Tools & vendored

- **compressor/** (repo root) — offline **mod-packaging tool** (verified `compressor/readme.txt`):
  `xrPackage.bat` runs `xrCompress mod -ltx xrCompress.ltx -pack -db` to pack a `mod/` dir (mirroring
  `gamedata/` layout) into a `mod.db0` archive for `db\mods` (numeric-prefixed `update-NNN-*.db0` go in
  `db\updates`). Confirms the `.db0` archives consumed by `LoadArchive` (`modlayer/_overview.md`).
- **src/3rd party/** — vendored libraries: `luabind-*`, `luajit-2`, `ode*`, etc. (directories exist;
  generated luajit headers are gitignored per `.gitignore`). Treat as upstream.
- **src/Include/** — shared public headers across modules.

## Deeper

- Particles: `src/xrParticles/` (`ParticlesAsyncManager.h`, `particle_actions.h`)
- XML: `src/xrXMLParser/xrXMLParser.cpp` (`XMLLuaCallback` call site); consumers in `src/xrGame/ui/`
- Vendored: `src/3rd party/` (luabind, luajit-2, ode)

## Maintenance

- The DXML interception point (`xrXMLParser.cpp` → `XMLLuaCallback`) is the load-bearing fact here;
  keep it synced with `flows/dxml-xml-interception.md`.
- Promote any *(INFERRED)* purpose to verified only after reading the module; otherwise leave marked.
