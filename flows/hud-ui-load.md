---
domain: flows/hud-ui-load
tags: [xrgame, ui, hud, xml, cuiwindow, dxml, shaders, flow]
sources: [src/xrXMLParser/xrXMLParser.cpp, src/xrGame/ScriptXMLInit.cpp, src/xrGame/ui/UIXmlInit.cpp, src/xrGame/ui/UIWindow.cpp, src/xrGame/ui/UIWindow.h, src/xrGame/ui/UIWindow_script.cpp, src/xrGame/ui/uiscriptwnd_script.h, src/xrGame/ui/UI3tButton.cpp, src/xrGame/ui/UIStatic.cpp, src/xrGame/UIStaticItem.cpp, src/xrGame/UIGameCustom.cpp, gamedata/scripts/lua_help_ex.script, gamedata/shaders/r3/hud_default.ps, gamedata/shaders/r3/hud_font.ps]
related: [flows/dxml-xml-interception.md, flows/engine-boot-frame.md, flows/lua-callback-lifecycle.md, flows/renderer-factory-bridges.md, flows/pda-map-news.md, flows/ui-scaling.md, features/pda-tab-scroll.md, modlayer/dxml.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: in-game HUD / UI load

How a UI window (HUD, menu, dialog) is built from an XML layout and then drawn each frame. The path
is: load XML → **DXML Lua interception** (every XML file passes through `_G.COnXmlRead`) → re-parse →
`CUIXmlInit` recursively instantiates `CUIWindow`-derived widgets and `AttachChild`s them into a tree
→ per frame the tree is recursively `Update`d and `Draw`n. The DXML mutation step itself is
`flows/dxml-xml-interception.md`; this flow is the widget-construction + draw spine around it.

**Provenance:** spine personally code-verified — `CXml::Load` (calls `XMLLuaCallback`), `XMLLuaCallback`
(`ScriptXMLInit.cpp`), `CUIXmlInit::InitWindow`/`InitAutoStaticGroup`/`InitStatic`,
`CUIWindow::AttachChild`/`Update`/`Draw`/`CollectGarbage`. The top-level game-UI owner
(`CUIGameCustom::Load`/`OnFrame`/`Render`) and the frame-loop hook (`CHUDManager` on `seqRender`/
`seqFrame`) are confirmed homes but *(agent-traced — not personally re-read)*.

## Construction sequence (verified)

1. **`CXml::Load`** (`xrXMLParser.cpp`) — opens the XML file, pre-processes, parses with TinyXML,
   then calls **`XMLLuaCallback(*this, buffer)`** — i.e. *every* XML load is run through Lua before the
   engine uses the DOM.
2. **`XMLLuaCallback`** (`ScriptXMLInit.cpp`) — strips the BOM, looks up the Lua functor
   `_G.COnXmlRead`, calls it with `(file_name, xml_string)`, and **re-parses the returned string** via
   `m_xml.LoadFromString(res)`. This is the DXML seam (`flows/dxml-xml-interception.md`): modxml
   handlers can rewrite the layout in memory before any widget is built.
3. **`CUIXmlInit::InitWindow`** — for a node path: `NavigateToNode` (asserts if missing), read
   `x`/`y`/`width`/`height` (+ `InitAlignment`), `SetWndPos`/`SetWndSize`, optional `window_name`, then
   call `InitAutoStaticGroup`. Most widget initializers (`InitFrameWindow`, `InitStatic`, …) call
   `InitWindow` for their common geometry.
4. **`CUIXmlInit::InitAutoStaticGroup`** — the recursive tree builder: iterate the node's XML children,
   and for each `auto_static`/`auto_frameline`/`auto_framewindow`/`auto_text` create the matching C++
   widget (`CUIStatic`/`CUIFrameLineWnd`/`CUIFrameWindow`/…), run its `Init*`, `SetAutoDelete(true)`,
   and **`pParentWnd->AttachChild(widget)`**.
5. **`CUIXmlInit::InitStatic`** — element initializer example: `InitWindow` + text/texture/color-anim
   from the node's attributes.
6. **`CUIWindow::AttachChild`** — `SetParent(this)` and push into `m_ChildWndList` under the `csUi`
   critical section (dup-attach is logged, not fatal). The parent/child list *is* the widget tree.

Script-driven UI (Lua-built windows via `CScriptXmlInit`/`CUIScriptWnd`) uses the same `CUIXmlInit`
initializers from Lua. *(agent-traced.)*

## Per-frame sequence (verified spine; owner agent-traced)

The game UI owner (`CUIGameCustom`, driven by `CHUDManager` on the render/frame registrators of
`flows/engine-boot-frame.md`) calls into the tree each frame:

- **`CUIWindow::Update`** — focus enter/leave detection against the cursor, then recurse `Update` over
  shown children (under `csUi`), then `CollectGarbage` (deferred-delete children marked via
  `SetAutoDelete`/`m_ChildWndToDelete`). The dialog stack (`CDialogHolder`) gives its
  `TopInputReceiver` update/input priority.
- **`CUIWindow::Draw`** — recurse `Draw` over children (under `csUi`), skipping hidden or
  custom-drawn children; leaf widgets (e.g. `CUIStatic`) draw their texture/text.

## Texture draw & per-element shader (verified)

A leaf `CUIStatic::Draw` runs `DrawTexture` → the element's `CUIStaticItem::Render`: it binds the
item's `ui_shader` and pushes a four-vertex quad, every vertex carrying the item's `dwColor`. The
texture is sampled and **multiplied by `dwColor`** — the per-element tint+alpha, set from XML by
`CUIXmlInit::GetColor` (`r`/`g`/`b`/`a` attributes, each defaulting to 255) via `SetTextureColor`.

The shader is chosen per element. `CUIXmlInit::InitTexture` reads an optional `shader` attribute on
the `<texture>` node: absent → `CUIStatic::InitTexture` → the default `hud\default` blender; present →
`InitTextureEx(tex, shader)` → that blender (`CUIStaticItem::CreateShader` via `CUITextureMaster`).
Two UI pixel shaders matter (`gamedata/shaders/r3/`):

- **`hud_default.ps`** — `colour = texture.Sample(...) * I.Color`: the normal multiply tint. Tinting a
  multi-colour icon with one colour can't yield a flat colour (the icon's own colours show through).
