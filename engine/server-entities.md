---
domain: engine/server
tags: [engine, cpp, alife, server, net, luabind]
sources: [src/xrServerEntities/script_engine.h, src/xrServerEntities/script_engine_export.cpp, src/xrServerEntities/xrServer_Objects_ALife.h, src/xrServerEntities/xrServer_Object_Base.h, src/xrNetServer/]
related: [engine/_overview.md, engine/xrGame.md, flows/lua-callback-lifecycle.md, flows/alife-online-offline.md, flows/save-load.md, modlayer/scripting.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Server entities, ALife & script engine

`xrServerEntities` holds two things: the ALife **server objects** and the **Lua script engine host**.
`xrNetServer` is the network transport. Structure and the script-engine/exports mechanism are
verified; the offline-brain update mechanism is verified, only the online-radius threshold + online
counterpart spawn/destroy remain inferred.

## Script engine host (verified)

- **`CScriptEngine`** — `class CScriptEngine : public CScriptStorage` (`script_engine.h`). Owns Lua
  error handling (`lua_panic`, `lua_error`, `lua_pcall_failed`, `lua_hook_call`), GC
  (`collect_all_garbage`), and script loading (`load_common_scripts`; `init` and
  `do_file` in the `.cpp`/`CScriptStorage`). It extends `CScriptStorage`, which implements `do_file`
  (the loader + unlocalizer — see `modlayer/unlocalizer.md`).
- **Exports** — `export_classes(lua_State* L)` (`script_engine_export.cpp`) registers C++ classes into
  Lua by calling `<Class>::script_register(L)` for each (verified list incl. `CScriptEngine`,
  `CScriptNetPacket`, `CScriptFvector/Fcolor/Fmatrix/Flags`, `CScriptIniFile`, `CScriptReader`,
  `CScriptActionBase/Planner`, gamespy classes…). The actual luabind `class_<>` bindings live inside
  each class's `script_register` (which is why they aren't in this aggregator file). This is the
  answer to "how do I expose X to Lua?" — add a `script_register` and call it here.
- Original author Dmitriy Iassenev (file header); this fork uses an updated luabind
  (`GLOSSARY.md#luabind`).

## ALife server objects (structure + offline-brain mechanism verified)

- Server-object classes live here: `xrServer_Object_Base.h`, `xrServer_Objects_ALife.h`,
  `xrServer_Objects_Alife_Smartcovers.h`, `alife_space.h` (CSE_* classes — file existence verified).
  `cast_online_offline_group()` on the base (`xrServer_Object_Base.h`) confirms the
  online/offline group split.
- Offline AI brains present: `alife_human_brain.h`, `alife_monster_brain.h`,
  `alife_online_offline_group_brain.h`. **Offline simulation verified** in
  `CALifeMonsterBrain::update` (`alife_monster_brain.cpp`): `select_task` → `process_task` when
  the object is assigned a smart_terrain (`m_smart_terrain_id != 0xffff`) else `default_behaviour`,
  then `movement().update()` — i.e. offline objects run task/job selection + movement on the server.
  `on_switch_online`/`on_switch_offline` delegate to `movement()` on transition.
- **Online↔offline switch (verified — see `flows/alife-online-offline.md`):** driven by actor
  proximity with a hysteresis band. `switch_distance`/`switch_factor` are read from the `"alife"`
  system.ltx section; `online_distance = switch_distance*(1 - factor)`, `offline_distance =
  switch_distance*(1 + factor)`. `CALifeUpdateManager::update_switch` walks the level registry each
  tick; `add_online` destroys the server copy then `server().Process_spawn`s the online `CGameObject`,
  `remove_online` `Perform_destroy`s it (with a `VERIFY(children.empty())` guard) and re-registers the
  entity offline. (This resolves the prior inferred online-radius/counterpart-spawn note.)
- Lua-side scripted squads: `gamedata/scripts/sim_squad_scripted.script` (file exists).

## Networking (verified by listing)

`xrNetServer` — `NET_Client.*`, `NET_Common.*`, `NET_Compressor.*`, `NET_AuthCheck.*`, `NET_Log.*`:
packet transport + compression + auth used by ALife transitions and (historically) multiplayer.

## Deeper

- Script engine: `src/xrServerEntities/script_engine.cpp/.h` (`init`, `load_common_scripts`,
  `collect_all_garbage`), `script_storage.cpp` (`do_file`)
- Exports: `src/xrServerEntities/script_engine_export.cpp` (`export_classes`); per-class bindings via
  each class's `script_register`
- ALife objects: `src/xrServerEntities/xrServer_Objects_ALife.h`, `xrServer_Object_Base.h`
- Net: `src/xrNetServer/`
- Lua squads: `gamedata/scripts/sim_squad_scripted.script`

## Maintenance

- The offline-brain update is verified (`alife_monster_brain.cpp`); the online↔offline switch logic
  (threshold + counterpart spawn/destroy) is now verified in `flows/alife-online-offline.md` (git
  history mentions transition crash fixes — that flow documents the crash surface).
- Keep the exports pointer accurate; it's the canonical "expose C++ to Lua" recipe.
