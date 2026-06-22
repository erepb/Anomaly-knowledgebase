---
domain: engine/core
tags: [engine, cpp, scheduler, imgui]
sources: [src/xrEngine/device.cpp, src/xrEngine/XR_IOConsole.h, src/xrEngine/IInputReceiver.h, src/xrEngine/xr_input.h, src/xrEngine/xrSheduler.h, src/xrEngine/xrSheduler.cpp]
related: [engine/_overview.md, engine/renderer.md, modlayer/scripting.md, flows/engine-boot-frame.md, flows/lua-callback-lifecycle.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# xrEngine (host core)

The executable host: device, main loop, console, input, ImGui, and the Lua debug bridge. The
end-to-end **boot → device init → per-frame loop** ordering (and how the `seqFrame`/`seqRender`
registrators + the MT `GameThread` scheduler fit together) is the flow **`flows/engine-boot-frame.md`**.
Claims below
are verified from the named files, including the scheduler update internals.

## Key pieces (verified)

- **Device + main loop** — `device.cpp`: `message_loop()` pumps Win32 messages and calls `on_idle()`
  each iteration; `on_idle()` calls `FrameMove()`, which runs `Device.seqFrame.Process(rp_Frame)` —
  the per-frame update sequence. Device
  lifecycle is split across `Device_create.cpp`, `Device_Initialize.cpp`, `Device_destroy.cpp`,
  `Device_wndproc.cpp` (filenames verified; contents not all read).
- **Console** — `class ENGINE_API CConsole` (`XR_IOConsole.h`) with `Execute(cmd)`,
  `ExecuteScript`, `ExecuteCommand`. Lua `get_console():execute(...)` reaches this.
  Command files: `XR_IOConsole*.cpp`, `Text_Console*.cpp`.
- **Input** — `class ENGINE_API IInputReceiver` (`IInputReceiver.h`) with virtual
  `IR_OnKeyboardPress(int dik)`; device input in `xr_input.h`/`Xr_input.cpp`. *(Note: the Lua
  `on_key_press`/`on_key_release` callbacks are fired from `xrGame/Level_input.cpp` and engine
  line-edit code — verified by grep — not purely from this input layer.)*
- **ImGui** — `imgui_base_input.cpp` (file exists, verified by listing) integrates ImGui; the mod's
  debug/options UI uses it (`modlayer/options-ui.md`).
- **Lua debug bridge** — `ai_script_lua_debug.cpp`, `ai_script_lua_extension.cpp` (files exist) —
  engine side of script debugging / LuaPanda.

## Scheduler (verified)

`class ENGINE_API CSheduler` (`xrSheduler.h`): objects implement `ISheduled` and `Register(A, RT)`
with a real-time flag. `Update()` (`xrSheduler.cpp`) runs `UpdateRT()` then `UpdateDeferred()`.
**Verified — both are serial, on the calling (main) thread:** `UpdateRT` iterates `ItemsRT`
and calls `shedule_Update` on every item each frame. `UpdateDeferred` sets a time budget
(`cycles_limit` from `psShedulerCurrent` ms) and calls `ProcessStep`, which collects a batch
from the priority queue under an SRW lock (`ItemsLock`), then **processes it in a serial for-loop**
with a time-budget check every 8th item. The SRW lock makes *registration* thread-safe;
it does **not** parallelize execution — there is no worker-thread dispatch of the deferred batch
(corrects an earlier README-based "MT threads the batch" claim). Driven per frame via `device.cpp`
`seqFrame.Process`. See `GLOSSARY.md#scheduler`.

## Deeper

- Device/loop: `src/xrEngine/device.cpp` (`message_loop`, `on_idle`, `FrameMove`, `seqFrame`),
  `src/xrEngine/Device_*.cpp`
- Console: `src/xrEngine/XR_IOConsole.h/.cpp` (`CConsole::Execute*`), `Text_Console*.cpp`
- Input: `src/xrEngine/IInputReceiver.h` (`IR_OnKeyboardPress`), `src/xrEngine/Xr_input.cpp`;
  Lua key callbacks: `src/xrGame/Level_input.cpp`
- ImGui + Lua debug: `src/xrEngine/imgui_base_input.cpp`, `src/xrEngine/ai_script_lua_*.cpp`

## Maintenance

- If the threading model changes, verify/replace the scheduler section by reading the actual update
  dispatch (start at `device.cpp` `seqFrame`).
- Re-grep `CConsole`, `IInputReceiver`, `message_loop` if line numbers drift.
