---
domain: flows/ui-scaling
tags: [engine, cpp, ui, hud, renderer, config, lua, assets, flow]
sources: [src/xrGame/ui_defs.h, src/xrGame/ui_base.h, src/xrGame/ui_base.cpp, src/xrGame/UIStaticItem.cpp, src/xrGame/UICursor.cpp, src/xrGame/UIZoneMap.cpp, src/xrGame/HUDManager.cpp, src/xrGame/ui/UIXmlInit.cpp, src/xrGame/ui/UIWindow.cpp, src/xrGame/ui/uiabstract.h, src/xrGame/ui/UILines.cpp, src/xrGame/ui/UISubLine.cpp, src/xrGame/ui/UIFrameWindow.cpp, src/xrGame/ui/UIFrameLineWnd.h, src/xrGame/ui/UIFrameLineWnd.cpp, src/xrGame/ui/UITextureMaster.h, src/xrGame/ui/UIComboBox.cpp, src/xrGame/ui/UIEditKeyBind.h, src/xrGame/ui/UIInteractiveBackground.h, src/xrGame/ui/UIMap.cpp, src/xrGame/ui/UITabControl_scroll.cpp, src/xrGame/MainMenu.cpp, src/xrEngine/GameFont.h, src/xrEngine/GameFont.cpp, src/xrEngine/XR_IOConsole.cpp, src/xrEngine/Device_destroy.cpp, src/Layers/xrRender/dxFontRender.cpp, src/Layers/xrRender/dxUIRender.cpp, src/Layers/xrRender/FVF.h, gamedata/configs/ui/textures_descr, unpacked/shaders/r3/stub_notransform_t.vs, unpacked/configs/fonts.ltx, unpacked/scripts/utils_xml.script]
related: [flows/hud-ui-load.md, flows/gameplay-ui-screens.md, flows/actor-menu-inventory-ui.md, flows/pda-map-news.md, features/pda-tab-scroll.md, flows/renderer-backend.md, flows/utils-libraries.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: UI resolution scaling (the 1024x768 virtual canvas)

How XML layout coordinates become device pixels at any resolution. The model: every layout is
authored against a **fixed 1024x768 virtual canvas**, and at draw time each vertex is multiplied by
**independent X and Y factors** — an *anisotropic* stretch, not a uniform one. Text and window frames
deliberately opt out of that stretch, and a correction factor `kx` is applied by hand at the sites
that must stay square. This is the geometry half of `flows/hud-ui-load.md` (which covers XML → widget
tree → Draw); read that first for the construction spine.

**Provenance:** personally code-verified end to end — `ui_core::OnDeviceReset`,
`ClientToScreenScaled*`, `CUIStaticItem::RenderInternal`, `CUILines::Draw`, `dxFontRender::OnRender`,
`GetFontTextureName` + `CFontManager::GetFontTexName`, `CUIFrameWindow::DrawElements`,
`CUIFrameLineWnd::DrawElements`/`ReadBorder`/`DrawSlice`/`inc_pos`,
`ui_core::get_current_kx`/`screenmode`/`get_xml_name`, `CUICursor::UpdateCursorPosition`,
`ui_core::PushScissor`, `ui_core::pp_start`, plus the DX10/DX11 `stub_notransform_t.vs` transform and
the DX9 `FVF::TL` declaration.

## Canvas and scale factors

`UI_BASE_WIDTH` / `UI_BASE_HEIGHT` (`ui_defs.h`) are compile-time constants, 1024 and 768. Every
`x`/`y`/`width`/`height` in a UI XML is read raw by `CUIXmlInit::InitWindow` with no transform, and
the whole `CUIWindow` tree — parenting, hit-testing, `CUIWindow::GetAbsoluteRect` — does its
arithmetic in these units. Nothing between XML load and draw call is resolution-aware. `xrEngine`
keeps a private duplicate of the same two constants (`XR_IOConsole.cpp`) because it cannot include
`xrGame` headers.

`ui_core::OnDeviceReset` computes `m_scale_` as `Device.dwWidth / UI_BASE_WIDTH` and
`Device.dwHeight / UI_BASE_HEIGHT`, and rebuilds `m_2DFrustum` from the full device rect. `ui_core`
derives from `CDeviceResetNotifier`, which registers on `Device.seqDeviceReset`; that queue is pumped
by `CRenderDevice::Reset` and `CRenderDevice::ChangeOutputMonitor` (`Device_destroy.cpp`), so the
factors follow resolution changes, device loss and monitor swaps. `m_current_scale` is a pointer
normally aimed at `m_scale_`, temporarily redirected to `m_pp_scale_` by the main-menu PP pass.

Any aspect wider than 4x3 is horizontally over-stretched by exactly `aspect / (4/3)` — 1.33x at 16x9,
1.79x at 3440x1440. That anisotropy is the root cause of most UI-scaling bugs in this engine.

## The draw-time seam

Four inline helpers on `ui_core` are the *only* conversion points:
`ClientToScreenScaled` (base units → device px, multiply), `ClientToScreenScaledWidth` /
`ClientToScreenScaledHeight` (device px → base units — they **divide**, the names are backwards), and
`AlignPixel` (floor onto the device pixel grid). All four early-out when
`m_currentPointType == pttLIT`, because the LIT vertex path (detector screens, binoculars) is already
in device coordinates and must not be scaled twice.

The output feeds `FVF::TL` vertices, declared `D3DFVF_XYZRHW` (`FVF.h`) — pre-transformed screen
coordinates. DX9 consumes them directly; DX10/DX11 do the divide in `stub_notransform_t.vs` using
`screen_res`. So the UI backend contract (`dxUIRender::PushPoint`) is **absolute device pixels**, and
`ClientToScreenScaled` exists to satisfy it. See `flows/renderer-backend.md`.

## What stretches, and what does not

**Sprites/statics — fully stretched.** `CUIStaticItem::RenderInternal` scales both position and size,
then clips against the device-pixel screen frustum. A square icon becomes visibly squashed at 16x9
unless the call site applies `kx`. `AlignPixel` is applied to the top-left only, not the bottom-right,
so element widths can differ by a pixel at non-integer scales.

**Text — not stretched at all.** `dxFontRender::OnRender` builds glyph quads straight from the atlas
metrics in device pixels with no scale factor (`g_current_font_scale` is identity outside the PP
pass). Layout compensates by pulling font metrics *backwards* into base units — every measurement site
(`CUILines::ParseText`/`GetVisibleHeight`/`Draw`, `GetElipsisText`, `CUIStatic::AdjustWidthToText`,
`CUIFrameWindow::SetWndSize`) calls `ClientToScreenScaledWidth`/`Height` on a `CGameFont::SizeOf_` or
`CurrentHeight_` result, and `CUISubLine::Draw` converts the position forward again.

Text size therefore comes almost entirely from **which atlas rung got picked**. That selection is now
a single engine function, `GetFontTextureName(section)` (`GameFont.cpp`, `ENGINE_API`). It replaced
**three** copies of the old ladder: `CFontManager::GetFontTexName` (`HUDManager.cpp`) and
`_GetFontTexName` (`x_ray.cpp`) are now one-line forwards, and the duplicated lambda in the `CGameFont`
section constructor is gone. Two kinds of rung key are accepted in a `fonts.ltx` section:

- **arbitrary rungs** `texture_h<N>` — the digits *are* the authored device height (`parse_rung_height`
  requires the exact `texture_h` prefix and all-digit tail), so a section can declare as many atlases
  as it has;
- **the legacy four** `texture800` / `texture` / `texture1600` / `texture2160`, with hardcoded
  authored heights 600 / 768 / 1200 / 2160 and the vanilla `[lo, hi)` bands. Exactly **one** legacy
  candidate survives: `legacy_winner_idx` picks the band that claims `Device.dwHeight`, then walks
  *down* until it finds a key the section actually declares (vanilla fallback behaviour, preserved).

All candidates are then ranked by `dist = |log(authored / device_height)|` — a scale-symmetric
distance, so being 20% too small ties with being 20% too large — with ties broken toward an explicit
`texture_h` rung and then toward the larger atlas (`candidate_better`). Candidates are tried
best-first and each is **probed for an actual atlas** (`font_atlas_exists` → language-suffix
resolution via `resolve_font_texture`, then `FS.exist` on the `.ini` metadata beside the `.dds` under
`$game_textures$`); a declared-but-missing atlas is skipped with a `~[font]` log line instead of
producing invisible text. The winner is logged with its authored height and the percentage mismatch;
if nothing is usable the legacy `texture` key is forced.

Glyph *widths* now track the requested height rather than the atlas: `CGameFont::WidthScale()` =
`fCurrentHeight / fHeight` (requested ÷ atlas native) multiplies every `SizeOf_` overload, and
`dxFontRender::OnRender` applies the same ratio per string (`PS.height / owner.fHeight`) to each
glyph advance and to the alignment measurement. This matters for the **device-independent** fonts —
`hud_font_di` / `hud_font_di2` (the console) and `stat_font`, the sections constructed with
`fsDeviceIndependent` and sized by `SetHeightI`, i.e. as a fraction of `Device.dwHeight`. Their
requested height diverges from the atlas native height at every resolution, and before this fix they
advanced at atlas width and so overlapped or gapped. No `ui_font_*` widget path calls `SetHeight`, so
for ordinary UI text the atlas remains the sole size authority.

Atlas pixel heights live in the per-atlas `.ini` beside each `.dds` (`unpacked/textures/ui/` for base
content). `CFontManager::OnDeviceReset` re-runs `InitializeFonts` (registered at `REG_PRIORITY_HIGH`)
so the rung switches live on a resolution change.

**What this does and does not fix.** The ladder is now open-ended and self-validating, but it is still
a *ladder*: text is still emitted at atlas pixel size, so size only changes when a rung boundary is
crossed, and the base game's `unpacked/configs/fonts.ltx` still declares no `texture2160` for any
`ui_font_*` section — with base content only, high-resolution UI text is still undersized (~93% of
design size at 1080p, ~70% at 1440p, ~47% at 2160p). Closing that gap is a *content* job: add
`texture_h<N>` rungs plus the atlases, patched in via `mod_system_*.ltx` (`fonts.ltx` is reached
through `system.ltx`, so a `mod_fonts_*.ltx` would be silently ignored — see the DLTX glossary entry).
This repo currently ships no such patch.

**Frames — not stretched, tiled.** `CUIFrameWindow::DrawElements` converts the rect to device pixels
*first*, then lays out the nine slices at their **native texture pixel size**, tiling edges and centre
(`draw_tile_line`, `draw_tile_rect`). Consequence: window chrome stays a constant device-pixel
thickness at every resolution — hairline-thin relative to the window at 4K — and backgrounds tile more
often instead of magnifying. `CUIFrameWindow::SetWndSize` also enforces a minimum size by
round-tripping through both conversions.

**Frame-lines — three modes, opted into per texture.** `CUIFrameLineWnd::DrawElements` still defaults
to the vanilla behaviour (cap slices at native texture pixel width on screen, never scaled; the minor
axis stretched to fill), but two opt-ins change that, both keyed off data rather than resolution:

- **cap-scaled** (`SetCapScaled`, currently used by `CUIComboBox` for its line background): the draw
  pass computes `content_scale` = (element minor size ÷ texture minor size) × (device px ÷ base units
  along the major axis), i.e. scale the cap to the element by height, then to the device by
  resolution. Caps keep their authored aspect at any size and any resolution instead of being fixed
  screen pixels that only line up near 1080p.
- **crisp bands** — a `TEX_INFO` carries `border_l/t/r/b` texel insets, read from the `border_*`
  attributes of a `textures_descr` `<texture>` entry (`CUITextureMaster`, `CUIFrameLineWnd::ReadBorder`,
  which validates them against the slice size and logs + ignores nonsense). Any non-zero border on any
  of the three slices sets `m_has_border`, which turns on crisp mode (and cap scaling with it). In
  crisp mode each listed outer row/column is drawn as its **own flat band** of `band_px` device pixels
  (the scale rounded to a whole pixel, min 1) while the content between them keeps a normal stretched
  UV. Placement uses a cut table (`BuildAxis` / `SpanAt` / `CutAt` / `LeadCut` / `TrailCut`) that gives
  every texel `length/span` pixels ±1, so bands and content keep their art proportions and no seam
  gaps appear. `CellCount`/`CapExtent` widen the vertex reservation accordingly, and `DrawSlice`
  falls back to a single `draw_rect` whenever an axis cannot be built.

The repo ships this as vanilla `textures_descr` copies with `border_*` added on the slices that
visibly suffered from stretching: `ui_listline2`/`ui_listline3` (list rows), `ui_scroll_back`/
`ui_scroll_box` (scrollbars) in `ui_common.xml`, `ui_inGame2_opt_slider_bar` (option tracks) in
`ui_actor_main_menu_options.xml`, and the edit box / spin box / combobox line backgrounds in
`ui_actor_multiplayer_menu_screen.xml`. Nothing else changes behaviour, so third-party descriptors
render exactly as before.

Two smaller fixes ride along: `CUIInteractiveBackground::InitState` takes the `vertical` flag and
applies it through the `ib_set_vertical` overload pair **before** `InitTexture` (a frameline asserts
its cap/back slice dimensions per axis, so the orientation has to be set first), and `CUIEditKeyBind`
was rebuilt on a frameline background to stop stretching its art.

## kx — the anisotropy correction

`ui_core::get_current_kx` returns `(h/w) / (UI_BASE_HEIGHT/UI_BASE_WIDTH)`, which is algebraically
`scale.y / scale.x` — 1.0 at 4x3, 0.75 at 16x9, 0.5625 at 21x9. Multiplying a width in base units by
`kx` and letting `scale.x` act on it yields `width * scale.y`, so a square stays square.

It is **opt-in per call site**; there is no automatic mechanism. Roughly two dozen consumers, notably:
the mouse cursor sprite (`CUICursor::InitInternal`), the minimap and its compass ring
(`CUIZoneMap::Init`), map spots (`map_spot.cpp`), world-map pan/zoom/rotation (`UIMap.cpp` — with
`CUIMapWnd` un-applying it for mouse hit-testing), inventory cell icons and addon overlays
(`UICellItem.cpp`, `UICellCustomItems.cpp`), and rotated sprites (`S2DVert::rotate_pt` takes `kx` as a
parameter, fed from the rotating `CUIStaticItem::RenderInternal` overload, so rotation happens in a
corrected space before the stretch). Anything not on the list is simply left stretched.

## Aspect-class XML variants

The content-side half. `ui_core::screenmode` buckets the display into three classes: 0 for aspect at
or below ~1.3433 (4x3, 5x4), 1 up to 1.8 (16x10, 16x9), 2 above 1.8 (21x9 and wider).
`ui_core::get_xml_name` rewrites the requested filename to a `_16` or `_21` suffix with a documented
fallback chain (`_21` → `_16` → base). `ui_core::is_widescreen` is the boolean form of the same
threshold. `CMainMenu::OnFrame` detects an aspect-class change at runtime and hot-reloads the menu.

Coverage in the base game is partial — of the UI XMLs under `unpacked/configs/ui/`, well under half
have a `_16` variant and exactly one (`scopes_21.xml`) has a `_21` variant. On ultrawide almost
everything falls back to the 16x9 layout, and files with no variant at all fall back to 4x3 geometry
and just get stretched. Some code hardcodes small per-aspect pixel deltas instead
(`UIInvUpgrade.cpp`, `UIOutfitInfo.cpp`, `UIInventoryUpgradeWnd_add.cpp`).

A related but distinct use of the same predicate: `is_widescreen` also gates the *3D* weapon HUD,
selecting the `_16x9` config keys and forcing `hud_item_measures::load` (`player_hud.cpp`). That is
config-driven art correction, not UI scaling — see the `hud_offset` glossary entry.

## Input, clipping, post-process

**Cursor** lives in the same base-unit space, so hit-testing needs no conversion:
`CUICursor::UpdateCursorPosition` maps the OS pointer linearly by `UI_BASE_WIDTH / Device.clientWidth`
and clamps to the canvas; `CUICursor::SetUICursorPosition` inverts it. Note this uses
`clientWidth`/`clientHeight` (the Win32 client rect) rather than `dwWidth`/`dwHeight` (the backbuffer)
— they agree in fullscreen but can diverge windowed. ImGui uses the same mapping
(`imgui_base_input.cpp`). Because the mapping carries no `kx`, raw cursor *deltas* are anisotropic in
base units.

**Clipping** uses two independent stacks. `ui_core::PushScissor`/`PopScissor` keep rects in **base
units** (asserting they lie inside the canvas, intersecting with the parent on push) and convert only
when handing to `dxUIRender::SetScissor`. The clip-frustum stack (`ui_core::PushClipFrustum`,
`C2DFrustum::AddEdgePlane`) is in **device pixels**; its base entry is the full-screen rect built in
`OnDeviceReset`. The scrollable PDA tab strip is the live consumer — `CUITabControl::DrawTabsClipped`
converts its edge points before building the planes (`features/pda-tab-scroll.md`).

**Post-process** — `ui_core::pp_start`/`pp_stop` are used only by the main menu
(`CMainMenu::OnRenderPPUI_main`/`OnRenderPPUI_PP`). They swap in a scale computed against the render
target instead of the backbuffer and set `g_current_font_scale` to the RT/backbuffer ratio, which is
the *only* path where text does scale. Everywhere else that global stays identity.

## Lua mirror

Script UI works in the same base-unit space and reimplements the same helpers in
`unpacked/scripts/utils_xml.script`: `is_widescreen` (same threshold), `screen_ratio` (literally
`kx`), `correct_ratio` (the Lua equivalent of `ClientToScreenScaledWidth`/`Height` — converts an
element sized in device pixels down into base units), plus `sync_cursor` / `align_to_center` /
`adjust_pos_to_element` which hardcode the canvas bounds. Consumers are catalogued in
`flows/utils-libraries.md` and used throughout `flows/actor-menu-inventory-ui.md`.

## Failure & edge cases

- **The stretch is anisotropic and correction is opt-in.** New code that draws a circle, square icon
  or rotated sprite and forgets `kx` will be visibly elliptical at 16x9.
- **Text does not scale with the UI.** Discrete atlas rungs that do not keep pace with the layout
  scale. The ladder is now extensible (`texture_h<N>`) and skips missing atlases, but with base-game
  content only there is still no rung above `texture1600` for any `ui_font_*` section, so above 1440p
  they fall back to the 1600 atlas. Fixing it needs atlases + a `mod_system_*.ltx` patch, not code.
- **Frames do not scale.** `CUIFrameWindow` chrome is still constant device-pixel thickness;
  backgrounds tile rather than magnify. Frame-*lines* can now opt out per texture (cap-scaled /
  crisp bands) but only where a descriptor declares it.
- **No user-facing UI scale and no DPI awareness.** `UI_BASE_WIDTH`/`HEIGHT` are `#define`s; there is
  no console variable or option. The only `ui_scale` in the tree is an unrelated per-attachment script
  property (`script_attachment_manager.cpp`). Windows display scaling is invisible to the UI.
- **Ultrawide is effectively unsupported** — one `_21` XML exists in the whole game.
- **`CUIXmlInit::ApplyAlignX`/`ApplyAlignY` are no-ops** (they return the coordinate unchanged), so the
  `align="r"/"b"/"c"` XML attribute does nothing. Only `alignment="c"` has an effect, via `waCenter`
  in `CUISimpleWindow::GetWndRect`, which reinterprets `m_wndPos` as the centre instead of the
  top-left.
- **Vertical text alignment used to ignore the canvas.** `CUILines::Draw` added `GetVIndentByAlign`
  in *device pixels* after the conversion; it is now added in base units before it, so a
  vertically-centred label stays centred at every resolution. Any layout hand-tuned against the old
  behaviour shifts.
- **`ui_core` is constructed before `m_current_scale` is assigned** — safe today only because
  `OnDeviceReset` does not dereference it.

## Deeper

- Canvas + factors: `src/xrGame/ui_defs.h` (`UI_BASE_WIDTH`, `UI_BASE_HEIGHT`, `C2DFrustum`),
  `src/xrGame/ui_base.h` / `ui_base.cpp` (`ui_core::OnDeviceReset`, `ClientToScreenScaled*`,
  `AlignPixel`, `get_current_kx`, `is_widescreen`, `screenmode`, `get_xml_name`, `pp_start`/`pp_stop`)
- Device reset chain: `src/xrEngine/Device_destroy.cpp` (`CRenderDevice::Reset`,
  `ChangeOutputMonitor` → `seqDeviceReset` / `seqResolutionChanged`)
- Layout space: `src/xrGame/ui/UIXmlInit.cpp` (`InitWindow`, `GetFRect`, `InitAlignment`,
  `ApplyAlignX`/`ApplyAlignY`), `src/xrGame/ui/UIWindow.cpp` (`GetAbsoluteRect`),
  `src/xrGame/ui/uiabstract.h` (`CUISimpleWindow::GetWndRect`, `waCenter`)
- Sprite draw: `src/xrGame/UIStaticItem.cpp` (`CUIStaticItem::RenderInternal` both overloads,
  `ComputeRenderUV`)
- Text: `src/xrEngine/GameFont.cpp` (`GetFontTextureName`, `parse_rung_height`, `legacy_winner_idx`,
  `candidate_better`, `font_atlas_exists`, `resolve_font_texture`, `CGameFont` section ctor, `SizeOf_`,
  `WidthScale`, `CurrentHeight_`, `SetHeight`/`SetHeightI`, `g_current_font_scale`),
  `src/xrGame/HUDManager.cpp` (`CFontManager::GetFontTexName`, `InitializeFonts`, `OnDeviceReset`),
  `src/Layers/xrRender/dxFontRender.cpp` (`dxFontRender::OnRender`), `src/xrGame/ui/UILines.cpp`
  (`ParseText`, `GetVisibleHeight`, `Draw`, `get_str_width`, `GetElipsisText`),
  `src/xrGame/ui/UISubLine.cpp` (`CUISubLine::Draw`), `unpacked/configs/fonts.ltx` (rung keys),
  `unpacked/textures/ui/` (per-atlas `.ini` `height`)
- Frames: `src/xrGame/ui/UIFrameWindow.cpp` (`DrawElements`, `get_points`, `draw_tile_line`,
  `draw_tile_rect`, `SetWndSize`)
- Frame-lines: `src/xrGame/ui/UIFrameLineWnd.h` (`SBorder`, `SSpan`, `SAxis`, `SDraw`),
  `src/xrGame/ui/UIFrameLineWnd.cpp` (`DrawElements`, `ReadBorder`, `CapExtent`, `CellCount`,
  `BuildAxis`, `SpanAt`, `CutAt`, `DrawSlice`, `inc_pos`, `draw_rect`, `SetCapScaled`),
  `src/xrGame/ui/UITextureMaster.h` (`TEX_INFO::border_l/t/r/b`),
  `gamedata/configs/ui/textures_descr/` (the shipped `border_*` opt-ins)
- kx consumers: `src/xrGame/UICursor.cpp` (`InitInternal`), `src/xrGame/UIZoneMap.cpp` (`Init`),
  `src/xrGame/ui/UIMap.cpp` (`CUIMiniMap::Draw`, `GetPointerTo`), `src/xrGame/map_spot.cpp`,
  `src/xrGame/ui/UICellItem.cpp`, `src/xrGame/ui/UICellCustomItems.cpp`
- Backend contract: `src/Layers/xrRender/dxUIRender.cpp` (`PushPoint`, `StartPrimitive`,
  `SetScissor`), `src/Layers/xrRender/FVF.h` (`FVF::TL`, `F_TL`),
  `unpacked/shaders/r3/stub_notransform_t.vs`
- Clip stack in use: `src/xrGame/ui/UITabControl_scroll.cpp` (`CUITabControl::DrawTabsClipped`) —
  detail in `features/pda-tab-scroll.md`
- Lua mirror: `unpacked/scripts/utils_xml.script` (`correct_ratio`, `screen_ratio`, `is_widescreen`,
  `sync_cursor`, `align_to_center`)
- Widget construction spine: `flows/hud-ui-load.md`

## Maintenance

- Re-grep `UI_BASE_WIDTH` if the canvas constant ever becomes configurable — every claim here assumes
  it is a compile-time `#define` in two places (`ui_defs.h` and the private copy in `XR_IOConsole.cpp`).
- Re-grep `ClientToScreenScaled` if a new conversion helper appears or a widget starts converting
  somewhere other than draw time; the "conversion happens only at primitive emission" invariant is what
  makes the whole tree resolution-agnostic.
- Re-grep `get_current_kx` when auditing a new square/rotated element — a missing call is the standard
  bug, and the consumer list above is the reference set.
- Re-read `GetFontTextureName` and `unpacked/configs/fonts.ltx` if font sizing changes: check the rung
  key grammar (`texture_h<N>` prefix), whether the legacy band table or its authored heights moved,
  and whether any shipped `mod_system_*.ltx` now adds rungs for `ui_font_*` sections (that alone would
  fix the high-resolution text-size complaint). `CFontManager::GetFontTexName` and `_GetFontTexName`
  must stay forwards — if the selection logic is ever duplicated again, say so here.
- Re-count `_16` / `_21` files under `unpacked/configs/ui/` if ultrawide support is revisited.
- Re-check `g_current_font_scale` consumers if anything besides the main menu starts using
  `pp_start`/`pp_stop`.
