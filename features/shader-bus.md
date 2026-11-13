---
domain: features/shader-bus
tags: [engine, renderer, shaders, lua, dx9, dx10, dx11, console, imgui, cpp]
sources: [docs/shader_bus.md, src/xrEngine/shader_bus.cpp, src/xrEngine/shader_bus.h, src/xrGame/shader_bus_script.cpp, src/Layers/xrRender/Blender_Recorder_StandartBinding.cpp, src/Layers/xrRender/r_constants.cpp, src/Layers/xrRenderDX10/dx10r_constants.cpp, gamedata/scripts/imgui_shader_bus.script]
related: [flows/renderer-resources-shaders.md, flows/renderer-blender-compile.md, flows/ui-debug-tools.md, features/script-attachments.md, GLOSSARY.md#shader-bus]
updated: 2026-11-13
---

# Shader bus

Named float4 lanes between Lua and shaders, replacing the anonymous `shader_param_1..8` /
`s3ds_param_1..4` console constants (which still work, but log a one-time nudge naming the writer).
By TheLostInPlace (PRs #665, #736). **Authoritative guide: `docs/shader_bus.md`** (API table, lane
kinds, limits, migration recipe, rules). `shader_bus.version()` returns the bus version; shaders can
test the `SHADER_BUS` define (added by every renderer's shader-define list).

## Model

- **Shader side** — any constant named `bus_<id>` is a lane. At constant-table reflection the binder
  setup in `Blender_Recorder_StandartBinding.cpp` (`bus_binder`, one per lane index) attaches it.
  Binds `float`..`float4`; on DX10/11 also arrays, matrices (rows of 16 bytes, ≤4096 rows) and `uint`
  kinds. A type the bus can't bind (int/bool, arrays on DX9, oversize) binds nothing and logs once.
  Constant loads now keep their declared type and size (`r_constants.cpp`, `dx10r_constants.cpp`).
- **Script side** — `shader_bus.register(id, owner, desc)` → token; `set`, `set_array`, `set_uint`,
  `set_array_uint`, `set_texture` (lane texture sampled via `$user$bus_<id>`); `try_register`,
  `register_shared` (many writers, largest value wins); `list`, `get`, `get_pending`, `owner_of`.
  Exported in `shader_bus_script.cpp`.
- **Commit point** — pending → bound once per frame (before frame callbacks), so `get` trails `set`
  by a frame; runs in menus and loading too.
- **Ownership** — one owner per non-shared lane for the process lifetime; a different owner on
  `register` is **fatal by design**. Lanes are never removed.

## Special lanes

- `engine_*` — engine-filled (`engine_sun_los`, `engine_sky_open`), only computed once a shader
  declares them.
- `cvar_<name>` — mirrors a console variable, no script needed.
- `obj_*` — per-object values (`set_object` / `clear_object`, `set_attachment` for script
  attachments); HUD draws read the actor's value; ≤16 object lanes per pass. `obj_hotness` is shared
  from the start and overrides the stock `L_hotness`.

## Debugging

Console `bus_list`, `bus_get <id>`, `bus_force <id> x y z w`, `bus_release <id>`. ImGui monitor
(`imgui_shader_bus.script`, Debug > Shader bus). At level load, lanes declared by a shader but
registered by nobody are warned — a missing-dependency signal. Full lane dump only with `-dbg`.

## Deeper

- Engine registry: `src/xrEngine/shader_bus.h` (`ShaderBus::lane`), `shader_bus.cpp`
- Binder: `Blender_Recorder_StandartBinding.cpp` (`bus_binder`)
- Lua module: `src/xrGame/shader_bus_script.cpp`
- Monitor: `gamedata/scripts/imgui_shader_bus.script`

## Maintenance

- Check `shader_bus.version()` in `docs/shader_bus.md` intro vs code when the bus changes.
- Don't restate the API table here; update `docs/shader_bus.md` instead.
