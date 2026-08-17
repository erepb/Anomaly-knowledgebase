---
domain: features/pda-tab-scroll
tags: [ui, pda, tabs, scripting, cpp, lua]
sources: [src/xrGame/ui/UITabControl.h, src/xrGame/ui/UITabControl.cpp, src/xrGame/ui/UITabControl_scroll.cpp, src/xrGame/ui/UITabButton.h, src/xrGame/ui/UITabButton.cpp, src/xrGame/ui/UITabScrollArrows.h, src/xrGame/ui/UITabScrollArrows.cpp, src/xrGame/ui/UIPdaWnd.cpp, src/xrGame/ui/UIXmlInit.cpp, src/xrGame/ui/UIWindow.h, src/xrGame/ui_defs.h, src/xrGame/ui_base.h, gamedata/scripts/pda_dynamic_tabs.script, gamedata/scripts/lua_help_ex.script]
related: [INDEX.md, GLOSSARY.md, flows/pda-tabs.md, flows/ui-scaling.md, flows/hud-ui-load.md]
updated: 2026-08-17
verified: code
---

# PDA dynamic tabs + tab-strip scrolling + tab icons

Merged into the mainline branch (PRs #608/#609 dynamic tabs, #630 hi-res fixes, #633 PDA reskin
follow-ups). Three related capabilities on `CUITabControl`, all built so a mod never has to override
the single-owner `pda_16.xml`:

- **dynamic tabs** — mods register a tab from Lua; the engine clones one of the strip's own tabs;
- **overflow handling** — the strip first squeezes authored spacing, then scrolls with two arrows;
- **tab icons** — per-tab art fitted into a slot the strip's XML declares.

Nothing here ships content: the repo carries **no `pda_16.xml`**, so every tab renders with whatever
art the installed PDA layout (vanilla or a reskin) declares. The earlier design that healed vanilla
static strips into framelines at runtime, and the authored `endcap_width` / `frame_cap_scaled`
attributes, were both removed — do not look for them.

## Registry (Lua side)

`gamedata/scripts/pda_dynamic_tabs.script` is the whole public API: `register(caption_key, ui_getter,
after)` → returns a registry-assigned section id (`ept_dynamic_<n>`, monotonic, never reused),
`unregister(id)`, and `build_extra_tabs(tab_control)` which the engine calls back. The script also
wraps `pda.set_active_subdialog` so a registered id routes to that entry's `ui_getter()` and
everything else falls through to the base implementation (`flows/pda-tabs.md`).

`register`/`unregister` may be called at any time: `apply()` no-ops before `level.present()`, and
otherwise calls `ActorMenu.get_pda_menu():RebuildTabs()` so the live strip re-materializes
immediately. Feature detection is `CUITabControl.AddTab ~= nil`.

Tabs sharing an `after` anchor come out in **reverse** registration order, because the engine inserts
each one directly after the anchor.

**Icon naming is the caption key, not the tab id.** `build_extra_tabs` calls `SetTabIcon(id,
caption_key)`, so art is looked up under the string-table id the registering mod chose (`_e`, plus
optional `_d`/`_h`/`_t`, or a single unsuffixed texture). That id is stable and known to both sides,
so a PDA reskin can supply icons for tabs it has never heard of and a mod can ship a default for its
own — neither has to coordinate. Missing art is not an error.

## Engine entry points

