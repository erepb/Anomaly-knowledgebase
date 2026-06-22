---
domain: modlayer/scripting
tags: [modlayer, lua, callbacks, luabind, hot-reload]
sources: [gamedata/scripts/axr_main.script, gamedata/scripts/callbacks_gameobject.script, gamedata/scripts/class_registrator_modded_exes.script, src/xrServerEntities/script_storage.cpp]
related: [modlayer/_overview.md, modlayer/options-ui.md, engine/server-entities.md, engine/xrGame.md, flows/lua-callback-lifecycle.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Lua scripting model

How the mod layer's Lua works: file model, globals, the callback system, and module lifecycle.

##.script files & namespaces

Each `gamedata/scripts/<name>.script` is a Lua chunk loaded by the engine's `CScriptEngine`
(`engine/server-entities.md`). Its top-level declarations form the `<name>` namespace, reached from
other scripts as `name.symbol`. `local` symbols are private unless promoted by the **unlocalizer**
(`modlayer/unlocalizer.md`). The **common** scripts set is config-driven: `load_common_scripts`
(`script_engine.cpp`) reads `script.ltx` `[common] script=` and `process_file`s each, calling its
`<name>_initialize` if present; other namespaces are lazily `process_file`d on first reference.
`.lua` kernel scripts (`global.lua`, `dynamic_callbacks.lua`, `socket.lua`, `LuaPanda.lua`) are loaded
via `LoadKernelScriptToGlobal` **only when `lua_debug` is enabled** (verified:
`script_storage.cpp` — all four are inside the `isDebugEnabled` block; socket + LuaPanda are
the debugger transport). *(Note: corrects an earlier KB claim that `dynamic_callbacks.lua` always
loads — it is debug-only here; cross-check `flows/lua-callback-lifecycle.md`.)*

File-name prefixes: `modxml_*` = DXML patchers (verified: gathered by `dxml_core.script`),
`options_modded_exes_*` = option modules (verified: `import_into`+`GROUP`). *(INFERRED — convention,
not engine code: `aaa*`/`zz*` ordering and `_g*` = global helpers are Lua-side load-order conventions,
e.g. axr_main's `on_game_start` enumeration in base `unpacked/` scripts, not enforced by the engine.)*

## Callback system

> Full boot→dispatch sequence with function digests: **`flows/lua-callback-lifecycle.md`**. Summary
> below.

Anomaly's event bus. Modules subscribe with `RegisterScriptCallback("on_event", fn[, obj])`; events
fire via `SendScriptCallback("on_event",...)`. The manager is **axr_main.script** (by Alundaio):
it auto-runs any script exposing `on_game_start` and wires modules in (verified, see flow).
**callbacks_gameobject.script** bridges engine object hooks to the bus — verified: it defines
`_G.CGameObject_NetSpawn/NetDestroy` (engine-invoked) that call `SendScriptCallback(...)`, and uses
`CObjectBinder:net_spawn` → `self.object:set_callback(callback.net_spawn_after, …)`. Many callbacks
originate in the engine (input, hit, XML read, …) — see `GLOSSARY.md#callbacks`.

Representative callbacks seen in this repo's scripts: `on_xml_read`, `on_key_press`/`on_key_release`,
`on_before_hit_after_calcs`, `on_death`, `on_map_spot_selected`, `on_specific_character_init`,
`on_luabind_fatal_error`, `on_loading_screen_*`, `on_belt_size_get`, `on_artefact_count_get`.

## Module lifecycle / hot reload

`on_game_start` (auto-invoked by axr_main) is the standard module entry point where you register
callbacks (verified, see flow). GC is managed by `CScriptEngine` (`collect_all_garbage` =
double `lua_gc(LUA_GCCOLLECT)`, verified `script_engine.cpp`). **Hot-reload is a real engine
path** (verified): `CScriptEngine::process_file(name, reload_modules=true)` (`script_engine.cpp`)
sets `m_reload_modules`, which makes `process_file_if_exists` re-run `load_file_into_namespace` even
when the namespace is already loaded — re-executing the module chunk. The engine itself uses
this at init to force-reload `_G` (`script_engine.cpp`). *(INFERRED — a runtime/console
trigger that reloads an arbitrary gameplay module mid-session, and the README's "MT idle-time parallel
GC", were not traced.)* See `GLOSSARY.md#hot-reload`.

## Class registration & exports

`class_registrator_modded_exes.script` `register(object_factory)` maps engine classes to scripted
classes via `cs_register(factory, client_class, server_class, clsid, script_clsid)` — verified:
`CWeaponStatMgun`↔`se_stmgun.se_stmgun` (`_S_STMGU`) and `CProjector`↔`se_projector.se_projector`
(`_S_SEARC`). New C++→Lua callables are bound engine-side (`engine/server-entities.md` exports), not
here.

## Key infra files

- `axr_main.script` — callback/module manager, reads `axr_options.ltx` via `ini_file_ex`.
- `callbacks_gameobject.script` — game-object callback dispatch.
- `dynamic_callbacks.lua` — dynamic callback helpers.
- `_g_patches.script` — global function patches/fixes.
- `lua_help_ex.script` — extra Lua helpers added by modded exes.
- `modded_exes_migration.script` — version migration of mod settings.
- `imgui_helper.script`, `imgui_lua_debug.script`, `lua_help_imgui.script` — ImGui binding helpers.

## Deeper

- Manager + callbacks: `gamedata/scripts/axr_main.script`, `gamedata/scripts/callbacks_gameobject.script`
- Class registration: `gamedata/scripts/class_registrator_modded_exes.script`
- Engine side (state, exports, GC): `src/xrServerEntities/script_engine*.cpp`
- Full module list: list `gamedata/scripts/`

## Maintenance

- Refresh the representative-callback list by grepping `RegisterScriptCallback(`/`on_[a-z_]+` across
  `gamedata/scripts/` if new events appear.
- If a new infra/manager script is added, list it under "Key infra files".
- Keep the engine-exports boundary pointer accurate (where new Lua callables are bound).
