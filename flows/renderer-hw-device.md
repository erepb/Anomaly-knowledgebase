---
domain: flows/renderer-hw-device
tags: [engine, renderer, dx11, build, flow]
sources: [src/Layers/xrRender/HW.h, src/Layers/xrRender/HW.cpp, src/Layers/xrRenderDX10/dx10HW.cpp, src/Layers/xrRender/HWCaps.cpp, src/Layers/xrRender/HWCaps.h, src/Layers/xrRender/dxRenderDeviceRender.cpp]
related: [flows/engine-boot-frame.md, flows/renderer-backend.md, flows/renderer-resources-shaders.md, flows/renderer-factory-bridges.md, flows/render-frame.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-07
verified: code
---

# Flow: HW device, swapchain & capabilities

The lowest renderer layer: the D3D11 device + DXGI swapchain that everything else uses via
`HW.pDevice`/`HW.pContext`, the back-buffer render-target / depth-stencil, present, device reset/resize
(resolution or windowed↔fullscreen change), and GPU capability detection. Created during engine device
init (`flows/engine-boot-frame.md`), consumed by the backend (`flows/renderer-backend.md`), and
bracketed by the resource manager's reset (`flows/renderer-resources-shaders.md`) on a mode change.

**Provenance:** researched with a read-only agent, then spot-verified against code. The `CHW` structure
(`pDevice` = `ID3D11Device1*`, `pContext`, `m_pSwapChain` = `IDXGISwapChain1*`, `pBaseRT`/`pBaseZB`),
the create/reset/destroy entries (`CreateD3D`/`CreateDevice`/`UpdateViews`/`Reset`/`DestroyDevice` in
`dx10HW.cpp`), `CHWCaps::Update`, and the **present location** (`dxRenderDeviceRender.cpp`:
`HW.m_pSwapChain->Present(interval, flags)`, plus a `Present(0, DXGI_PRESENT_TEST)` device-lost probe;
`CHW::selectPresentInterval` picks vsync) are personally code-verified. Swapchain-desc details, caps
internals, and the fullscreen-transition handlers are *(agent-traced — not personally re-read)*.

## Trigger / entry

Engine device init (`flows/engine-boot-frame.md`) → `dxRenderDeviceRender::Create` → `HW.CreateDevice`.
On a resolution/mode change the device-render wrapper drives `reset_begin` → `HW.Reset` → `reset_end`.
Each frame ends with the device-render wrapper's present.

## Sequence (verified structure)

### What CHW owns

The global `CHW HW` holds the D3D11 device (`pDevice`, `ID3D11Device1*`), the immediate context
(`pContext`, `ID3D11DeviceContext1*` — what `RCache` issues every call through), the DXGI factory/
adapter/output, the swapchain (`m_pSwapChain`, `IDXGISwapChain1*`), the back-buffer render-target view
(`pBaseRT`) + a separately-created depth-stencil view (`pBaseZB`), the negotiated feature level, and the
caps (`CHWCaps`). (`HW.h` also declares DX9/DX10 members behind macros; this all-in-one build compiles
the DX11 path.)

### Device + swapchain creation

`CHW::CreateDevice(hwnd, …)` → `CreateD3D()` (DXGI factory, enumerate + select the adapter/output for
the window's monitor, detect tearing/VRR support) → `D3D11CreateDevice` (feature level 11_0) →
`CreateSwapChainForHwnd` (format R8G8B8A8 or R10G10B10A2 for HDR10, flip-discard, mode-switch/tearing
flags) → set color space → `UpdateViews()` creates the back-buffer RTV and a `D24S8` depth texture +
DSV. Caps (`CHWCaps::Update`) detect feature level, MSAA quality levels, vendor/VRAM, and gate renderer
options. *(swapchain-desc details + caps internals agent-traced.)*

### Reset / resize

On a resolution or windowed↔fullscreen change, the device-render wrapper calls `Resources->reset_begin`
(release device-bound renderer resources — `flows/renderer-resources-shaders.md`), then
**`CHW::Reset`** (release `pBaseRT`/`pBaseZB`, update the chain desc, `SetFullscreenState`,
`ResizeTarget`/`ResizeBuffers`, `UpdateViews` to recreate RTV/DSV, reposition the window), then
`Resources->reset_end` (recreate renderer resources). Fullscreen focus transitions
(`OnAppActivate`/`OnAppDeactivate`) similarly toggle fullscreen + resize. *(agent-traced.)*

### Present (verified location)

Frame end is in **`dxRenderDeviceRender::End`** (not `CHW`): it does a `HW.m_pSwapChain->Present(0,
DXGI_PRESENT_TEST)` device-lost probe, then `HW.m_pSwapChain->Present(present_interval, present_flags)`
to flip. `CHW::selectPresentInterval` chooses the vsync interval; tearing flag is used when VRR is
supported, windowed, and vsync is off.

## Function digests (spine)

- **`dx10HW.cpp:CHW::CreateDevice`** — *Does:* full device+swapchain bring-up (factory/adapter →
  `D3D11CreateDevice` → swapchain → `UpdateViews`). *Writes:* `pDevice`/`pContext`/`m_pSwapChain`/
  caps. *Called by:* `dxRenderDeviceRender::Create`. *(verified entry.)*
- **`dx10HW.cpp:CHW::CreateD3D`** — *Does:* DXGI factory + adapter/output selection + VRR detection.
  *(verified entry.)*
- **`dx10HW.cpp:CHW::UpdateViews`** — *Does:* (re)create back-buffer RTV + depth-stencil DSV from the
  current swapchain. *Called by:* `CreateDevice`, `Reset`, focus transitions. *(verified entry.)*
- **`dx10HW.cpp:CHW::Reset`** — *Does:* release views → resize swapchain (fullscreen/target/buffers) →
  `UpdateViews` → window props; coordinates with `Resources->reset_begin/end`. *(verified entry; body
  agent-traced.)*
- **`HWCaps.cpp:CHWCaps::Update`** — *Does:* detect feature level / MSAA levels / vendor / GPU count;
  gate renderer options. *(verified entry; internals agent-traced.)*
- **`dxRenderDeviceRender.cpp` present** — *Does:* device-lost `Present(0, TEST)` probe, then
  `m_pSwapChain->Present(interval, flags)`. *Calls:* `CHW::selectPresentInterval`. *Called by:* the
  frame loop's End (`flows/engine-boot-frame.md`). *(verified location.)*

## Isolation ledger (the seams)

- **Owned state:** the D3D11 device/context, DXGI factory/adapter/output/swapchain, base RTV/DSV,
  feature level, `CHWCaps`. Single global `HW`.
- **Creation seam:** engine boot → `dxRenderDeviceRender::Create` → `HW.CreateDevice`
  (`flows/engine-boot-frame.md`).
- **Consumption seam:** `RCache`/`CBackend` issue every D3D call through `HW.pContext`; `pBaseRT`/
  `pBaseZB` are the default frame targets (`flows/renderer-backend.md`); resource creation uses
  `HW.pDevice` (`flows/renderer-resources-shaders.md`).
- **Reset seam:** `Resources->reset_begin` → `HW.Reset` → `reset_end` on any mode change
  (`flows/renderer-resources-shaders.md`).
- **Present seam:** `dxRenderDeviceRender::End` flips the swapchain at frame end
  (`flows/render-frame.md`, `flows/engine-boot-frame.md`).
- **Caps seam:** `CHWCaps` gates which renderer features/MSAA modes are allowed (the compile-time render
  path is fixed; see `engine/renderer.md`).
- **Config:** resolution/`g_screenmode` (windowed/fullscreen), vsync, HDR10, MSAA, adapter/monitor
  selection.

## Threading

Device create/reset and present run on the main render thread; the immediate `pContext` is
single-threaded.

## Failure & edge cases

- **Device lost / occluded** is detected by the `Present(0, DXGI_PRESENT_TEST)` probe before the real
  present.
- **Reset must bracket renderer resources** — a missing `reset_begin`/`reset_end` around `HW.Reset`
  leaves dangling RTV/DSV/geometry on a mode change.
- **Fullscreen focus transitions** resize buffers — alt-tab in exclusive fullscreen goes through
  `OnAppActivate`/`OnAppDeactivate`.
- **HDR10** changes the swapchain format + color space — assumptions about R8G8B8A8 break under HDR10.

## Deeper

- Structure: `xrRender/HW.h` (`CHW`: `pDevice`/`pContext`/`m_pSwapChain`/`pBaseRT`/`pBaseZB`),
  `HW.cpp` (cross-API shell)
- DX11 device/swapchain: `xrRenderDX10/dx10HW.cpp` (`CreateD3D`, `CreateDevice`, `UpdateViews`,
  `Reset`, `DestroyDevice`, `selectPresentInterval`)
- Caps: `xrRender/HWCaps.cpp`/`.h` (`CHWCaps::Update`)
- Present + reset driver: `xrRender/dxRenderDeviceRender.cpp` (`End` present, `Reset`)
- Consumers: `flows/renderer-backend.md`; reset: `flows/renderer-resources-shaders.md`; boot/frame:
  `flows/engine-boot-frame.md`, `flows/render-frame.md`

## Maintenance

- `CHW::CreateDevice`/`Reset` + the `dxRenderDeviceRender` present are the spine — re-grep if the
  swapchain model (flip mode, HDR10, VRR) changes.
- Promote the *(agent-traced)* swapchain-desc, caps internals, and fullscreen-transition handlers to
  verified when a device/mode task needs exact behavior.
- Keep the "present lives in `dxRenderDeviceRender::End`, not `CHW`" fact — it's where the frame flips.
