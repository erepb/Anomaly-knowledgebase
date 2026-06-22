---
domain: flows/save-load
tags: [xrgame, server, alife, save, load, serialization, lua, flow]
sources: [src/xrGame/alife_storage_manager.cpp, src/xrGame/alife_object_registry.cpp, src/xrGame/script_binder.cpp, src/xrGame/GameObject.cpp, src/xrGame/game_sv_single.cpp]
related: [flows/object-lifecycle.md, flows/alife-online-offline.md, flows/lua-callback-lifecycle.md, engine/server-entities.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: save / load lifecycle

How a single-player game is serialized to a `.sav` file and restored. The world state lives in the
**ALife server-entity registry** (offline objects, abstract `CSE_ALifeDynamicObject`s); online level
objects (`CGameObject`s) push their state back into their server entity first, so the single ALife
save covers everything. Builds on the object backbone (`flows/object-lifecycle.md`) and the ALife
online/offline split (`flows/alife-online-offline.md`).

**Provenance:** spine personally code-verified — `game_sv_Single::save_game`/`load_game`,
`CALifeStorageManager::save`/`load` (both overloads), `CALifeObjectRegistry::save`/`load`/`get_object`,
`CScriptBinder::save`/`load`, `CGameObject::net_Save`/`net_Load`. Header/time/spawn sub-registry
serializers (`header().save()`, `time_manager().save()`, `spawns().save()`) are confirmed call sites
but their bodies are *(agent-traced — not personally re-read)*.

## Save sequence (verified)

1. **`game_sv_Single::save_game`** (`game_sv_single.cpp`) — entry; if no ALife, return. Else call
   `alife().save(net_packet)`.
2. **`CALifeStorageManager::save(NET_Packet&)`** — `prepare_objects_for_save()` then reads the game
   name (and an `update_name` flag) off the packet and calls the `save(LPCSTR, bool)` overload.
3. **`prepare_objects_for_save`** — `Level().ClientSend()` then `Level().ClientSave()`: flushes the
   live client objects so each online `CGameObject`'s current state is written back into its server
   entity *before* the registry is serialized (this is the online→server sync seam).
4. **`CALifeStorageManager::save(LPCSTR, bool)`** — builds the byte stream into a `CMemoryWriter`:
   `header().save()` → `time_manager().save()` → `spawns().save()` → **`objects().save()`** →
   `registry().save()`. Then `rtc_compress` the buffer, open `$game_saves$/<name>.sav`, and write a
   3-word header `w_u32(u32(-1))` (magic) + `w_u32(ALIFE_VERSION)` + `w_u32(source_count)` followed by
   the compressed payload. (Optional Lua hooks fire around this — see ledger.)
5. **`CALifeObjectRegistry::save(IWriter&)`** — iterates all server entities; skips any failing
   `can_save()`, `redundant()`, or with `ID_Parent != 0xffff` (children are saved by their parent).
   For each top-level object calls the per-object recursive `save`.
6. **`CALifeObjectRegistry::save(IWriter&, object, count&)`** — per object: `Spawn_Write` packet then
   an `M_UPDATE` `UPDATE_Write` packet (each length-prefixed `w_u16`), then recurses into `children`
   (each filtered by `can_save()`). **Note:** the registry serializes via `Spawn_Write`/`UPDATE_Write`,
   *not* `STATE_Write`.
7. **online-object state** — for objects that are online, their per-frame state reached the server
   entity in step 3 via **`CGameObject::net_Save`** → `save(net_packet)` (engine state) +
   **`CScriptBinder::save`** (the Lua `save_state`), wrapped in a `w_chunk_open16`/`close16` pair.

## Load sequence (verified)

1. **`game_sv_Single::load_game`** — reads the game name; calls `alife().load_game(name, true)` (the
   update-manager wrapper) → `CALifeStorageManager::load(LPCSTR)`.
2. **`CALifeStorageManager::load(LPCSTR)`** — `FS.r_open` the `.sav`; if missing, log and restore the
   previous `m_save_name`, return false. `CHECK_OR_EXIT(CSavedGameWrapper::valid_saved_game(...))`
   (hard exit on version mismatch / corruption). Then `LoadTitle`, **`unload()`** (tear down current
   world) + **`reload(section)`** (re-init ALife structures), read `source_count`, `rtc_decompress`
   into a buffer, call the `load(buffer, size, file)` overload, then `groups().on_after_game_load()`
   and `VERIFY(graph().actor())`.
3. **`CALifeStorageManager::load(void*, size, file)`** — deserialize in the same order:
   `header().load()` → `time_manager().load()` → `spawns().load()` → `graph().on_load()` →
   **`objects().load()`**. Then the **ID-regeneration pass**: with `can_register_objects(false)`, walk
   every loaded object, `ID = server().PerformIDgen(id)` (asserting the id is unchanged), and
   `register_object(obj, false)`. Then `registry().load()`, flip `can_register_objects(true)`, and run
   a second pass calling **`on_register()`** on each object (lets smart-terrains/registries re-sync).
   Finally `Level().autosave_manager().on_game_loaded()` if a level is present.
4. **`CALifeObjectRegistry::load`** — finds `OBJECT_CHUNK_DATA`, clears `m_objects`, reads the count,
   and for each calls `get_object` then `add`.
5. **`get_object`** — reads the spawn packet (asserts `M_SPAWN`), reads the section name, creates the
   server entity via `F_entity_Create(name)` (null/non-ALife handled: skip or assert), `Spawn_Read`,
   then reads the `M_UPDATE` packet and `UPDATE_Read`.
6. **online re-creation** — online objects are re-instantiated as level `CGameObject`s through the
   normal spawn path (`flows/object-lifecycle.md`); **`CGameObject::net_Load`** restores engine state
   then **`CScriptBinder::load`** runs the Lua `load_state`.

## Isolation ledger (the seams)

- **Lua per-object state (binder seam):** `CScriptBinder::save`/`load` call the bound Lua object's
  `save`/`load` inside a `try/catch` — **a throwing Lua save/load is swallowed and the binder is
  `clear()`ed**, not propagated. This is the modlayer's `xr_logic` pstor / `save_var` persistence path
  (`flows/lua-callback-lifecycle.md`).
- **Optional global Lua hooks:** only compiled when `ENGINE_LUA_ALIFE_STORAGE_MANAGER_CALLBACKS` is
  defined — `alife_storage_manager.CALifeStorageManager_before_save` / `_save` (around the file write)
  and `_load` (at the start of the buffer load). Located via `ai().script_engine().functor(...)`.
- **online→server sync:** `Level().ClientSend()`/`ClientSave()` in `prepare_objects_for_save` is the
  one point where live `CGameObject` state is flushed into server entities; without it online objects
  would save stale.
- **Server / ID generator:** `server().PerformIDgen` regenerates every id on load (collision-safe
  re-registration); `F_entity_Create` is the server-entity factory keyed by section name.
- **Config / FS:** `$game_saves$` FS path; save format header carries `ALIFE_VERSION`; the `"alife"`
  system.ltx section parameterizes the simulator (shared with `flows/alife-online-offline.md`).
- **No callbacks/scheduler during the write:** ALife time does not advance during serialize/restore
  (no explicit lock — relies on running on the main thread, see Threading).

## Threading

Save and load run synchronously on the main game thread and block until the file is written /
decompressed and all objects registered. There is no engine-level locking; the implicit guarantee is
that the scheduler is not advancing ALife mid-serialize.

## Failure & edge cases

- **Version/corruption:** `CSavedGameWrapper::valid_saved_game` failing triggers `CHECK_OR_EXIT`
  (fatal, with the file name) — not a soft error.
- **Missing file on load:** soft — logs "Cannot find saved game", restores prior name, returns false.
- **Lua binder exception:** caught per-object; the binder is `clear()`ed and save/load continues — a
  broken script `save_state` silently drops that object's script state rather than crashing.
- **Un-creatable section on load:** `F_entity_Create` returning null logs and skips that object
  (advances past its update packet); a non-ALife class asserts.
- **`STATE_Write` vs `Spawn/UPDATE`:** the offline registry uses Spawn+Update packets; don't assume a
  single `STATE_Write` blob (a common misreading).

## Deeper

- Orchestrator: `src/xrGame/alife_storage_manager.cpp` (`save`/`load` overloads,
  `prepare_objects_for_save`)
- Registry: `src/xrGame/alife_object_registry.cpp` (`save`, `get_object`, `load`)
- Entry: `src/xrGame/game_sv_single.cpp` (`save_game`, `load_game`)
- Per-object online state: `src/xrGame/GameObject.cpp` (`net_Save`, `net_Load`),
  `src/xrGame/script_binder.cpp` (`save`, `load`, `net_SaveRelevant`)
- Server entity serializers: `Spawn_Write`/`Spawn_Read`/`UPDATE_Write`/`UPDATE_Read` in
  `src/xrServerEntities/` (`engine/server-entities.md`)

## Maintenance

- Re-grep `CALifeStorageManager::save`/`load` if the on-disk header or sub-registry order changes
  (header → time → spawns → objects → registry must stay in sync between save and load).
- If a save-format bump happens, confirm `ALIFE_VERSION` and `CSavedGameWrapper::valid_saved_game`.
- Promote the *(agent-traced)* sub-registry bodies (`header`/`time_manager`/`spawns` save/load) to
  verified if a task needs exact field layout.
- Keep the binder try/catch-swallows-and-clears behavior documented — it is a real silent-failure mode
  for script persistence.