- `CUIPdaWnd::RebuildExtraTabs` (Lua: `pda_menu:RebuildTabs()`) — remembers the active id, calls
  `RemoveDynamicTabs`, invokes `pda_dynamic_tabs.build_extra_tabs`, then `RecalcScroll`. Restore
  policy: a surviving **XML** tab keeps its pushed state and only gets `EnsureVisible` (no subdialog
  churn); a dynamic active tab (its button was destroyed and rebuilt) or one that disappeared (falls
  back to the strip's first tab) goes through `ResetTab` + `SetActiveTab`, which routes
  `TAB_CHANGED` → `SetActiveSubdialog` so the content window follows. `CUIPdaWnd::Init` calls it too,
  right after `CUIXmlInit::InitTabControl`.
- `CUITabControl::AddTab(id, caption, after_id)` — id-aware insertion after `after_id` (empty or
  unknown → appended, with a log line). Geometry comes from the strip itself: size = the first
  non-parked tab's size, position = previous tab's x + its `Pitch()`, and the button is
  `first_tab->Clone(pos, size)`. Fails with a log line on a control with no tabs — there is no
  hardcoded fallback size, atlas name or font. Marks `m_dynamic = true`.
- `CUITabControl::RemoveDynamicTabs` — compacts the dynamic entries out of `m_TabsArr` (detaching the
  children); the layout re-derives from what is left.
- `CUITabControl::SetTabIcon(id, art)` / `SetIconLayout(pos, box, anchor)` — icons, below.
- `CUITabControl::SetScrollArrow(side, arrow)` — hand over a skin-authored arrow (`Adopt`).
- Lua exports: `AddTab`, `SetTabIcon`, `RecalcScroll`, plus the pre-existing `RemoveAll`/`AddItem`.

`CUITabButton::Clone` reproduces the reference tab's art (re-`InitTexture` from `ArtBase()`, skipped
when empty → an untextured clone, exactly like a texture-less vanilla tab), seam overlap, per-state
text colours and the caption's font/alignment/offset. The caller fills id and caption only. It does
**not** copy `m_frameline_mode`, so the clone's `InitButton` always builds the default `CUI_IB_Static`
background — on a frameline-backed strip a dynamic tab is static-backed (it still draws the same
per-state art, just stretched rather than sliced).

Background widgets are created with `SetCustomDraw(true)` in `CUI3tButton::InitButton` for exactly
one reason, spelled out in the comment there: `DrawTexture` paints the background itself, and the
child list is drawn *after* it, so a non-custom-draw background would paint a second time on top of
anything layered on the button face (the icon).

## Geometry: authored by the layout, owned per tab

`CUITabControl` stores no tab shape. Each `CUITabButton` owns:

- `Overlap()` / `SetOverlap()` — **signed** seam overlap: positive = the next tab tucks under this
  one, negative = plain spacing between square tabs.
- `Pitch()` = `width - overlap` — the advance to the next tab.
- `Seam()` = the overlap when it is strictly inside `(0, width)`, else 0. This is the slant width
  used for clip planes and hit polygons.
- `CapWidthTexels(art)` / `CapWidthUI()` — **the one deliberate hack**, and it is commented as such
  in the header: tab art has slanted ends but nothing declares how wide the slant is (the atlas rect
  carries no cap boundary, and `pitch = width - cap + margin` is one equation in two unknowns), so
  the cap is taken to be as wide as the art is tall. It is fitted to the vanilla art. Making it
  exact would require the caps to be declared, i.e. tabs rebuilt as `CUIFrameLineWnd`, whose
  begin/end slice rects state the cap widths outright.
- `ArtBase()` — the authored base texture name, captured in the `CUITabButton::InitTexture`
  override rather than read back from the render widgets. The arrows synthesize `_e/_d/_h/_t` from
  it; an empty base (art declared per-state only) degrades to a glyph-only arrow and an untextured
  clone, never a fabricated texture name.
- `HitPoly(out, max)` / static `SlantPoly(out, max, pos, size, slant)` — the drawn outline in
  control-local coords, emitted BL, BR, TR, TL (a parallelogram for slanted art, the bounding rect
  when `Seam()` is 0). The arrows reuse the same formula.

