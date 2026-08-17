---
domain: flows/alife-services
tags: [xrgame, alife, simulator, time, surge, emission, location, flow]
sources: [src/xrGame/alife_simulator.h, src/xrGame/alife_simulator_base2.cpp, src/xrGame/alife_update_manager.cpp, src/xrGame/alife_time_manager.cpp, src/xrGame/alife_time_manager_inline.h, src/xrGame/alife_surge_manager.cpp, src/xrGame/alife_spawn_registry_spawn.cpp, src/xrGame/location_manager.cpp, src/xrGame/alife_combat_manager.cpp]
related: [flows/alife-online-offline.md, flows/save-load.md, flows/engine-boot-frame.md, flows/ai-squad-logic.md, flows/spawns-alife-graphs-configs.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: ALife simulation services

The simulator-service managers that run the offline world beyond the online↔offline *switch*
(`flows/alife-online-offline.md`) and save *storage* (`flows/save-load.md`): **game time**, **surge
(emission) spawning**, **location/terrain**, and the (mostly inert) communication/interaction managers.
The headline fact: **the C++ surge manager is a spawn-only stub — the actual emission mechanic lives in
Lua** (`gamedata`/`unpacked` `surge_manager.script`). Verify which layer owns a behavior before
touching it.

**Provenance:** spine personally code-verified — `CALifeTimeManager::game_time` (the time-advance
formula) and `set_time_factor`, and that **`alife_surge_manager.cpp` contains only `spawn_new_spawns`/
`fill_spawned_objects`/`spawn_new_objects`** (no update/kill/emission/blowout — confirming the stub).
`CALifeUpdateManager::update`→`update_switch`/`update_scheduled` is verified in
`flows/alife-online-offline.md`. The `CALifeSimulator` ownership/accessors, `CLocationManager`, and the
commented-out communication/interaction managers are confirmed homes but *(agent-traced — not
personally re-read)*. The Lua `surge_manager.script` is base-game `unpacked/` (read-only reference).

## Simulator ownership (agent-traced; entry verified elsewhere)

`CALifeSimulator` (reached via `ai().alife()`) is the multi-inheritance root that owns the service
managers (storage, switch, time, surge, …) and exposes them by accessor (`ai().alife().time_manager()`
etc.). Its per-update entry is **`CALifeUpdateManager::update`** → `update_switch()` (proximity
online/offline, `flows/alife-online-offline.md`) + `update_scheduled()` (offline object ticks) — verified
in that flow. *(ownership/ctor agent-traced.)*

## Time manager (verified)

`CALifeTimeManager` advances game time independently of frames:
- **`game_time()`** = `m_game_time + _TIME_ID(m_time_factor · (Device.dwTimeGlobal − m_start_time))` —
  i.e. real elapsed ms × the **time factor**. This is what `Level().GetGameTime()` ultimately reads.
- **`set_time_factor(f)`** rebases (`m_game_time = game_time(); m_start_time = now`) then sets
  `m_time_factor` — so changing speed doesn't jump the clock. Base/normal factors come from `alife.ltx`
  (`time_factor`, `normal_time_factor`); Lua bumps the factor during a surge and restores it after.

Game time is the single clock the whole game reads (tasks' deadlines, day/night, ALife scheduling).

## Surge / emission — C++ stub vs Lua (verified split)

**C++ (`CALifeSurgeManager`, verified):** the entire `.cpp` is `spawn_new_objects` /
`fill_spawned_objects` / `spawn_new_spawns` — it **only instantiates spawn-registry objects**. There is
**no** emission scheduling, no wave timing, no NPC-killing, no actor damage in the engine surge manager.

**Lua (`surge_manager.script`, agent-traced, base-game `unpacked/`):** the real emission is a Lua
`CSurgeManager` state machine — schedules surges (random ~3–12h interval), runs a ~222s multi-stage
sequence (siren → blowout → waves → quake), applies psi/telepathic damage to the actor unless in cover,
and kills/zombifies/explodes offline NPCs not in safe smart-terrains. It calls `level.set_time_factor`
(the verified C++ time hook) to accelerate time during the surge. So **for any surge behavior question,
the answer is in the Lua `surge_manager`, not the C++ class** — the C++ side just spawns and advances
time. (This is a classic "Anomaly Lua supersedes the engine" case; treat the `unpacked/` script as
read-only reference, `flows/ai-squad-logic.md`.)

## Location & interaction managers (agent-traced)

- **`CLocationManager`** — maps objects to game-graph **terrain/location types** (vertex type masks),
  used by offline path/spawn placement; `Load`/`reload`/`add_location_type`/`vertex_types`. Not surge.
- **`CALifeCommunicationManager`** / **`CALifeInteractionManager`** — largely commented-out skeletons
  in this build (offline trade/interaction was never fully active). `CALifeCombatManager::kill_entity`
  is the one live utility (detach items, set death, drop from scheduled) the offline-death paths call.
  Don't document these as functional systems — they're stubs.

## Simulator update → service ticks (overview)

`CALifeUpdateManager::shedule_Update` → `update()` → `update_switch()` (online/offline) +
`update_scheduled()` (offline objects). Game time auto-advances via `CALifeTimeManager`. The surge is
driven from the **Lua** main loop (`surge_manager:update()`), not this C++ update — it only reaches into
C++ for `set_time_factor` and object spawn/kill utilities.

## Isolation ledger (the seams)

- **Online/offline switch:** `update_switch` (`flows/alife-online-offline.md`); surge only affects
  offline-eligible NPCs and respects safe smart-terrains.
- **Save/load:** time (`m_game_time`/`m_time_factor`) and the registries persist via the storage
  manager (`flows/save-load.md`); surge state persists Lua-side.
- **Scheduler:** the simulator update runs on the engine scheduler (`flows/engine-boot-frame.md`).
- **Lua (dominant for surge):** `surge_manager.script` owns emission timing/damage/cover; `level.
  set_time_factor`/`get_time_factor`, `level.GetGameTime` are the C++ hooks it uses. Actor survival
  (drugs/suit/helmet psi-protection) is computed in Lua `hit_power`.
- **Config:** `alife.ltx` (`time_factor`, `normal_time_factor`, schedule budgets); `surge_manager.ltx`
  (safe-cover zone list + enable condlist) — both consumed by the Lua surge.

## Threading

The ALife update runs on the engine scheduler (serial; optional MT-ALife batches switching —
`flows/alife-online-offline.md`). Time/surge are main-thread.

## Failure & edge cases

- **Looking for surge logic in C++ is a dead end:** the engine `CALifeSurgeManager` spawns only; timing/
  damage/cover/NPC-fate are all in `surge_manager.script`. Edit the Lua for surge behavior.
- **Time factor rebasing:** `set_time_factor` preserves the current time; a surge speeds time then
  restores `normal_time_factor` — a stuck fast/slow clock is usually a Lua factor not restored.
- **Communication/interaction are stubs:** offline trade/interaction managers are inert; don't rely on
  them for offline economy/combat (that's squad/ALife logic + Lua).
- **Game time is the universal clock:** anything time-gated (task deadlines, day/night) reads
  `game_time()`; messing with the factor affects all of them at once.
- **Spawn-graph ids are not all valid** (2026 crash fixes, MT). Three guards were added along the
  spawn path and are the shape of this bug class:
  `CALifeSpawnRegistry::fill_new_spawns` skips a leaf whose `m_tSpawnID` is `_SPAWN_ID(-1)` instead of
  pushing it; `CALifeSurgeManager::spawn_new_spawns` now null-checks `spawns().spawns().vertex(*I)`
  once and `continue`s (it used to dereference it three times inline); and
  `CALifeSimulatorBase::unregister_object` calls `scheduled().remove(object, true)` **unconditionally**
  before the online/offline branch, because `m_bOnline` cannot be trusted to reflect scheduler
  membership during destruction — that was the `CALifeUpdateManager::update` crash. If you touch
  registration/unregistration order, preserve this: remove from the scheduler first, then decide by
  online state.

## Deeper

- Time: `src/xrGame/alife_time_manager.cpp`/`alife_time_manager_inline.h` (`game_time`,
  `set_time_factor`, `init`, save/load)
- Surge (stub): `src/xrGame/alife_surge_manager.cpp` (`spawn_new_objects` only) — real surge is
  `unpacked/scripts/surge_manager.script` (`CSurgeManager`, read-only base-game)
- Update entry: `src/xrGame/alife_update_manager.cpp` (`update`/`update_switch`/`update_scheduled`) —
  `flows/alife-online-offline.md`
- Location/interaction: `src/xrGame/location_manager.cpp` (`CLocationManager`),
  `alife_combat_manager.cpp` (`kill_entity`), `alife_communication_manager.cpp`/
  `alife_interaction_manager.cpp` (stubs)
- Owner: `src/xrGame/alife_simulator.h` (`CALifeSimulator`) — `engine/server-entities.md`

## Maintenance

- Keep the **C++ surge = spawn-only stub / Lua = real emission** distinction prominent — re-grep
  `alife_surge_manager.cpp` for any `update`/kill method if a future build moves logic into C++.
- Keep `game_time()` = `m_game_time + factor·elapsed` accurate — it is the clock everything reads.
- Promote the *(agent-traced)* `CALifeSimulator` ownership and `CLocationManager` to verified when a
  task needs the exact accessor set or terrain-type usage.
