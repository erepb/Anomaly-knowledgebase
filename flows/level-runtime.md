---
domain: flows/level-runtime
tags: [xrgame, level, object-collection, frame, input, network, flow]
sources: [src/xrGame/Level.cpp, src/xrGame/Level.h, src/xrGame/Level_input.cpp, src/xrGame/Level_load.cpp, src/xrGame/Level_network_spawn.cpp]
related: [flows/engine-boot-frame.md, flows/object-lifecycle.md, flows/weapon-fire-callbacks.md, flows/pda-map-news.md, flows/render-frame.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: CLevel runtime (the rest of Level.cpp)

What the game-side **level object** (`CLevel`, one per loaded level) does each frame, on input, and at
load — beyond the ballistics (`CBulletManager`, `flows/weapon-fire-callbacks.md`). This is the
orchestrator that ticks the object collection, processes network spawn/events, drives map/task updates,
and routes input. It sits inside the engine frame loop (`flows/engine-boot-frame.md`) and is the seam
where objects register/update/destroy (`flows/object-lifecycle.md`). **Resolves the residual
`(INFERRED)` "rest of Level.cpp" note in `engine/xrGame.md`.**

**Provenance:** spine personally code-verified — `CLevel : IGame_Level, IPureClient` and
`CLevel::OnFrame` exist (the per-frame entry, which notably updates wallmarks before render —
`flows/wallmarks-particles.md`). The detailed per-frame/OnRender/input/load sequences, the owned
sub-managers, and the object-collection (`CObjectList`) mechanics are confirmed homes but
*(agent-traced — not personally re-read)*.

## What CLevel is (hierarchy verified; ownership agent-traced)

`CLevel : IGame_Level, IPureClient` (verified) — `IGame_Level` brings the **object collection**
(`CObjectList Objects` — every `CGameObject` on the level, by net-ID) and the engine frame/render/input
interfaces (`pureFrame`/`pureRender`/`IInputReceiver`); `IPureClient` is the network client. Reached
globally as **`Level()`**. It owns *(agent-traced)*: `CBulletManager` (`flows/weapon-fire-callbacks.md`),
`CMapManager` + `CGameTaskManager` (`flows/pda-map-news.md`, `flows/tasks.md`), the seniority/agent
hierarchy (`flows/agent-squad-combat.md`), the client-spawn manager, space-restriction manager, level
sound manager, the physics commanders, and static particles.

## Per-frame — `CLevel::OnFrame` (entry verified; sequence agent-traced)

The level's frame work, inside the engine `seqFrame` (`flows/engine-boot-frame.md`):
1. update wallmarks (verified, before render — `flows/wallmarks-particles.md`); `BulletManager().
   CommitEvents()` (finalize last frame's hits).
2. **`ClientReceive`** → network packets into a `game_events` queue.
3. **`ProcessGameEvents`** → dispatch `M_SPAWN` (`cl_Process_Spawn`) / `M_EVENT` (`cl_Process_Event`),
   with prefetch anti-freeze for spawns.
4. `MapManager().Update()` + `GameTaskManager().UpdateTasks()`.
5. **`inherited::OnFrame`** → **`Objects.Update()`** (tick every object's scheduled update —
   `flows/object-lifecycle.md`) + HUD frame.
6. physics commanders (main + script), the **level-scope Lua script processor**, level sound update,
   Lua GC step, script attachments.
`OnRender` adds the 3D PDA, HUD/UI render, bullet traces, the **Lua debug-render queue**, and debug
overlays. *(sequence agent-traced.)*

## Input — `Level_input.cpp` (agent-traced)

`CLevel` is the `IInputReceiver`: `IR_OnKeyboardPress`/`Release`/`Hold` + mouse route input in order —
**Lua callbacks** (`level_input.on_key_*`, `_G.COnMouseWheel`) → UI → game → **current entity**
(actor/control). Mouse buttons map to keys. Debug/cheat keys (non-gold builds): F7 reload, numpad `*`
1000× timefactor, F4 cycle entities, etc. `IR_OnActivate` re-fires held movement/fire keys on focus.

## Load / start — `Level_load.cpp` (agent-traced)

Level load runs `Load_GameSpecific_Before` (ALife sim if single-player, level/game graph, patrol paths,
reset communities/ranks/reputation), the inherited geometry/render/`Objects` load, then
`Load_GameSpecific_After` (static particles, static sounds + environment/occlusion, level scripts, set
time/timefactor, enable HUD), and `Load_GameSpecific_CFORM` (remap collision triangle materials to
`GameMtl` indices — the seam to `flows/wallmarks-particles.md`). Net start is the
`net_Start`/`net_start_client*` handshake.

## Object collection (agent-traced) — the F-row seam

`CObjectList Objects` (from `IGame_Level`) stores active + sleeping objects, a destroy queue, and a
net-ID→object map (`net_Find` is O(1)). The lifecycle (`flows/object-lifecycle.md`): a spawn packet →
`cl_Process_Spawn` → `g_sv_Spawn` → `Objects.Create` → the object's `net_Spawn` (registers with the
level via `net_Register`) → per-frame `Objects.Update` ticks it → `setDestroy`/`Destroy` queues it →
`ProcessDestroyQueue` runs relcase callbacks + `net_Unregister` + frees it. `net_Export`/`net_Import`
replicate active-object state.

## Isolation ledger (the seams)

- **Engine frame loop:** `CLevel` is `pureFrame`/`pureRender` — `OnFrame`/`OnRender` run inside
  `seqFrame`/`seqRender` (`flows/engine-boot-frame.md`).
- **Object lifecycle:** `Objects` is *the* collection; `net_Spawn` registers, `Objects.Update` ticks,
  `ProcessDestroyQueue` destroys (`flows/object-lifecycle.md`).
- **Network:** `IPureClient` — `ClientReceive`/`ClientSend`, spawn/event packets → game_events.
- **Map/tasks:** `MapManager`/`GameTaskManager` updated here (`flows/pda-map-news.md`, `flows/tasks.md`).
- **Bullets/wallmarks:** `BulletManager` committed + wallmarks updated each frame
  (`flows/weapon-fire-callbacks.md`, `flows/wallmarks-particles.md`).
- **Lua:** the `level.*` surface (`spawn`, `object`/`object_by_id`, weather/time, debug render) lives in
  `level_script.cpp`; the level-scope script processor + `level_input.*` callbacks run from `OnFrame`/
  input.
- **Config:** level files (geometry/spawn/graph/`level.ltx`), `level.snd_env`, static particles
  (`level.ps_static`).

## Threading

`OnFrame`/`OnRender`/input on the main thread; several sub-updates (map, physics commanders, level
sound, script GC) are flagged parallelizable. Object scheduler is serial
(`flows/engine-boot-frame.md`). Network on the client/server update.

## Failure & edge cases

- **Objects are net-ID keyed:** lookups are by net-ID (`net_Find`); a wrong/stale ID returns null. The
  destroy queue is **deferred** — objects die at `ProcessDestroyQueue`, not inline (prevents
  mid-iteration frees), same principle as the inventory/UI deferred deletes.
- **Spawn prefetch anti-freeze:** spawns can be postponed across frames to avoid hitches — a spawned
  object may not exist the same frame the packet arrived.
- **Input order matters:** input goes Lua→UI→game→entity; a Lua `on_key_press` returning handled, or a
  UI receiver, eats the key before the actor sees it.
- **`Level()` only valid in a level:** the global is null in the main menu / between levels; level-scope
  Lua and `level.*` calls require a loaded level.
- **CFORM material remap at load:** collision triangle materials are remapped to `GameMtl` indices on
  load — the basis for impact wallmarks/particles/sounds (`flows/wallmarks-particles.md`).

## Deeper

- Core + frame: `src/xrGame/Level.cpp`/`.h` (`OnFrame`, `OnRender`, `ProcessGameEvents`,
  `ProcessSpawnEvents`, the owned managers, `Level()`)
- Input: `src/xrGame/Level_input.cpp` (`IR_OnKeyboard*`/`IR_OnMouse*`, debug keys)
- Load: `src/xrGame/Level_load.cpp` (`Load_GameSpecific_Before`/`After`/`CFORM`)
- Spawn/objects: `src/xrGame/Level_network_spawn.cpp` (`cl_Process_Spawn`, `g_sv_Spawn`),
  `IGame_Level`/`xr_object_list` (`CObjectList`, `Objects.Update`, `ProcessDestroyQueue`) —
  `flows/object-lifecycle.md`
- Lua: `level_script.cpp` (`level.*`); frame loop: `flows/engine-boot-frame.md`

## Maintenance

- Re-grep `CLevel::OnFrame` if the per-frame order changes — it is the game-side frame orchestrator
  inside `seqFrame`.
- Keep the object-collection lifecycle aligned with `flows/object-lifecycle.md` (this flow is the
  level-side registration/update/destroy seam).
- This flow resolves `engine/xrGame.md`'s residual `(INFERRED)` "rest of Level.cpp" — keep that pointer
  in sync. Promote the *(agent-traced)* per-frame/load/input sequences to verified when a task needs the
  exact ordering.