`CUITabControl::RebuildTabOverlaps` derives every overlap from the **live XML rest positions** (next
tab's x vs this tab's right edge) and records the strip origin in `m_origin_x`; the last strip tab
inherits its predecessor's overlap. Dynamic and parked tabs are skipped. It re-runs from
`InsertItem` on every non-dynamic insert, so the authored rhythm is re-read as the XML strip is
built and a dynamic insert can never redefine it. Layout X is never stored:
`ApplyScroll` re-sums pitches from the viewport's left wall, so the live widget x is the scroll
offset's only home (`CurrentScroll` reads it back) and a script-triggered `RecalcScroll` cannot
drift the strip.

## Overflow: squeeze first, then scroll

`RecalcScroll` measures two spans from the origin: `m_content_w` over **all** non-parked tabs (what
must fit) and `m_strip_w` over the **static** (XML) tabs alone (the frame to stay inside).
`GetWidth()` is deliberately not used as the frame — the legacy PDA layout ships it wrong; anchoring
on both authored ends is what keeps the last tab's right edge in place while the interior
redistributes.

**Slack** is authored spacing beyond `TAB_MIN_GAP` (3 UI units) on negatively-overlapped tabs
(`TabSlack`). If the overflow fits in the total slack, `RecalcScroll` computes a `squeeze` factor in
[0,1] and `ApplyScroll` lays the strip out with `LaidPitch`/`LaidOverlap` — the tabs simply sit
closer together and no arrows appear. Only what slack cannot absorb turns into scrolling
(`CanScroll()` = `m_content_w > m_strip_w + 0.5`).

