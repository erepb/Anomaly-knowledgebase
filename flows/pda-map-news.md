---
domain: flows/pda-map-news
tags: [xrgame, pda, map, map-location, news, contacts, flow]
sources: [src/xrGame/map_manager.cpp, src/xrGame/map_manager.h, src/xrGame/map_location.cpp, src/xrGame/map_location.h, src/xrGame/map_spot.cpp, src/xrGame/map_spot.h, src/xrGame/ui/UIMap.cpp, src/xrGame/level_script.cpp, src/xrGame/PDA.cpp, src/xrGame/PDA.h, src/xrGame/game_news.cpp, src/xrGame/actor_communication.cpp]
related: [flows/tasks.md, flows/hud-ui-load.md, flows/relations-info.md, flows/save-load.md, flows/pda-tabs.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-22
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
- Promote the *(agent-traced)* `CMapLocation`/`CPda` feel-touch / `GAME_NEWS_DATA` bodies to verified
  when a PDA/map task needs the exact spot-loading or contact logic.
