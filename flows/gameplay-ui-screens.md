---
domain: flows/gameplay-ui-screens
tags: [modlayer, gameplay, unpacked, lua, ui, hud, main-menu, message-box, flow]
sources: [unpacked/scripts/ui_main_menu.script, unpacked/scripts/ui_dyn_msg_box.script, unpacked/scripts/ui_enemy_health.script, unpacked/scripts/ui_dosimeter.script, unpacked/scripts/ui_extra.script, unpacked/scripts/ui_freeplay_dialog.script, unpacked/scripts/ui_registrator.script, src/xrGame/UIDosimeter.cpp]
related: [flows/actor-menu-inventory-ui.md, flows/hud-ui-load.md, flows/options-runtime.md, flows/game-modes.md, flows/game-services.md, flows/item-behaviors-lua.md, flows/lua-callback-lifecycle.md, modlayer/gameplay/ui-scripts.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: gameplay UI screens (F109)

The Lua UI screens and HUD widgets outside the inventory window (F104 owns that). Two engine
seams recur: a **CUIScriptWnd** subclass for full dialogs (the engine pumps `Update`/`OnKeyboard`),
and the **HUD custom-static** pattern (`get_hud():AddCustomStatic(name, …)` → `:GetCustomStatic` →
`:wnd()` to attach widgets, `:RemoveCustomStatic` to drop them) for on-screen overlays. The XML
layouts are loaded with `CScriptXmlInit:ParseFile` (F19 builds the widget tree from XML). Six files
READ-IN-FULL; no `gamedata/` override of any.

## ui_main_menu — the SP main menu

`class "main_menu" (CUIScriptWnd)` built from `ui_mm_main.xml` (the "shniaga" wheel, a
`CUIMessageBoxEx`, version/disclaimer statics, a transient warning banner `msg_wnd`). Buttons wire
to the rest of the menu system: **new game** → `ShowFactionUI` (`ui_mm_faction_select`, F106),
**options** → `ui_options.UIOptions` (F17), **load/save** → `ui_load_dialog`/`ui_save_dialog`,
**last save**, **quit/disconnect/return**. `StartGame` runs the console
`start server(all/single/alife/new) client(localhost)`.

It is also a **callback hub** — extension points other modules (axr addons, debug tools) hook:
`main_menu_on_init`, `main_menu_on_quit`, `main_menu_on_keyboard`, `on_localization_change`, and
the `MAIN_MENU_RELOADED` UI event. On init with no level present it boots `xrs_debug_tools`.

**Save gating (F106 tie-in):** both the Save button and the F6 hardsave path fire
`SendScriptCallback("on_before_save_input", flags, …)` and abort if a gamemode plugin (agony /
timer / ironman) sets `flags.ret` — the menu honors the same save-veto bus as in-game saving. F6
builds a descriptive hardsave name (`<community> - <map> - hardsave - <date>`); quick-save/load
binds route to `level_input`. In `DEV_DEBUG` with no level, **F2** writes a canned
`character_creation` config and `StartGame`s (the fast new-game test path); NUMPAD0 reloads the
system ini, F1 shows debug help.

## The HUD widgets (custom-static pattern)

- **ui_enemy_health** — the last-hit-enemy health bar. Registered on `monster_/npc_/heli_on_hit`;
  shows only when the hit object has `db.storage[id].show_health` or the `video/hud/show_enemy_health`
  option is on, and only when the actor (`who:id()==0`) is the hitter. Adds `cs_enemy_health` from
  `ui_enemy_health.xml`, sets the progress bar from object health (heli via
  `get_helicopter():GetfHealth`), and labels it: heli → `st_ui_heli`, stalker → `character_name`,
  monster → an encyclopedia name resolved from the section `kind`/`species` (a `strings` table plus
  `rotan`/`psy_dog` special cases), else story id. Auto-removes after 30 s unless hit again, or
  immediately on the `from_death_callback` marker.
- **ui_dosimeter** — a Lua dosimeter HUD widget (`CUIScriptWnd` singleton via `get_UI`) that draws
  a 4-segment digital readout from `hud\p3d` textures with a blinking work indicator. The reading is
  `level.get_env_rads()×2500` or `actor.radiation×10000×0.387` (per `item_device.dosimeter_env_rads_mode`),
  spiked hard during surges/psi-storms. **No base-corpus caller invokes `get_UI`** — the engine's
  own `UIDosimeter.cpp` loads `ui_dosimeter.xml` and renders the device, and the device logic
  (mode toggle, identical rad formula) lives in `item_device.script` (F102); this Lua widget is
  dormant/alternate in the shipped corpus.
- **ui_extra** — a battery-condition HUD (`cs_battery_state` from `maingame_battery.xml`,
  `axr_battery.getBatteryCondition`) plus a `get_protection` helper. **Dead code:** its
  `on_game_start` `actor_on_update` registration is commented out, nothing references the module
  anywhere in the corpus, and `get_protection` dereferences a nil `self` (module function, not a
  method) so it would error if called.

## ui_dyn_msg_box — the reusable widget toolkit

Alundaio's dynamic-UI library (no fixed XML of its own; pulls layouts from `utils_ui.get_utils_xml`).
Four `CUIScriptWnd`/`CUIListBoxItem` classes used across the corpus:

- **msg_box_ui** — a `CUIMessageBoxEx` wrapper taking `func_ok`/`func_cancel` + vararg params, so
  any code can pop a confirm/yes-no box with callbacks (ESC = cancel).
- **multi_choice** — a numbered list-of-actions dialog; each entry carries a `func` run on
  double-click.
- **context_menu** / **context_props** — cursor-positioned right-click popups (a list of
  translated actions, each with a `func`, `context_props` also passing `owner` + per-item params and
  drawing a hover highlight). They flip `owner.disabled` while open and auto-hide when the cursor
  leaves. Consumers: `ui_load_dialog` (multi_choice/msg_box), `ui_sr_teleport` (context_props),
  `xr_effects` (msg_box_ui). The inventory window (F104) uses `utils_ui`'s own context framework,
  not these.

## ui_freeplay_dialog — the Pripyat endgame prompt

A 40-line `CUIMessageBoxEx` wrapper for the "stay in the Zone / leave" finale: its OK/Yes/No
handlers give the `pri_a28_actor_in_zone_stay`/`leave` info portions. Driven by `xr_effects`
(`ui_freeplay_dialog.show(...)`) at the story trigger; `main()` is a test stub.

## ui_registrator — vestigial MP UI clsid

`get_ui_clsid(mode)` maps a multiplayer mode string to a UI game-class clsid (`UI_SINGL`/`UI_DM`/…).
The engine calls it on the game-class boot path (no Lua caller); only `"single"` is reachable in
Anomaly. Direct sibling of `game_registrator` (F107).

## Where this sits

These are F65-bus plugins and `CUIScriptWnd` dialogs over the engine UI core (F19,
`flows/hud-ui-load.md`): XML→widget-tree, the HUD custom-static registry, message-box events.
Neighboring UI domains: the inventory/actor menu (F104), PDA tabs (F110 ⬜), debug tools (F111 ⬜),
news (F112 ⬜), options (F17). All routed by the M20 map (`modlayer/gameplay/ui-scripts.md`).

## Deeper

- Inventory window / context menus / quick slots → `flows/actor-menu-inventory-ui.md` (F104).
- Engine UI core (XML load, HUD, dialog base) → `flows/hud-ui-load.md` (F19).
- Save-veto bus the menu honors → `flows/game-modes.md` (F106).
- Options dialog the menu opens → `flows/options-runtime.md` (F17).
