---
domain: flows/engine-boot-frame
tags: [engine, cpp, boot, frame, device, scheduler, threading, flow]
sources: [src/xrEngine/x_ray.cpp, src/xrEngine/EngineAPI.cpp, src/xrEngine/device.cpp, src/xrEngine/device.h, src/xrEngine/EngineThreading.cpp, src/xrEngine/pure.h]
related: [flows/object-lifecycle.md, flows/render-frame.md, engine/engine-core.md, engine/_overview.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: engine boot → device init → frame loop

From process entry through engine/device initialization to the per-frame loop that drives input, the
scheduler, game logic, and rendering. This is the outermost loop everything else runs inside —
object updates (`flows/object-lifecycle.md`) and the render frame (`flows/render-frame.md`) are steps
within it. The scheduler and device internals are in `engine/engine-core.md`; the compile-time
renderer selection is in `engine/_overview.md` — this flow is the end-to-end ordering that ties them
together.

**Provenance:** researched with a read-only agent, then spot-verified against code. The spine homes
(`WinMain`/`WinMain_impl` in `x_ray.cpp`; `CEngineAPI::Initialize`/`InitializeNotDedicated`;
`CRenderDevice::message_loop`/`on_idle`/`FrameMove`; the `seqFrame`/`seqRender`/`seqAppStart`/
`seqFrameMT` registrators in `device.h`) are personally code-verified. The detailed bodies (exact
init order inside `WinMain_impl`, the secondary-thread task contents) are *(agent-traced — not
personally re-read)*.

## Boot sequence (verified homes; order agent-traced)

1. **`WinMain` → `WinMain_impl`** (`x_ray.cpp`) — process entry (SEH-wrapped). Early init (LuaJIT
   pool, DPI), xrCore/xrPhysics DLL attach, single-instance mutex, splash. Then config boot:
   `Core._initialize` (parses `-fsltx`), `InitSettings` (loads `system.ltx`/`game.ltx` via `CInifile`).
2. **`InitEngine`** — binds `Device` singleton, `Engine.Initialize` (scheduler init via
   `Engine.Sheduler.Initialize`), then `Device.Initialize` (window class + timers).
3. **Renderer selection** — `Engine.External.CreateRendererList` builds the quality tokens; the actual
   renderer (R1–R4) is selected and statically bound (`engine/_overview.md`: `CEngineAPI`).
   `CEngineAPI::Initialize`/`InitializeNotDedicated` (`EngineAPI.cpp`, verified) connects the device
   to the chosen render backend and loads `xrGame` (binds the `pCreate`/`pDestroy` object factory).
4. **`Startup`** (`x_ray.cpp`) — sound init, `execUserScript` (`user.ltx`, which carries the
   `renderer`/`vid_quality` console vars), parse `-start`/`-load`. Then **`Device.Create`** (window +
   render device + shaders + `PreCache`), create `CApplication` (`pApp`) and `g_pGamePersistent`, the
   spatial DBs. Finally **`Device.Run`** — enter the loop.

## Frame sequence (verified homes; per-step detail agent-traced)

`CRenderDevice::Run` spawns the helper threads and calls `seqAppStart`, then enters
**`message_loop`** (verified). Each iteration:

1. **Message pump** — `PeekMessage`/`Translate`/`Dispatch`; `WM_QUIT` exits; activation/close routed
   through `WndProc` → `OnWM_Activate` (`seqAppActivate`/`seqAppDeactivate`). With no message → call
   **`on_idle`**.
2. **`on_idle`** (verified) — the per-frame kernel. If loading, run the loading events + draw and
   return early. Otherwise:
   - **`FrameMove`** (verified) — advance `dwFrame`, compute `fTimeDelta` (exponential-smoothed, or
     fixed under constant-FPS; 0 when paused), update `fTimeGlobal`/`dwTimeGlobal`, then
     **`seqFrame.Process(rp_Frame)`** — dispatches `OnFrame` to all `pureFrame` subscribers (input
     poll, **the game's `CApplication::OnFrame`**, sound).
   - **Secondary tasks** (`EngineThreading.cpp`) — `PreRenderThread` (rain/particles),
     `CalculateBonesThread` (skinned-bone calc from the saved frustum), and **`GameThread`** which
     runs the **scheduler** (`Engine.Sheduler.UpdateDeferred`/`Update` — the time-budgeted **serial**
     dispatch, `engine/engine-core.md`), `seqParallel`, `seqFrameMT`, then parallel Lua GC.
     **Order matters (2026.11.10):** `seqFrameMT` runs *before* the parallel GC because physics contact
     callbacks inside it can enter Lua; running GC first leaked Lua objects (attachment mod).
   - **Render** — `Begin` (device-state check) → set view/projection (save matrices for next frame's
     bone calc) → **`seqRender.Process(rp_Render)`** (all `pureRender` subscribers: game/HUD/stats) →
     `End` (present; precache countdown; gamma).
   - **Sync** — wait on the secondary task group before the next frame.

## Isolation ledger (the seams)

- **Message-registry seams (the plugin contract, verified in `device.h`):** `seqFrame`
  (`pureFrame::OnFrame`), `seqRender` (`pureRender::OnRender`), `seqFrameMT` (MT frame),
  `seqAppStart`/`seqAppEnd`, `seqAppActivate`/`seqAppDeactivate`, plus `seqParallel`. Subsystems
  (incl. xrGame's `CApplication`) register here to get called each frame/phase — this is how the game
  plugs into the engine loop without the engine knowing about it.
- **Game-logic hook:** xrGame is bound at boot via the object factory (`CEngineAPI`); its per-frame
  entry is `CApplication::OnFrame` on `seqFrame`, plus `g_hud->OnFrameMT` and the scheduler-driven
  object updates on `GameThread`.
- **Threading:** main thread = message pump + `on_idle` (owns device Begin/End). A secondary task
  group runs rain/bones/game-scheduler/`seqFrameMT` concurrently and is **joined before render End**.
  The scheduler dispatch itself is serial (`engine/engine-core.md`); MT here parallelizes *distinct
  task groups*, not the scheduler queue.
- **Config read at boot:** `fs.ltx` (FS roots), `system.ltx`/`game.ltx` (engine/content),
  `user.ltx` (console vars incl. renderer/video), command line (`-fsltx`/`-start`/`-load`/…).
- **Links to other flows:** object `UpdateCL` (real-time) runs in the render/seqFrame path and
  `shedule_Update` (deferred) runs under the scheduler on `GameThread` (`flows/object-lifecycle.md`);
  the render step (`seqRender` → backend `CRender::Render`) is `flows/render-frame.md`; Lua boot
  (`on_game_start` etc.) is `flows/lua-callback-lifecycle.md`.

## Threading

Main thread drives the loop and the render device; a joined secondary task group runs particles,
bone calc, and the game scheduler/`seqFrameMT` each frame. The serial scheduler runs inside
`GameThread`.

## Failure & edge cases

- **Boot/load fixes (2026-09 → 2026-11):** `-nosplashwindow` sizes the splash window to zero
  (`x_ray.cpp`); `dump_cvar_enable` (default on) gates the cvar dump at start/crash; an alt-tab during
  loading no longer issues an unbalanced `Device.Pause` (missing sounds); on a zone transition the
  destination's loading artwork is chosen before the reload begins (opt-in `g_loading_screen_early`,
  also a Modded Exes option; `x_ray.cpp`, hinted from `alife_update_manager.cpp`).
- **`fTimeDelta` is smoothed and clamped** (and zero when paused) — game logic that reads it gets a
  filtered dt, not the raw frame time; a stall doesn't produce a huge dt.
- **Loading frames short-circuit `on_idle`** (run load events + draw, return) — normal frame steps
  don't run while a level loads.
- **Secondary tasks are joined before present** — a hang in `GameThread`/bone calc stalls the frame;
  the MT split is not fire-and-forget.
- **Renderer is fixed at boot** (compile-time/static selection + `user.ltx`), not switchable mid-run.

## Deeper

- Entry/boot: `src/xrEngine/x_ray.cpp` (`WinMain`, `WinMain_impl`, `InitEngine`, `Startup`),
  `src/xrEngine/EngineAPI.cpp` (`CEngineAPI::Initialize`, `InitializeNotDedicated`)
- Loop: `src/xrEngine/device.cpp` (`Run`, `message_loop`, `on_idle`, `FrameMove`, `Begin`, `End`),
  `src/xrEngine/device.h` (the `seq*` registrators)
- MT: `src/xrEngine/EngineThreading.cpp` (`PreRenderThread`, `CalculateBonesThread`, `GameThread`)
- Registrator: `src/xrEngine/pure.h` (`CRegistrator<T>::Process`)
- Scheduler/device detail: `engine/engine-core.md`; renderer selection: `engine/_overview.md`

## Maintenance

- Re-grep `CRenderDevice::on_idle`/`message_loop`/`FrameMove` and the `seq*` registrator fields in
  `device.h` if the loop shape changes.
- Promote the *(agent-traced)* detail (the exact `WinMain_impl` init order, the secondary-thread task
  bodies) to verified by reading them when a task needs boot-order precision.
- Keep the "serial scheduler inside an MT task group" framing aligned with `engine/engine-core.md`
  (MT parallelizes task groups, not the scheduler queue).
