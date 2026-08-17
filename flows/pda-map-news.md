---
domain: flows/pda-map-news
tags: [xrgame, pda, map, map-location, news, contacts, flow]
sources: [src/xrGame/map_manager.cpp, src/xrGame/map_manager.h, src/xrGame/map_location.cpp, src/xrGame/map_location.h, src/xrGame/map_spot.cpp, src/xrGame/map_spot.h, src/xrGame/ui/UIMap.cpp, src/xrGame/ui/UIMapWnd.cpp, src/xrGame/ui/UIMapWnd.h, src/xrGame/ui/UIPropertiesBox.cpp, src/xrGame/ui/UIPropertiesBox.h, src/xrGame/level_script.cpp, src/xrGame/PDA.cpp, src/xrGame/PDA.h, src/xrGame/game_news.cpp, src/xrGame/actor_communication.cpp, gamedata/scripts/callbacks_gameobject.script, gamedata/scripts/modxml_fix_faction_spot_size.script]
related: [flows/tasks.md, flows/hud-ui-load.md, flows/relations-info.md, flows/save-load.md, flows/pda-tabs.md, features/pda-tab-scroll.md, modlayer/unlocalizer.md, modlayer/dxml.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: PDA, map locations & news

The data layer behind the PDA: **map locations** (spots/pointers that track an object on the map),
**PDA contacts** (nearby NPCs sensed via Feel::Touch), and the **news/notification** queue. This is
the registry/data side; the UI widgets that draw it are built from XML (`flows/hud-ui-load.md`). Tasks
own map locations (`flows/tasks.md`), and relation spots use the goodwill system
(`flows/relations-info.md`).

**Provenance:** spine personally code-verified — `CMapManager::AddMapLocation` (`xr_new<CMapLocation>`),
`CPda : CHudItemObject, Feel::Touch`, and `CActor::AddGameNews` (pushes into `game_news_registry`). The
`CMapManager::Update`/`RemoveMapLocation`, `CMapLocation` binding/spot/`CalcPosition`,
`CRelationMapLocation`, the `CPda` feel-touch contact → `AddRelationLocation` path, `GAME_NEWS_DATA`,
and the `give_game_news` Lua surface are confirmed homes but *(agent-traced — not personally re-read)*.
The task↔map-location seam is verified in `flows/tasks.md`.

## Map locations (manager verified at entry; details agent-traced)

`CMapManager` (`Level().MapManager()`) owns all map locations in an ALife registry keyed by
`(object_id, spot_type)` (`SLocationKey`). **`AddMapLocation(spot_type, id)`** (verified)
`xr_new<CMapLocation>(spot_type, id)` and registers it; `RemoveMapLocation`/`RemoveMapLocationByObjectID`
defer-destroy and erase; `GetMapLocation(s)` look up. **`Update()`** *(agent-traced)* runs a culled
round-robin: each `CMapLocation::Update` checks TTL and whether its tracked object still exists, recomputes
cached position/direction/level, sorts actual-first, and pops dead ones into a deferred-destroy queue
(freed next frame).

**`CMapLocation`** *(agent-traced)* binds an `object_id` (or an offline `CSE_ALifeDynamicObject`) to a
**spot** graphic + hint, on a specific level map. It loads its visuals from `map_spots.xml` by
`spot_type` (level-map spot, minimap spot, complex spot, and the off-screen **pointer** arrow), tracks
the object's position (`CalcPosition` — live object or SE position; `pos_to_actor` variant), and shows
an "active task" border when its owning task is the active one (`flows/tasks.md`).
**`CRelationMapLocation`** is the NPC-marker subclass: it picks the spot by relation
(friend/neutral/enemy/deadbody via `RELATION_REGISTRY`, `flows/relations-info.md`) and gates visibility
on memory/helmet (enemies hidden unless recently seen).

## Map spots: drawing & draw order (verified)

A `CMapLocation` owns up to three spot graphics — level-map (`CMapSpot`), minimap (`CMiniMapSpot`),
complex (`CComplexMapSpot`) — plus their off-screen pointer arrows; all subclass `CUIStatic`
(`map_spot.cpp`). `CMapLocation::LoadSpot` builds them from `map_spots.xml` by `spot_type`: the
`<level_map spot="X"/>` node names the `_spot` element, which the spot loads through
`CUIXmlInit::InitStatic`. `pointer` is optional (omit it for an overlay that must not add a second
arrow).

- **Draw order is data-driven via `location_level`.** `CMapSpot::Load` reads the `location_level`
  attribute (default 0); `CUILevelMap::UpdateSpots` (`ui/UIMap.cpp`) re-attaches the frame's visible
  spots then `std::stable_sort`s them by `get_location_level` ascending — so **lower draws first
  (behind), higher on top**, ties keep insertion order. Negative values are legal and sit behind the
  default-0 faction/quest spots. This is how a mod stacks a coloured outline *under* an icon: two
  spots on one object, the outline at a lower level.
- **Per-spot tint and shader.** The `<texture>` carries r/g/b/a (`CUIXmlInit::GetColor`, channels
  default 255) → the spot's `dwColor`, applied as a per-vertex *multiply* by the UI shader
  (`CUIStaticItem::Render`). The shader is selectable per element via the `<texture shader="...">`
  attribute (`CUIXmlInit::InitTexture` → `InitTextureEx`); see `flows/hud-ui-load.md` for `hud\default`
  (multiply) vs `hud\font` (alpha-as-mask, solid-colour silhouette).
- **`static_border` child is active-task-only.** A `_spot` may define a `static_border` child static
  (a second textured layer), but `CMapLocation::UpdateSpot` calls `show_static_border(true)` *only*
  when the location's owning task is the active task — it is not a general second-layer slot for
  non-task spots.

## Right-click context menu (engine seam → Lua bus, verified)

Right-clicking a spot opens a **properties (context) menu**: the engine dispatches and owns the
`CUIPropertiesBox`, but the menu *content* is built by the mod layer over the same `map_spot_menu_*`
bus the PDA tabs use (`flows/pda-tabs.md`).

- **Select the spot.** `CMapSpot::OnMouseDown(MOUSE_2)` (`map_spot.cpp`) sends `MAP_SELECT_SPOT2` to its
  message target and returns `true`. RBUTTON dispatch in `CUIWindow::OnMouseAction` walks children in
  **reverse draw order and stops at the first that consumes it** → only the **topmost** overlapping spot
  is selected. `CUILevelMap::SendMessage` (`ui/UIMap.cpp`) routes it to
  `CUIMapWnd::ActivatePropertiesBox(spot)`.
- **Build (multi-spot merge).** `CUIMapWnd::ActivatePropertiesBox` (`ui/UIMapWnd.cpp`) `RemoveAll()`s
  the box, gathers **all** spots under the cursor (`GatherSpotsUnderCursor`, deduped by object id,
  topmost first), and per spot adds an `AddHeader` title (first hint line via `clean_hint_title`;
  the literal `"no hint"` XML default is treated as empty → level-name fallback) then calls Lua
  **`pda.property_box_add_properties(box, id, level, hint)`** → broadcast
  **`map_spot_menu_add_property`**: each mod (`game_fast_travel`, `tasks_placeable_waypoints`,
  Warfare, …) conditionally `box:AddItem(text)` keyed on `id`. Headers whose spot contributed nothing
  are removed. Every row (incl. submenu leaves, `TagSubmenuOwners`) is tagged with its owner
  `{id, level, hint}` in `m_property_owners`. Finally **`_G.COnRightClickMap(box, table)`**
  (`gamedata/scripts/callbacks_gameobject.script`) fires once for position-based "anywhere" items,
  passing `object_id` = top spot id, or **65535 when the click hit empty map**. `Show()` only if the
  box gained items.
- **Click (context replay).** Selecting an item sends `PROPERTY_CLICKED`; `CUIMapWnd::SendMessage`
  first **replays** `pda.property_box_add_properties` for the clicked row's owner into the hidden
  `m_UIScratchBox` (skipped for 65535). Mods are *stateful*: they wipe + rebuild dispatch state at the
  top of every add call (`game_fast_travel` keeps a label→action `menu_options` map; PAW dispatches on
  `last_clicked_id`, ignoring the passed id), so after a merged N-spot build that state reflects the
  *last* spot enumerated — the replay resets it to what a plain single-spot right-click on the owner
  would have left. Then `pda.set_map_spot_context(id, level)` sets `pda.map_spot_property`, and
  **`pda.property_box_clicked(box)`** reads the clicked item's **text** and fires
  **`map_spot_menu_property_clicked(box, id, level, text, tag)`** — each mod checks "is this text mine
  for this id?" and acts.

`pda.map_spot_property` is one shared table (written by the build + `COnRightClickMap` +
`set_map_spot_context`, read by the click) reachable cross-script only because the **unlocalizer**
strips its `local` (`modlayer/unlocalizer.md`). Known residual: replayed labels that embed live data
(fast travel's distance/cost) can drift from the displayed text if the actor moved while the menu was
open → that click no-ops once.

`pda.set_map_spot_context` and the `property_box_clicked` that forwards the tag are **repo additions in
`gamedata/scripts/callbacks_gameobject.script`**, replacing the base-game `pda.property_box_clicked`
wholesale — the base version reads only the label. Two extra `CUIPropertiesBox` surfaces support this
menu shape, all Lua-exported and documented in `gamedata/scripts/lua_help_ex.script`:

- **`AddHeader(text)`** — a non-selectable coloured title row (what groups a merged multi-spot menu).
- **`AddSubmenu(label)`** — appends a `label >` row and returns the child box to populate
  (`AddItem`/`AddHeader`/nested `AddSubmenu`). The child expands on hover and is owned by the parent
  (`m_item_submenus`, torn down by `RemoveAll`); the engine tags submenu leaves with their owning spot
  via `TagSubmenuOwners`, so a leaf click replays and dispatches like a top-level row. This **replaced**
  the vanilla one-shot mechanism (a `CUIPropertiesBox` constructed with a child box + `ShowSubMenu()`);
  its only user, the MP demo-play control (`UIDemoPlayControl.cpp`), now positions and hides its second
  box itself.
- **`AddItem(text, id)`** — a `u32` per-row id, read back as `GetSelectedItem():GetTAG()` and passed as
  the trailing `tag` argument of `map_spot_menu_property_clicked`. This is what lets one menu carry
  **duplicate labels** (the same action for two merged spots) without the text-matching dispatch
  becoming ambiguous. `RemoveItemByTAG` removes by that id.

`CUIMapWnd::DrawHint` early-outs while the properties box is shown, so the spot tooltip no longer
draws under the open menu.

Unrelated repo fix on the same screen: `gamedata/scripts/modxml_fix_faction_spot_size.script` (a DXML
`on_xml_read` hook) rewrites `circle_army_spot` / `circle_ecolog_spot` in `map_spots.xml` and
`map_spots_16.xml` to 24x24, matching every other faction — stock ships them at 28 in the widescreen
variant, which rendered those two icons oversized.

## PDA contacts (class verified; feel-touch agent-traced)

`CPda : CHudItemObject, Feel::Touch` (verified) — the PDA is a HUD device (show/hide/battery like a
detector, `flows/detector-artefact.md`'s `CCustomDevice` pattern) that also **feels** nearby objects.
*(agent-traced:)* it `feel_touch_update(Position(), m_fRadius)` to track living non-monster NPCs in
`m_active_contacts`; `feel_touch_new`/`_delete` call the inventory owner's `NewPdaContact`/`LostPdaContact`
→ `MapManager.AddRelationLocation`/`RemoveRelationLocation` (so nearby NPCs appear as relation spots on
the minimap). Low battery dims the PDA screen and fires a Lua `on_low_battery` functor.

## News (entry verified; structure agent-traced)

`GAME_NEWS_DATA` *(agent-traced)* — a news/talk message (caption, text, texture, show-time, type,
receive-time), serializable. **`CActor::AddGameNews(news)`** (verified) pushes it into the actor's
`game_news_registry` and dispatches it to the HUD (`UIMainIngameWnd->ReceiveNews`);
`AddGameNews_deffered` queues it to fire after a delay. The Lua surface
*(agent-traced)*: `db.actor:give_game_news(caption, text, texture, delay, show_time[, type])` →
`CScriptGameObject::GiveGameNews` → `AddGameNews[_deffered]`. The queue persists with the save.

## Add → update → remove sequence (verified entry; body agent-traced)

1. **Add:** a task (`CGameTask::CreateMapLocation`, `flows/tasks.md`) or PDA contact or script calls
   `CMapManager::AddMapLocation(spot_type, id)` → new `CMapLocation` (loads spots from `map_spots.xml`),
   registered; the task caches the pointer and tags `m_owner_task_id`.
2. **Per frame:** `CMapManager::Update` ticks each location (TTL, object-alive, position/direction/level),
   sorts, and the UI draws spots/pointers from the cached values; active-task locations get a border.
3. **Remove:** task complete/fail (or contact lost, or TTL) → `RemoveMapLocation` → deferred destroy →
   `GameTaskManager::MapLocationRelcase` unlinks the owning task and alerts the UI.

## Isolation ledger (the seams)

- **Tasks:** `CGameTask` owns a `CMapLocation` via `AddMapLocation`; the back-link is `m_owner_task_id`
  + `GameTaskManager::HasGameTask(location)` / `MapLocationRelcase` (`flows/tasks.md`).
- **Relations:** `CRelationMapLocation` picks its spot from `RELATION_REGISTRY` attitude
  (`flows/relations-info.md`); PDA contacts add/remove these.
- **UI:** spots/pointers are `CUIStatic`-derived widgets attached to the map window; news goes to
  `UIMainIngameWnd` (`flows/hud-ui-load.md`). The PDA device is a HUD item.
- **Save/load:** map-location registry, news registry, and the task→location link all persist
  (`flows/save-load.md`).
- **Lua surface:** two families. Object map-spots ARE scriptable: `level.map_add_object_spot` /
  `map_add_object_spot_ser` / `map_remove_object_spot` / `map_change_spot_hint` / `map_has_object_spot`
  (`level_script.cpp`, wrapping `CMapManager::AddMapLocation`/`RemoveMapLocation`/`GetMapLocation`/
  `HasMapLocation`) — mods (e.g. Warfare) drive their markers entirely this way. News: `give_game_news`
  (verified-called). What is commented-out is the *broader* `CMapManager` surface (the `map_manager`
  accessor and friends), not the object-spot calls.
- **Config:** `map_spots.xml` (spot/pointer/border defs per `spot_type`), the PDA `.ltx` section
  (`radius`, power-saving thresholds, sounds), task config (`map_location`/`map_object_id`).

## Threading

Main thread (map update, PDA feel-touch on the scheduled channel, news dispatch). Single-threaded with
gameplay.

## Failure & edge cases

- **Location dies with its object:** a `CMapLocation` whose tracked object despawns (or TTL expires)
  goes non-actual and is destroyed — a "missing quest marker" is often the tracked object gone or the
  task no longer active.
- **Relation spot visibility:** enemy markers are hidden unless recently seen (memory/helmet gate) — not
  a bug; intended fog-of-war.
- **News is queued, not instant for deferred:** `AddGameNews_deffered` shows after its delay; immediate
  `AddGameNews` shows now.
- **`AddMapLocation` does not dedup:** `CMapManager::AddMapLocation` unconditionally `xr_new`s a
  `CMapLocation` and pushes it, so adding the same `(object_id, spot_type)` twice creates a duplicate.
  Callers must guard (`map_has_object_spot`, or remove-before-readd). `RemoveMapLocation` erases the
  registry entry immediately and defers only the object destroy to next frame, so a remove-then-add of
  the same key within one frame is safe (no duplicate in the live list).

## Deeper

- Manager: `src/xrGame/map_manager.cpp`/`.h` (`AddMapLocation`, `RemoveMapLocation`, `Update`,
  `AddRelationLocation`), `map_location_defs.h` (`SLocationKey`)
- Location/spot: `src/xrGame/map_location.cpp`/`.h` (`CMapLocation`, `CRelationMapLocation`,
  `LoadSpot`, `CalcPosition`, `UpdateSpot` border-gating), `map_spot.cpp`/`.h` (`CMapSpot` family,
  `Load` reads `location_level`, `show_static_border`)
- Lua spot bindings: `src/xrGame/level_script.cpp` (`map_add_object_spot`, `map_add_object_spot_ser`,
  `map_remove_object_spot`, `map_change_spot_hint`, `map_has_object_spot`)
- Spot draw order: `src/xrGame/ui/UIMap.cpp` (`CUILevelMap::UpdateSpots` stable-sort by
  `get_location_level`); per-element tint/shader in `flows/hud-ui-load.md`
- Right-click menu: `src/xrGame/map_spot.cpp` (`OnMouseDown(MOUSE_2)`→`MAP_SELECT_SPOT2`),
  `src/xrGame/ui/UIMap.cpp` (`CUILevelMap::SendMessage` routing), `src/xrGame/ui/UIMapWnd.cpp`
  (`ActivatePropertiesBox`, `PROPERTY_CLICKED` dispatch); Lua bus in `flows/pda-tabs.md`,
  `gamedata/scripts/callbacks_gameobject.script` (`COnRightClickMap`)
- PDA: `src/xrGame/PDA.cpp`/`.h` (`CPda`, feel-touch contacts, battery)
- News: `src/xrGame/game_news.cpp`/`.h` (`GAME_NEWS_DATA`), `actor_communication.cpp`
  (`CActor::AddGameNews`), `script_game_object_inventory_owner.cpp` (`GiveGameNews`)
- Task seam: `flows/tasks.md`; relation spots: `flows/relations-info.md`; UI: `flows/hud-ui-load.md`

## Maintenance

- Keep the "object map-spots are scriptable via `level.map_*`, but `AddMapLocation` doesn't dedup"
  facts — they explain how mods (Warfare) drive markers and why every add must be guarded.
- Re-grep the `map_add_object_spot` family in `level_script.cpp` if the Lua spot surface changes, and
  `CUILevelMap::UpdateSpots` if the `location_level` draw-order sort moves.
- Re-grep `CMapManager::AddMapLocation`/`Update` if the registry or culling changes.
- Re-grep `ActivatePropertiesBox` / `MAP_SELECT_SPOT2` / `COnRightClickMap` if the right-click menu
  dispatch changes; the Lua `map_spot_menu_*` bus is owned by `flows/pda-tabs.md`.
- Promote the *(agent-traced)* `CMapLocation`/`CPda` feel-touch / `GAME_NEWS_DATA` bodies to verified
  when a PDA/map task needs the exact spot-loading or contact logic.