Scrolling state: `m_view_left` / `m_view_right` (the band between the arrows, the left wall pulled
back by the reference tab's laid overlap so its tucked edge lands right), `MaxScroll`, `ClampScroll`,
`ScrollStep` = the reference tab's laid pitch. An arrow click scrolls one step — handled in
`CUITabControl::SendMessage` **before** the `GetAcceleratorsMode` early-return, so the arrows work on a
strip with accelerators off; a wheel notch is
`TAB_SCROLL_WHEEL_STEP` = 1/3 of it (handled in `CUITabControl::OnMouseAction` before the base class
sees it, and only while scrollable). `ApplyScroll` also `SetVisible`s each tab by whether it
overlaps the frame at all, so fully-scrolled-away tabs stop drawing. `EnsureVisible(id)` scrolls the
minimum distance to bring a tab fully inside the band; it no-ops on parked tabs.

## Drawing: two passes, and no children

`CUITabControl::Draw` falls straight through to the base implementation when the strip is not
scrollable. Otherwise `DrawTabsClipped`:

1. builds a `C2DFrustum` from the screen frustum plus two slanted `AddEdgePlane` half-planes along
   the viewport walls (skipped when `Seam()` is 0 → square strips need no slant planes);
2. pushes a **base-unit scissor** over the background band and draws each shown tab's
   `DrawTexture()`, wrapping the tabs that cross a wall in `UI().PushClipFrustum(&frustum)` so their
   slanted edge is cut instead of squared off;
3. pushes a second scissor inset by the tuck (the laid overlap) and draws every tab's `DrawText()`
   — fonts cannot be poly-clipped, so text gets a plain rect;
4. draws the arrows last.

**This path draws no child windows** (it calls `DrawTexture`/`DrawText`, not `Draw`). Anything a tab
must show therefore has to be emitted from inside `CUITabButton::DrawTexture` — which is exactly why
the icon is a `SetCustomDraw(true)` child drawn by hand there. A new decoration attached as a normal
child would silently vanish the moment the strip starts scrolling.

The clip-frustum stack, the software clip in `CUIFrameLineWnd`'s `draw_rect`, and the
`ClipBudget`-widened vertex reservations exist for this path; see `flows/ui-scaling.md`.

## Hover and click ownership

`CUIWindow::SetHitClip(pts, count)` installs an optional convex hit polygon (absolute coords, any
vertex count, degenerate input clears it); `HitClipPass` is an **AND-gate layered on the normal rect
test** in both `CUIWindow::Update` (hover) and the parent's `OnMouseAction` child routing, so
highlight and click always agree. Because a square tab's outline is its own bounding rect, applying
a clip there rejects nothing the rect test already passed — provably identical to vanilla hover.

`CUITabControl::Update` calls `ApplyStripHitClips` every frame: each non-parked tab gets its own
`HitPoly` (skipped entirely — clip cleared — for a square tab on a non-scrolling strip), and while
scrolling `SlidePolyIntoBand` **slides** the parallelogram inside the viewport walls, gauged on its
bottom edge so each slanted side moves as a unit (an earlier version clamped, which sheared the
shape). Then `CUITabScrollArrows::ApplyHitClips` does the same for the arrows. This is what fixes
the vanilla bug where two slant-overlapping tabs, or a tab plus an arrow, highlight at once because
their rects overlap.

## Scroll arrows (`CUITabScrollArrows`)

The pair lives in its own class, holding `CUIScrollArrowButton* m_arrow[2]` (a `CUITabButton`
subclass, so it inherits the same overlap/seam/hit-poly geometry) plus a per-side `m_pin` for
skin-authored placement. `Init(parent, msg_target)` attaches them as engine-owned children of the
control — they are **not** in `m_TabsArr`, so `RemoveAll` hides them explicitly. `SideOf(clicked)`
is how the control turns a click into a direction.

Three faces, decided in `EnsureBuilt(ref)` (called from `RecalcScroll` only when the strip actually
overflows):

- **skin-authored** — `<arrow_left>` / `<arrow_right>` nodes under the tab control node, read as
  plain 3t-buttons by `CUIXmlInit::InitTabControl` (`Init3tButton`) and handed over with
  `SetScrollArrow`. Validated: it needs a positive width/height and either a texture or text, else
  it is deleted with a log line. A non-zero `x` or `y` **pins** that axis where the skin put it,
  zero leaves that axis to the strip (`Layout`); a node without `<text>` stays glyphless because the
  art is the arrow.
- **spliced tab caps** — when the reference tab has an `ArtBase()` and `CanSplice` holds
  (`2 × CapWidthUI() ≥ TAB_ARROW_MIN_W` = 20 UI units, and the tab still has that much width left
  over), the arrow is two `CUIStatic` halves showing the begin-cap and end-cap texel slices of the
  same state art the tabs draw with (`SetupHalves` / `LayoutHalves` / `ApplyHalfArt`, re-applied on
  state change in `Update`). The raw cut edges meet in the middle so no vertical seam shows on
  filled states.
- **bare glyph** — otherwise a 20-unit-wide button with `"<"` / `">"` in the reference tab's font,
  coloured from its touched-state text colour.

Either way the glyph text and font come from the reference tab, so the arrows match the strip
without any art of their own.

## Tab icons

XML surface (both read in `CUIXmlInit::InitTabControl`):

- `<icon_layout>` on the tab control node — `width`/`height` are the box the icon is fitted into,
  `align` (`l`/`r`, else centre) and `vert_align` (`t`/`b`, else centre) snap the fitted icon's own
  anchor point onto the same point of the tab, and `x`/`y` offset it from there. Every default
  centres, so a strip that declares no slot needs no special case. Read **before** the first
  `SetTabIcon`. A missing or zero `width`/`height` falls back to a square of the tab height
  (`CUITabControl::IconBox`).
- `<icon>` inside a `<button>` — the art base name for that tab; a miss logs and is otherwise
  harmless.

`CUITabButton::SetIcon` probes `base_e` first: if it exists the icon is a staged
`CUI_IB_Static` (`_e`/`_d`/`_h`/`_t`, missing states just absent) drawn at full white; if only the
unsuffixed name exists it is a single-state icon **tinted with the caption's state colour**
(`StateTextColor`), so a flat glyph follows the tab's highlight like text does. `IconHasStateArt`
is what distinguishes the two at draw time. `FitIcon` scales the art down into the box preserving
aspect, then positions it by the anchor. `Update` tracks the button's `CurrentIBState()`.

`CurrentIBState()` and `StateTextColor()` were added to `CUI3tButton` for this and are the canonical
"what state is this button in / what colour is its text" helpers.

## Supported envelope

Vanilla `CUITabControl` never computed strip geometry — buttons were independent (own
x/y/w/h/texture/font, even multiple rows) — and every vanilla-era layout fits its strip by
construction, since scrolling did not exist. So the new geometry only engages when something
overflows.

- **Parked tabs** (zero width, e.g. Fatal Error's `eptTaskboard` at x=1000 w=0 — an invisible
  dispatch-target stub, a real vanilla-era pattern) are excluded from all strip geometry via
  `TabParked`: they do not count toward either span, are not repositioned, are never scrolled to,
  and stay dispatchable. "Tab 0" as a geometry/art reference always means `FirstStripTab()`, the
  first **non-parked** tab.
- Mixed widths and mixed art in one row work (overlap and hit shape are per tab); the clip-plane
  angle and the arrow face come from the reference tab.
- A multi-row strip is not detected or rejected. `RecalcScroll` measures it as one row, so a
  genuinely multi-row layout that also overflows would be laid out wrong — no vanilla layout does
  both.
- Dynamic tab captions can clip: width is fixed to the strip's tab width, and the caption is not
  measured.

## Deeper

- `src/xrGame/ui/UITabButton.h` / `.cpp` — per-tab geometry, art capture, icons, `Clone`,
  `SlantPoly`/`HitPoly`, the cap-width hack comment
- `src/xrGame/ui/UITabControl_scroll.cpp` — `RebuildTabOverlaps`, `AddTab`, `RecalcScroll`,
  slack/squeeze (`TabSlack`, `LaidPitch`, `LaidOverlap`), `ApplyScroll`, `EnsureVisible`,
  `DrawTabsClipped`, `SlidePolyIntoBand`, `ApplyStripHitClips`
- `src/xrGame/ui/UITabScrollArrows.h` / `.cpp` — `CUIScrollArrowButton` halves, `Adopt`,
  `EnsureBuilt`, `CanSplice`, `Layout`
- `src/xrGame/ui/UIXmlInit.cpp` (`InitTabControl`) — `<icon_layout>`, `<icon>`, `<arrow_left>`,
  `<arrow_right>`
- `src/xrGame/ui/UIPdaWnd.cpp` (`Init`, `RebuildExtraTabs`) — the Lua hook and active-tab restore
- `src/xrGame/ui/UIWindow.h` (`SetHitClip`, `HitClipPass`) — the hit-clip contract
- `src/xrGame/ui_defs.h` (`C2DFrustum::AddEdgePlane`, `ClipBudget`), `src/xrGame/ui_base.h`
  (`PushClipFrustum`, `ActiveClipFrustum`, `HasCustomClip`) — the clip stack
- `gamedata/scripts/pda_dynamic_tabs.script` — the registry; `gamedata/scripts/lua_help_ex.script`
  — the documented Lua surface
- Tab dispatch to content windows → `flows/pda-tabs.md`; canvas/clip/scaling model →
  `flows/ui-scaling.md`

## Maintenance

- If `CapWidthTexels` stops being `art.height()`, the arrow splice test and every slant assumption
  move with it — re-read the header comment first.
- Re-grep `TabParked` / `FirstStripTab` when touching strip geometry; parked tabs are the standard
  regression.
- `DrawTabsClipped` drawing no children is load-bearing for icons. Re-check it before attaching any
  new child to a tab.
- The `<icon_layout>` / `<arrow_left>` / `<arrow_right>` attribute set is the contract with PDA
  reskins; re-read `CUIXmlInit::InitTabControl` if it changes, and keep `lua_help_ex.script` in sync.
