---
domain: flows/renderer-factory-bridges
tags: [engine, renderer, ui, flow]
sources: [src/Layers/xrRender/dxRenderFactory.cpp, src/Layers/xrRender/dxRenderFactory.h, src/Layers/xrRender/dxRenderDeviceRender.cpp, src/Layers/xrRender/dxRenderDeviceRender.h, src/Layers/xrRender/dxUIRender.cpp, src/Layers/xrRender/dxUIShader.cpp, src/Layers/xrRender/dxFontRender.cpp, src/Layers/xrRender/dxConsoleRender.cpp, src/Layers/xrRender/dxDebugRender.cpp, src/Layers/xrAPI/xrAPI.cpp, src/Layers/xrRenderPC_R4/xrRender_R4.cpp]
related: [flows/hud-ui-load.md, flows/renderer-backend.md, flows/renderer-hw-device.md, flows/renderer-resources-shaders.md, flows/engine-boot-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: render factory & dx*Render bridges (engine ↔ Layers seam)

How the renderer-agnostic engine reaches the concrete renderer: the global interface pointers, the
`dxRenderFactory` that hands back per-subsystem render implementations, the device-glue
(`dxRenderDeviceRender`), and the UI/font/console/debug bridges that all funnel to `RCache`. This is
the structural seam between `xrEngine`/game subsystems and `src/Layers/`.

**Provenance:** researched with a read-only agent, then spot-verified against code. The factory class
(`dxRenderFactory`, abstract base commented out), the **global pointer wiring** (`::Render =
&RImplementation`, `::RenderFactory = &RenderFactoryImpl`, `UIRender = &UIRenderImpl`, `DRender =
&DebugRenderImpl` in `xrRender_R4.cpp`; declared in `xrAPI.cpp`), the device-glue entries
(`dxRenderDeviceRender::Create`/`Reset`/`End` — `End` holds the `m_pSwapChain->Present`), and the UI
bridge entries (`dxUIRender::StartPrimitive`/`FlushPrimitive`/`SetShader`) are personally code-verified.
The per-bridge bodies (font/console/debug/stats geometry fills) are *(agent-traced — not personally
re-read)*.

## Trigger / entry

At renderer load, `xrRender_R4.cpp`'s DLL entry wires the global pointers. During engine device init
(`flows/engine-boot-frame.md`), the engine calls `RenderFactory->Create*()` to obtain each subsystem's
render implementation. Per frame, the engine drives `dxRenderDeviceRender` (Begin/Clear/End) and UI/
font/etc. submit through their bridges.

## Sequence (verified structure)

### The factory pattern

The engine is renderer-agnostic: it depends only on abstract interfaces (`IRender_interface`,
`IRenderDeviceRender`, `IUIRender`, `IUIShader`, `IFontRender`, `IConsoleRender`, `IDebugRender`, … in
the engine include tree). Layers provides the concrete `dx*` implementations. The wiring (verified):
- `xrAPI.cpp` declares the globals `Render` (`IRender_interface*`), `RenderFactory`, `UIRender`,
  `DRender`.
- `xrRender_R4.cpp` (DLL attach) assigns `::Render = &RImplementation` (the `CRender` instance),
  `::RenderFactory = &RenderFactoryImpl`, `UIRender = &UIRenderImpl`, `DRender = &DebugRenderImpl`.
- `dxRenderFactory` exposes `Create*/Destroy*` pairs (`xr_new<dx…>` / `xr_delete`) for each subsystem
  (UIShader, RenderDeviceRender, FontRender, ConsoleRender, DebugRender, WallMarkArray, EnvironmentRender,
  Rain/Thunderbolt/LensFlare, ObjectSpace, Stats, …). The render path (R1/R2/R3/R4) is fixed at compile
  time (`engine/renderer.md`); this is the indirection that keeps the engine path-independent.

### Device glue — `dxRenderDeviceRender`

Implements `IRenderDeviceRender`; the bridge between `CRenderDevice` (engine) and `CHW` + `CResourceManager`
+ `RCache`:
- **`Create`** — `HW.CreateDevice` then allocate `Resources` (`CResourceManager`).
- **`OnDeviceCreate`** — `RCache.OnDeviceCreate`, `Resources->OnDeviceCreate`, `::Render->create`
  (R4 init).
- **`Reset`** — the three-phase mode-change: `Resources->reset_begin` → `HW.Reset` → `Resources->reset_end`
  (`flows/renderer-hw-device.md`, `flows/renderer-resources-shaders.md`).
- **`Begin`/`Clear`/`End`** — frame bracket; **`End` does the present** (`m_pSwapChain->Present(interval,
  flags)`, preceded by a `Present(0, DXGI_PRESENT_TEST)` device-lost probe — the verified present site
  referenced by `flows/renderer-hw-device.md`).

### UI bridge — `dxUIRender` / `dxUIShader`

The engine UI (`flows/hud-ui-load.md`) draws through `dxUIRender` (`IUIRender`): `StartPrimitive(maxVerts,
type, pointType)` locks a dynamic VB (`R_DStreams`), `PushPoint(x,y,z,color,u,v)` writes transformed-lit
or lit verts, `FlushPrimitive` unlocks + `set_Geometry` + `RCache.Render`. `SetShader` binds a
`dxUIShader` (a cached `Shader*` for the UI material, keyed by texture+shader). All UI geometry thus
becomes `RCache` draws (`flows/renderer-backend.md`).

### Other bridges

Each engine subsystem has a thin `dx*Render` that fills dynamic geometry and submits to `RCache`:
`dxFontRender` (glyph quads via the quad IB), `dxConsoleRender` (console background quad), `dxDebugRender`
(colored line lists batched by color, world + HUD projections), `dxObjectSpaceRender` (debug OBB/ellipse,
DEBUG only), `dxStatsRender`/`dxStatGraphRender` (perf text/graphs), `gifPlayer`. *(bodies agent-traced.)*

## Function digests (spine)

- **`xrRender_R4.cpp` DLL attach** — *Does:* wire `Render`/`RenderFactory`/`UIRender`/`DRender` to the
  R4 singletons. *(verified.)*
- **`dxRenderFactory.cpp:dxRenderFactory::Create*`** — *Does:* `xr_new` the concrete `dx*` impl for an
  interface; `Destroy*` deletes it. *Called by:* engine subsystem init. *(verified pattern.)*
- **`dxRenderDeviceRender.cpp:Create`/`OnDeviceCreate`/`Reset`/`End`** — *Does:* device lifecycle glue;
  `End` presents. *Calls:* `HW.*`, `Resources->*`, `RCache.*`, `m_pSwapChain->Present`. *Called by:*
  `CRenderDevice` (`flows/engine-boot-frame.md`). *(verified entries.)*
- **`dxUIRender.cpp:StartPrimitive`/`FlushPrimitive`/`SetShader`** — *Does:* stream UI geometry to
  `RCache`. *Calls:* `RCache.Vertex.Lock/Unlock`, `set_Geometry`, `Render`, `set_Shader`. *Called by:*
  engine UI (`flows/hud-ui-load.md`). *(verified entries.)*

## Isolation ledger (the seams)

- **Interface seam:** abstract `I*` interfaces in the engine include tree ↔ concrete `dx*` in
  `src/Layers/xrRender/`. The factory decouples creation.
- **Global pointers:** `Render`/`RenderFactory`/`UIRender`/`DRender` (declared `xrAPI.cpp`, assigned in
  `xrRender_R4.cpp`) — the engine's handle into the renderer.
- **Consumers:** engine UI (`flows/hud-ui-load.md`), font/console/debug/stats subsystems, the device
  (`CRenderDevice` → `dxRenderDeviceRender`).
- **Backend seam:** every bridge ultimately calls `RCache` (`flows/renderer-backend.md`) via dynamic
  streams.
- **Device seam:** `dxRenderDeviceRender` ↔ `CHW` (`flows/renderer-hw-device.md`) + `CResourceManager`
  (`flows/renderer-resources-shaders.md`); the present is `dxRenderDeviceRender::End`.
- **Config:** compile-time render path; vsync/screenshot flags consulted in `End`.

## Threading

All bridge submission + present run on the main render thread within the frame.

## Failure & edge cases

- **Present + device-lost** — `End` probes with `Present(0, DXGI_PRESENT_TEST)` before the real present;
  a lost device routes to reset.
- **UI shader caching** — `dxUIShader` dedups by texture+shader key; a per-frame `create` is cheap on
  a cache hit.
- **Renderer is fixed at compile time** — the factory does not swap render paths at runtime; a "wrong
  renderer" is a build/link issue (`engine/renderer.md`).
- **`DRender`/object-space bridges are DEBUG-weighted** — some are no-ops in release.

## Deeper

- Factory + globals: `xrRender/dxRenderFactory.cpp`/`.h`, `xrAPI/xrAPI.cpp` (global decls),
  `xrRenderPC_R4/xrRender_R4.cpp` (assignment)
- Device glue: `xrRender/dxRenderDeviceRender.cpp`/`.h` (`Create`/`OnDeviceCreate`/`Reset`/`End`)
- UI: `xrRender/dxUIRender.cpp`, `dxUIShader.cpp`; other bridges: `dxFontRender.cpp`,
  `dxConsoleRender.cpp`, `dxDebugRender.cpp`, `dxObjectSpaceRender.cpp`, `dxStats*Render.cpp`
- Consumers: `flows/hud-ui-load.md`; backend: `flows/renderer-backend.md`; device:
  `flows/renderer-hw-device.md`; boot: `flows/engine-boot-frame.md`

## Maintenance

- The global-pointer wiring (`xrRender_R4.cpp`) + `dxRenderDeviceRender::End` present are the spine —
  re-grep if the factory or device-glue changes.
- Promote the *(agent-traced)* per-bridge bodies (font/console/debug streaming) to verified when a UI/
  debug-render task needs exact behavior.
- Keep "engine talks to abstract interfaces; Layers provides concrete `dx*`; factory + globals connect
  them" — it's the whole point of this seam.