- **`hud_font.ps`** — `rgb = I.Color; a = texAlpha * I.Color.a`: discards the texture's colours, keeps
  only its alpha, filled with the vertex colour. Rendering any icon through `hud\font` yields a
  **solid-colour silhouette of that icon's shape**, derived live from its alpha — the basis for
  drawing a colour outline/halo from an arbitrary (mod-supplied) icon without baking a new texture.

`CUITextureMaster::InitTexture` resolves a texture name as either an atlas region (named in a
`textures_descr` XML — sets the sub-rect) or a direct file path. For an atlas region the UVs stay
within the region rect, so a scaled/offset quad magnifies that cell rather than bleeding into
neighbours (a `clamp` sampler on `hud\default`; `hud\font` relies on the rect + transparent cell
edges). Map spots use exactly this machinery — `flows/pda-map-news.md` for `location_level` z-order
and stacking an outline spot under an icon.

## Repo additions to the widget/Lua contract (verified)

Ownership, input and hit-testing gained a few seams that mod UIs depend on. All are catalogued from the
Lua side in `gamedata/scripts/lua_help_ex.script`.

- **`AttachChildKeepOwner`** (`UIWindow_script.cpp`) — the same `CUIWindow::AttachChild`, exported
  *without* luabind's `adopt<2>` policy. `AttachChild` transfers ownership to C++, which is wrong when
  attaching a **Lua-owned** window into an engine window (the engine would free a script object). Use
  `AttachChild` for engine-owned children, `AttachChildKeepOwner` when Lua keeps the reference.
- **`OnMouse` script hook** — `CWrapperBase` (`uiscriptwnd_script.h`) now wraps
  `OnMouseAction` the way it already wrapped `OnKeyboardAction`, so a `CUIScriptWnd` subclass can
  implement `OnMouse(x, y, action)` and call the base through the exported static. This is what makes
  wheel/scroll handling possible in script windows (`CUIScrollView`/`CUIScrollBar` hooks).
- **Convex hit clipping** — `CUIWindow::SetHitClip(pts, count)` / `HitClipPass(abs_pos)`
  (`UIWindow.h`): an optional convex polygon in absolute coords, AND-gated on top of the normal rect
  test in both `Update` (hover) and the parent's `OnMouseAction` child routing, so hover and click can
  never disagree. Degenerate input clears it. Built for the slanted PDA tabs
  (`features/pda-tab-scroll.md`), but it is a general widget facility.
- **Custom-draw backgrounds** — `CUI3tButton::InitButton` marks its static/frameline background
  `SetCustomDraw(true)` because `DrawTexture` paints it and the child list is drawn afterwards;
  without that a background paints twice, over anything layered on the button face.
- **List/properties-box exports** — `CUIPropertiesBox::AddHeader` / `AddSubmenu` / `AddItem(text, id)`
  and `CUIListBoxItem::Get/SetTAG` (`flows/pda-map-news.md` documents the menu shape they enable).
