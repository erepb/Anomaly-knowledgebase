---
domain: flows/alife-online-offline
tags: [server, alife, online, offline, switch, crash, hysteresis, flow]
sources: [src/xrGame/alife_switch_manager.cpp, src/xrGame/alife_dynamic_object.cpp, src/xrGame/alife_update_manager.cpp, src/xrGame/alife_switch_manager_inline.h, src/xrServerEntities/xrServer_Objects_ALife.cpp]
related: [flows/object-lifecycle.md, flows/save-load.md, flows/ai-squad-logic.md, flows/spawns-alife-graphs-configs.md, engine/server-entities.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: ALife online ↔ offline transition

How an ALife entity flips between **offline** (abstract `CSE_ALifeDynamicObject` simulated cheaply by
the simulator) and **online** (a real level `CGameObject` exists, spawned via the server). Driven by
actor proximity with a hysteresis band. This is the **crash-prone** seam — most of its complexity is
ordering guards that keep parent/child sets and registries consistent. Resolves the previously
*(INFERRED)* online-radius claim in `engine/server-entities.md`/`GLOSSARY.md`.

**Provenance:** spine personally code-verified — `CALifeUpdateManager::update_switch`,
`CSwitchPredicate`, `CALifeSwitchManager::switch_object`/`synchronize_location`/`try_switch_online`/
`try_switch_offline`/`add_online`/`remove_online`/`switch_online`/`switch_offline`, the object-side
`CSE_ALifeDynamicObject::try_switch_online`/`try_switch_offline`, the distance formulas in
`alife_switch_manager_inline.h`, and `CSE_ALifeObject`/`CSE_ALifeCreatureAbstract` `can_switch_*`.
`server().Process_spawn`/`Perform_destroy` bodies are confirmed call sites, internals *(agent-traced)*.

## Driver (verified)

- **`CALifeUpdateManager::update`** calls **`update_switch()`** (then `update_scheduled()`) each ALife
  tick. `update_switch` runs `graph().level().update(CSwitchPredicate(this), preload?)` over the
  current level registry.
- **`CSwitchPredicate`** — its filter `operator()` uses a per-object `m_switch_counter == cycle_count`
  guard so each object is evaluated once per cycle; its action `operator()` calls
  `switch_manager->switch_object(obj)`.

## Per-object decision (verified)

**`CALifeSwitchManager::switch_object(I)`**:
1. `I->redundant()` → `release(I)`, return.
2. `!synchronize_location(I)` → return (bad/invalid vertex; skip this object).
3. if `I->m_bOnline` → `try_switch_offline(I)` else → `try_switch_online(I)`.
4. re-check `redundant()` → `release(I)`.

**`synchronize_location(I)`** early-returns `true` (skip the vertex fix) when: not using AI locations;
attached (`ID_Parent != 0xffff` — the parent carries it); or offline with an invalid level vertex
(lets corpses with no map node survive). Otherwise validates/updates the level node via the object's
own `synchronize_location()`.

**`try_switch_online(I)`** (manager): if `I` is attached (`ID_Parent != 0xffff`) **return immediately**
(items only switch with their owner) — debug builds assert the parent is offline here. Otherwise call
the object's `try_switch_online()`; if it stayed offline and isn't keeping data, `clear_client_data()`.

**`try_switch_offline(I)`** (manager): same attached-guard (return; debug asserts parent online), then
the object's `try_switch_offline()`.

## Object-side proximity + hysteresis (verified)

**`CSE_ALifeDynamicObject::try_switch_online`**:
- maintains its schedulable membership (add/remove from `alife().scheduled()` based on `need_update`).
- `!can_switch_online()` → `on_failed_switch_online()`, return.
- `!can_switch_offline()` → force `alife().switch_online(this)` (object that can never go offline comes
  online unconditionally), return.
- distance: if `actor.distance_to(o_Position) > alife().online_distance()` → fail, return; else
  `alife().switch_online(this)`.

**`CSE_ALifeDynamicObject::try_switch_offline`**:
- `!can_switch_offline()` → return (stays online).
- `!can_switch_online()` → `alife().switch_offline(this)`, return.
- distance: if `actor.distance_to(o_Position) <= alife().offline_distance()` → return (stay online);
  else `alife().switch_offline(this)`.

**Distances** (`alife_switch_manager_inline.h`): read once from the **`"alife"` system.ltx section**
(`switch_distance`, `switch_factor`), then `online_distance = switch_distance * (1 - factor)` and
`offline_distance = switch_distance * (1 + factor)`. Since `online_distance < offline_distance`, the
band between them is a **hysteresis dead-zone** that prevents online/offline oscillation at the
boundary.

**`can_switch_*` flags** (`CSE_ALifeObject`): `can_switch_online = match_configuration() &&
flSwitchOnline`; `can_switch_offline = !match_configuration() || flSwitchOffline`. Creatures override
`can_switch_offline` to also require `get_health() > 0` — **a dead creature cannot go offline**, so its
corpse stays online until removed (intentional, prevents "zombie" abstract corpses).

## The transition mechanics (verified spine; server internals agent-traced)

**`add_online(object, update_registries)`** (offline→online): `VERIFY` the object's game-graph vertex
is on the current level; set `m_bOnline = true`; `server().entity_Destroy(abstract)` (drop the
server-only copy); set the `M_SPAWN_UPDATE` flag; `server().Process_spawn(...)` (create the level
object + broadcast); clear `M_SPAWN_UPDATE`; `object->add_online(update_registries)` (registry
cleanup). A commented-out `R_ASSERT` documents a historical crash with corpses outside the AI map.

**`remove_online(object, update_registries)`** (online→offline): set `m_bOnline = false`; stash
`m_saved_chidren = object->children`; `server().Perform_destroy(object, net_flags(TRUE,TRUE))`
(recursively detach/destroy children); **`VERIFY(object->children.empty())`** — the key crash guard:
if a child wasn't detached, this fires; regenerate the id via `PerformIDgen`; `object->add_offline(
m_saved_chidren, update_registries)` (re-register offline with its saved children).

`switch_online`/`switch_offline` are thin wrappers that call the object's own
`switch_online()`/`switch_offline()` (which internally call `add_online`/`remove_online`).

## Isolation ledger (the seams)

- **Server ↔ client object:** `add_online` → `server().Process_spawn` creates a `CGameObject`
  (`flows/object-lifecycle.md`, `net_Spawn`); `remove_online` → `server().Perform_destroy` tears it
  down (`net_Destroy`). State carries on the spawn/update packets.
- **Registries:** `add_online`/`add_offline` move the object between the scheduled set and the level/
  graph registries; the schedulable membership is also adjusted in `try_switch_online`.
- **Config:** `switch_distance`/`switch_factor` from the `"alife"` section (runtime-tunable via
  `set_switch_distance`/`set_switch_factor`, also exposed to Lua as `alife():switch_distance()` etc.).
- **Lua:** `on_register`/`on_unregister` server callbacks fire during the registry moves; squad/logic
  binders react to objects coming online (`flows/ai-squad-logic.md`). The actor position read each
  evaluation is `alife().graph().actor()->o_Position`.
- **Save/load:** offline entities are exactly what gets serialized (`flows/save-load.md`); a load
  re-registers everything offline, then proximity brings near objects online.

## Threading

Runs inside the ALife update on the game-scheduler path (see `flows/engine-boot-frame.md` — the serial
scheduler on `GameThread`). `update_switch` walking the level registry is **not re-entrant**; spawn/
destroy go through the single server entity map.

## Failure & edge cases (the crash surface)

- **Orphaned children:** the `VERIFY(object->children.empty())` after `Perform_destroy` is the canary
  for an incomplete child detach — a leftover child means the next switch corrupts the child list.
- **Attached items switching alone:** blocked by the `ID_Parent != 0xffff` returns in
  `try_switch_online`/`offline`; the debug `VERIFY2`s catch parent/child online-state mismatches
  ("Parent online, item offline" / vice-versa).
- **Invalid vertex:** `synchronize_location` skipping (offline + invalid vertex) lets corpses survive,
  but bringing such an object online with a bad graph vertex would hit the `add_online` `VERIFY`.
- **Dead creatures pinned online:** `can_switch_offline` health guard keeps corpses online by design.
- **Spawn after destroy ordering:** `add_online` destroys the server copy *before* `Process_spawn`; a
  spawn failure there leaves the object without a server entity (limbo) — order is load-bearing.

## Deeper

- Manager: `src/xrGame/alife_switch_manager.cpp` (`switch_object`, `synchronize_location`,
  `try_switch_online/offline`, `add_online`, `remove_online`, `switch_online/offline`)
- Distances: `src/xrGame/alife_switch_manager_inline.h` (ctor reads `"alife"` section; `online_/
  offline_distance`, `set_switch_distance`)
- Object decision: `src/xrGame/alife_dynamic_object.cpp` (`try_switch_online`, `try_switch_offline`,
  `redundant`, `on_failed_switch_online`)
- Driver: `src/xrGame/alife_update_manager.cpp` (`update`, `update_switch`, `CSwitchPredicate`)
- Flags: `src/xrServerEntities/xrServer_Objects_ALife.cpp` (`CSE_ALifeObject::can_switch_*`),
  `src/xrServerEntities/xrServer_Objects_ALife_Monsters.cpp` (creature health guard)
- Spawn/destroy internals: `server().Process_spawn`/`Perform_destroy` (`engine/server-entities.md`)

## Maintenance

- Re-grep the `try_switch_online`/`try_switch_offline` pair (manager + object) if proximity behavior
  changes; the hysteresis formula lives in `alife_switch_manager_inline.h`.
- Keep the `"alife"` section as the documented source of `switch_distance`/`switch_factor` (verify in
  the shipped `system.ltx` if exact numbers are needed).
- If the child-detach `VERIFY` or the `add_online` spawn-order changes, update the crash surface notes
  here — this flow is referenced specifically for transition crashes.
- Promote `Process_spawn`/`Perform_destroy` internals to verified if a task needs the exact packet/
  broadcast detail.
