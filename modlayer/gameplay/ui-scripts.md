---
domain: modlayer/gameplay/ui-scripts
tags: [modlayer, gameplay, unpacked, lua, ui, hud, pda, map]
sources: [unpacked/scripts/ui_main_menu.script, unpacked/scripts/ui_dyn_msg_box.script, unpacked/scripts/ui_enemy_health.script, unpacked/scripts/ui_inventory.script, unpacked/scripts/pda.script, unpacked/scripts/ui_debug_launcher.script, unpacked/scripts/xrs_debug_tools.script, unpacked/scripts/news_manager.script, unpacked/scripts/dynamic_news_manager.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/gameplay-ui-screens.md, flows/actor-menu-inventory-ui.md, flows/pda-tabs.md, flows/ui-debug-tools.md, flows/ui-news-tips.md, flows/hud-ui-load.md, flows/options-runtime.md, flows/pda-map-news.md, GLOSSARY.md]
updated: 2026-06-12
verified: map
---

# UI scripts — L1 map (M20)

The `ui_*` Lua layer: full-screen dialogs (`CUIScriptWnd` subclasses the engine pumps) and HUD
overlays (the `get_hud():AddCustomStatic` registry), all built from `configs/ui/*.xml` via
`CScriptXmlInit` (engine UI core = F19, `flows/hud-ui-load.md`). This is the gameplay UI tier; the
engine widget classes and XML loader are F19, options is F17.

## The pieces

| Cluster | File(s) | Role | Flow |
|---------|---------|------|------|
| inventory window | `ui_inventory`, `ui_item`, `ui_itm_details`, `ui_companion_inv`, `utils_ui` | the Lua window that replaced the engine ActorMenu (4 modes, context menu, quick slots) | F104 ✅ |
| menu & HUD screens | `ui_main_menu`, `ui_enemy_health`, `ui_dosimeter`, `ui_extra`, `ui_freeplay_dialog`, `ui_registrator` | SP main menu (+ save-veto/callback seams), HUD widgets, endgame prompt, vestigial MP clsid | F109 ✅ |
| widget toolkit | `ui_dyn_msg_box` | reusable message-box / multi-choice / context-menu / context-props classes | F109 ✅ |
| PDA & tabs | `pda`, `pda_actor`, `pda_flagger`, `ui_pda_*_tab` (contacts/encyclopedia/npc/radio/relations/warfare) | the PDA hub (engine `set_active_subdialog` dispatch + ranking board + map spots) and its tabs | F110 ✅ |
| debug tools | `ui_debug_launcher`/`_main`/`_item`/`_weather`/`_lighting`/`_wpn_hud`, `ui_ctrl_lighting`, `ui_map_debug_ex`, `xrs_debug_tools`, `debug_cmd_list` (+ gamedata `zz_ui_debug_inputs`, `imgui_lua_debug`) | dev-only launcher + HUD inspector + command registry, all behind `DEV_DEBUG` | F111 ✅ |
| news & notifications | `news_manager`, `dynamic_news_manager`/`dynamic_news_helper`, `faction_expansions` | universal PDA-tip/showcase dispatcher (~50 callers) + the self-scheduling Dynamic News ambient generator | F112 ✅ |
| save/load & misc | `ui_load_dialog`, `ui_save_dialog`, `ui_sleep_dialog`, `ui_numpad`, `ui_scenes`, `ui_wpn_params`, `ui_mutant_loot`, `ui_workshop`, `ui_sr_teleport` | leaf dialogs (each a `CUIScriptWnd` on the F19/F109 patterns) | leaves |
| options | `ui_options`, `ui_options_modded_exes` | settings dialog + the runtime option store | F17 |

## How it fits

Two engine seams recur (both F19): a **`CUIScriptWnd`** subclass for full dialogs (the engine
calls `Update`/`OnKeyboard`), and the **HUD custom-static** pattern for on-screen overlays
(`AddCustomStatic`/`GetCustomStatic`/`RemoveCustomStatic`). Screens read the option store via
`ui_options.get` (F17) and hook the F65 callback bus (e.g. `ActorMenu_on_before_init_mode` for the
inventory, `main_menu_on_*` for the menu, `on_before_save_input` for save vetoes). The PDA map-spot
menu seam is shared with the travel features (F39 / F107).

## Subsystem deep-dives (delegates to flows)

- Menu, HUD widgets, message-box toolkit, dead `ui_extra`, dormant Lua dosimeter, vestigial
  `ui_registrator` → `flows/gameplay-ui-screens.md` (F109) ✅
- Inventory window, context menus, quick slots, loot/trade/repair modes →
  `flows/actor-menu-inventory-ui.md` (F104) ✅
- PDA hub + tabs (contacts/npc/radio/relations/warfare + ranking board + map spots; warfare
  commander map plugins) → `flows/pda-tabs.md` (F110) ✅
- Debug launcher + HUD inspector + command registry + gamedata delta (all `DEV_DEBUG`-gated) →
  `flows/ui-debug-tools.md` (F111) ✅
- PDA tips/showcase dispatcher + Dynamic News ambient generator (channels/ticks/NewsToggle gate) →
  `flows/ui-news-tips.md` (F112) ✅
- Engine UI core (XML→widget tree, HUD, dialog base classes) → `flows/hud-ui-load.md` (F19)
- Options dialog + runtime option store → `flows/options-runtime.md` (F17)
- PDA & map locations data layer (engine tier) → `flows/pda-map-news.md` (F39)

## Maintenance

- F104 ✅ + F109 ✅ + F110 ✅ + F111 ✅ + F112 ✅ — **the M20 ui_* flow coverage is complete**
  (leaf save/load/sleep/numpad dialogs remain per-file leaves on the F19/F109 patterns; the axr/utils
  tiers are F113–F117). `gamedata/` overrides/additions in this map's scope: `ui_options_modded_exes` +
  `options_modded_exes_debug` (F17 domain), and the debug delta `zz_ui_debug_inputs` /
  `imgui_lua_debug` (F111). No F109/F110/F112 file is overridden. Re-check `gamedata/scripts/ui_*`,
  `*debug*` and `*news*` on pickup.