- **Frame-line rendering gained two opt-ins** (cap scaling, crisp `border_*` bands) plus an explicit
  `vertical` argument on `CUIInteractiveBackground::InitState`, since a frameline must know its
  orientation before `InitTexture` asserts on slice dimensions — see `flows/ui-scaling.md`.

## Isolation ledger (the seams)

- **DXML interception (every XML):** `CXml::Load` → `XMLLuaCallback` → `_G.COnXmlRead` →
  `LoadFromString`. This is *the* mod hook for UI layouts (`flows/dxml-xml-interception.md`,
  `modlayer/dxml.md`); it runs once per file load, synchronous, before widget construction.
- **Lua UI binding:** `CScriptXmlInit` exposes the `CUIXmlInit` initializers to Lua; `CUIScriptWnd`
  is the Lua-scriptable window base; button callbacks are Lua functors. Mod UIs build widgets this way.
  Ownership choice is `AttachChild` (adopt) vs `AttachChildKeepOwner` (no adopt); overridable virtuals
  reaching Lua now include `OnMouse` alongside `OnKeyboard`/`Update`.
- **Frame-loop integration:** the game UI is driven from `CHUDManager` on the engine's `seqFrame`/
  `seqRender` registrators (`flows/engine-boot-frame.md`) — `Update` in the frame phase, `Draw` in the
  render phase, both on the main thread.
- **Threading guards:** `csUi` critical section guards `m_ChildWndList` for attach/detach vs iterate;
  deferred delete (`CollectGarbage`) avoids freeing a child mid-iteration.
- **Config/XML read:** layout files under the UI config path (e.g. `maingame.xml`,
  `ui_custom_msgs.xml`); element attributes (pos/size/texture/text/anim) read by the `Init*` methods.

## Threading

UI construction and the per-frame Update/Draw run on the main thread (UI tree access guarded by
`csUi`). XML load + the DXML Lua callback are synchronous on the loading thread; Lua is single-threaded.

## Failure & edge cases

- **Missing XML node is fatal in construction:** `InitWindow`'s `R_ASSERT4(NavigateToNode...)` crashes
  (debug) if a layout node a widget expects is absent — a DXML edit that removes a required node breaks
  construction.
- **DXML graceful path:** if no `_G.COnXmlRead` functor exists, `XMLLuaCallback` leaves the parsed DOM
  as-is (no transform) — UI still loads.
- **Dup AttachChild is non-fatal:** logged and skipped (a guard added over the original crash).
- **Deferred delete:** widgets removed during Update are deleted in `CollectGarbage`, not inline —
  prevents use-after-free during tree iteration.

## Deeper

- XML load + DXML seam: `src/xrXMLParser/xrXMLParser.cpp` (`CXml::Load`, `LoadFromString`),
  `src/xrGame/ScriptXMLInit.cpp` (`XMLLuaCallback`, `CScriptXmlInit::ParseFile`)
- Widget construction: `src/xrGame/ui/UIXmlInit.cpp` (`InitWindow`, `InitAutoStaticGroup`,
  `InitStatic`, and the per-element `Init*` family)
- Tree + frame: `src/xrGame/ui/UIWindow.cpp` (`AttachChild`, `Update`, `Draw`, `CollectGarbage`)
- Leaf texture draw + shader: `src/xrGame/ui/UIStatic.cpp` (`CUIStatic::Draw`/`DrawTexture`,
  `InitTextureEx`), `src/xrGame/UIStaticItem.cpp` (`CUIStaticItem::Render`/`RenderInternal`,
  `CreateShader`, `dwColor`), `src/xrGame/ui/UIXmlInit.cpp` (`InitTexture`, `GetColor`)
- UI pixel shaders: `gamedata/shaders/r3/hud_default.ps` (multiply tint), `hud_font.ps`
  (alpha-as-mask silhouette)
- Map-spot use (location_level z-order, shader-per-spot): `flows/pda-map-news.md`
- Game UI owner: `src/xrGame/UIGameCustom.cpp` (`Load`, `OnFrame`, `Render`), `HUDManager.cpp`
- DXML detail: `flows/dxml-xml-interception.md`, `modlayer/dxml.md`

## Maintenance

- Re-grep `CXml::Load` → `XMLLuaCallback` if the XML→Lua interception point moves; this is the contract
  that makes DXML work for UI.
- Re-grep `CUIXmlInit::InitAutoStaticGroup` if new auto-element node types are added.
- Re-grep `CUIStaticItem::Render` / `CUIXmlInit::InitTexture` and the `hud_default.ps`/`hud_font.ps`
  shaders if the UI draw primitive, the per-element `shader=` selection, or the tint maths change.
- Promote the *(agent-traced)* `CUIGameCustom`/`CHUDManager` per-frame wiring to verified if a task
  needs the exact HUD draw order or the seqFrame/seqRender registration.
