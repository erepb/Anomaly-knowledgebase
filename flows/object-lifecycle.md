---
domain: flows/object-lifecycle
tags: [engine, cpp, lifecycle, flow, scheduler, alife, callbacks, luabind]
sources: [src/xrGame/GameObject.cpp, src/xrGame/GameObject.h, src/xrEngine/xr_object.cpp, src/xrEngine/xr_object.h, src/xrGame/Level_network_spawn.cpp, src/xrGame/game_object_space.h]
related: [engine/xrGame.md, engine/engine-core.md, engine/server-entities.md, flows/lua-callback-lifecycle.md, flows/weapon-fire-callbacks.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: game-object lifecycle (spawn → update → destroy)

The backbone every `xrGame` object hangs off: how a `CGameObject` is spawned from a server entity,
updated each frame (real-time vs scheduled), and torn down — and where the scheduler, ALife, and Lua
seams attach. This is the frame other domain flows (weapon-fire, AI, inventory) plug into.

**Provenance:** researched with a read-only agent, then spot-verified against code. The spine
(class hierarchy, `net_Spawn`/`net_Destroy`/`shedule_Update`/`UpdateCL` bodies, `CObject::net_Spawn`
registration, `g_sv_Spawn`, the two script callbacks, `eNetSpawnAfter`) is personally code-verified.
A few sub-steps are marked *(agent-traced — not personally re-read)*.

## Inheritance (verified)

`class CGameObject : public CObject, public CUsableScriptObject, public CScriptBinder`
(`GameObject.h`; `inherited == CObject`). So every game object is simultaneously an engine object
(`CObject`: scheduled + renderable + collidable), a Lua-usable object, and a **script binder** (the
mixin that drives the Lua-side object lifecycle). `CObject` itself implements `ISheduled` (engine
scheduler interface — see `engine/engine-core.md`).

## Spawn (network-driven)

1. **`CLevel::g_sv_Spawn`** (`Level_network_spawn.cpp`) — the spawn pipeline. The object is already
   created from its section name (`Objects.Create`), then `O->net_Spawn(E)` runs. On failure →
   `O->net_Destroy()` + `Objects.Destroy(O)`. On success → `client_spawn_manager().callback(O)`; if
   `ID_Parent != 0xffff` it raises a `GE_OWNERSHIP_TAKE` event; if the spawn flags are
   `M_SPAWN_OBJECT_LOCAL` + `M_SPAWN_OBJECT_ASPLAYER` it makes the object the control/current entity
   (`SetControlEntity`/`SetEntity`); finally fires the **`eNetSpawnAfter`** per-object callback (under
   the `NET_SPAWN_AFTER_CALLBACKS` build define). *(The upstream network entry `CLevel::cl_Process_Spawn`
   — read packet → `F_entity_Create` → `Spawn_Read` → `g_sv_Spawn` — is agent-traced.)*
2. **`CGameObject::net_Spawn`** (`GameObject.cpp`) — game-side spawn, in order:
   `VERIFY(!m_spawned)` → `m_spawned = true`, record `m_spawn_time`, create `m_ai_obstacle`; resolve
   the visual from `CSE_Visual` (with a `pSettings [section] visual` fallback if the model is missing);
   set names from `E->s_name`/`name_replace`; **reject a duplicate ID** (`Level().Objects.net_Find(E->ID)`
   → log + `return false`); `setID`; set `XFORM` from the server entity's `o_Angle`/`o_Position`; build
   `m_ini_file` from `CSE_ALifeObject::m_ini_string` (this is the object's `spawn_ini`); set AI level/game
   vertices from `m_tNodeID`/`m_tGraphID`. Then `inherited::net_Spawn(DC)` (base `CObject`),
   `m_bObjectRemoved = false`, **`spawn_supplies()`**, fire the Lua callback **`_G.CGameObject_NetSpawn`**
   (release path; passes `lua_game_object()`), and **`CScriptBinder::net_Spawn(DC)`**. *(The CScriptBinder
   `reload`/`reinit` calls earlier in the body are agent-traced.)*
3. **`CObject::net_Spawn`** (`xr_object.cpp`) — engine registration: `spatial_register()` (spatial DB),
   `shedule_register()` if `register_schedule()`, `processing_activate()`, `MakeMeCrow()`.
4. **`CGameObject::spawn_supplies`** — spawns child items declared in the `[spawn]` section of the
   object's `spawn_ini` (probability/condition/addon flags), via `Level().spawn_item`.
   *(agent-traced — not personally re-read.)*

## Per-frame update — two channels (verified)

An object is updated on **two** independent channels:

- **Real-time, every frame** — `CGameObject::UpdateCL`: `inherited::UpdateCL()`, then for each
  `m_script_attachments` entry call `->Update()`; if it has a parent (`H_Parent()`) return early;
  otherwise `on_matrix_change(m_previous_matrix)` (→ `obstacle().on_move()`) and store the new `XFORM`
  into `m_previous_matrix`. Driven by `CObjectList::SingleUpdate` *(engine `xr_object_list.cpp` —
  agent-traced)*.
- **Deferred, on the scheduler's budget** — `CGameObject::shedule_Update`: if `NeedToDestroyObject()`
  → `DestroyObject()`; then `inherited::shedule_Update(dt)` (base updates spatial position); then
  `CScriptBinder::shedule_Update(dt)` unless `g_dedicated_server`. Driven by `CSheduler::UpdateDeferred`
  (time-budgeted, serial — see `engine/engine-core.md`). `NeedToDestroyObject` is `false` in the base
  `CGameObject` — **derived classes** (Actor, weapons, items, grenade…) override it to decide self-destruction.

The **Crow** mechanism couples the two: `CObject::MakeMeCrow` re-flags an object so `UpdateCL` runs
next frame (called after spawn, after `shedule_Update`, and on spatial move), ensuring a scheduled
object still gets a real-time tick when it moved.

`CGameObject::shedule_Needed` gates whether the scheduler should call the object *(returns
`!getDestroy()` — agent-traced)*.

## Destroy (verified)

1. **Request** — `CGameObject::DestroyObject`: guarded by `m_bObjectRemoved` (set once); if already
   `getDestroy()` return; if `Local()` send a `GE_DESTROY` event (`u_EventGen`/`u_EventSend`). So
   destruction is requested via the network/event layer, not done inline.
2. **Teardown** — `CGameObject::net_Destroy` (invoked by the engine after `setDestroy`): delete the
   anim-mov controller, `inherited::net_Destroy()`, delete `m_ini_file`, fire the Lua callback
   **`_G.CGameObject_NetDestroy`**, **`g_pGameLevel->Objects.net_Unregister(this)`**, clear the control
   entity if this was it, and `CScriptBinder::net_Destroy()`. Base `CObject::net_Destroy` does
   `shedule_unregister()` + `spatial_unregister()` + collision-model delete *(base internals
   agent-traced)*.

## Isolation ledger (the seams)

- **Callbacks fired (per-object `callback(GameObject::e*)`):** `eNetSpawnAfter` (in `g_sv_Spawn`),
  `eOnFootStep` (`FootStepCallback`, with `lua_game_object`, power, play, on_ground, hud_view). Plus the
  script functors `_G.CGameObject_NetSpawn` / `_G.CGameObject_NetDestroy` (bridged to the global bus by
  `callbacks_gameobject.script` — see `flows/lua-callback-lifecycle.md`). Derived classes raise more
  (`eHit`, `eDeath`, `eExitLevelBorder`/`eEnterLevelBorder` from Actor — see `engine/xrGame.md`). Enum
  ↔ Lua-name mapping in `script_game_object_script.cpp` (e.g. `net_spawn_after` → `eNetSpawnAfter`).
- **Config read at spawn:** `pSettings [section] visual` (fallback model); the object's `spawn_ini`
  (`CSE_ALifeObject::m_ini_string`) `[spawn]` section (child items).
- **Scheduler seam:** `CObject::shedule_register` at spawn; **real-time** path = `UpdateCL` every frame,
  **deferred** path = `shedule_Update` on the scheduler budget; `MakeMeCrow` bridges them. Scheduler is
  serial (`engine/engine-core.md`).
- **ALife / server seam:** spawn is driven entirely by the server entity (`CSE_Abstract` → `CSE_Visual`
  → `CSE_ALifeObject`): position/angle, `s_flags` (`M_SPAWN_OBJECT_LOCAL` = client-controlled),
  `m_ini_string`, AI vertices. `Objects.net_Unregister` on destroy. See `engine/server-entities.md`.
- **Lua seam:** the `CScriptBinder` base drives the script object lifecycle (`net_Spawn`/`net_Destroy`/
  `shedule_Update`/save-load); `CScriptGameObject` is the Lua wrapper obtained via `lua_game_object()`;
  per-object callbacks live in the object's `callback()` map. See `modlayer/scripting.md`.
- **Other xrGame domains:** AI location (`CAI_ObjectLocation`, `ai_obstacle`), script attachments
  (`m_script_attachments`), and derived mixins (`CInventoryOwner` for owners, `CHudItem`/`CWeapon` for
  items — the weapon-fire flow plugs in here, `flows/weapon-fire-callbacks.md`).

## Threading

`UpdateCL` and `shedule_Update` both run on the main thread (the scheduler dispatch is serial — see
`engine/engine-core.md`). The bullet manager is the notable thing that *can* offload (its own flow).

## Failure & edge cases

- **Duplicate spawn ID** → `net_Spawn` logs and returns false → `g_sv_Spawn` calls `net_Destroy` +
  `Objects.Destroy` (object never enters the world).
- **Destruction is two-phase**: `DestroyObject` only *requests* (sets `m_bObjectRemoved`, sends
  `GE_DESTROY`); actual teardown is the later engine-driven `net_Destroy`. Don't assume the object is
  gone the frame `DestroyObject` ran.
- Base `CGameObject::NeedToDestroyObject` returns false — searching the base for self-destruct logic
  misleads; it's per-derived-class.

## Deeper

- Game object: `src/xrGame/GameObject.cpp/.h` (`net_Spawn`, `net_Destroy`, `UpdateCL`,
  `shedule_Update`, `shedule_Needed`, `DestroyObject`, `spawn_supplies`, `on_matrix_change`,
  `FootStepCallback`)
- Engine base: `src/xrEngine/xr_object.cpp/.h` (`CObject::net_Spawn`, `net_Destroy`,
  `processing_activate`, `MakeMeCrow`, `spatial_register`/`shedule_register`)
- Spawn pipeline: `src/xrGame/Level_network_spawn.cpp` (`CLevel::g_sv_Spawn`, `cl_Process_Spawn`)
- Object list / per-frame driver: `src/xrEngine/xr_object_list.cpp` (`CObjectList::SingleUpdate`)
- Callback enum: `src/xrGame/game_object_space.h`; Lua mapping `src/xrGame/script_game_object_script.cpp`

## Maintenance

- Re-grep `CGameObject::net_Spawn`, `net_Destroy`, `shedule_Update`, `UpdateCL`, `DestroyObject`, and
  `g_sv_Spawn` if they move; re-grep `_G.CGameObject_NetSpawn`/`NetDestroy` and `eNetSpawnAfter` to
  confirm the callback fires.
- Promote the *(agent-traced)* items (CObject::net_Destroy internals, `spawn_supplies` body,
  `CObjectList::SingleUpdate`, `cl_Process_Spawn`, `shedule_Needed`) to verified by reading them.
- This flow is the per-domain template's anchor; new xrGame domain flows should reference its
  spawn/update/destroy spine rather than re-describe it.
