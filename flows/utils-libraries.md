---
domain: flows/utils-libraries
tags: [modlayer, gameplay, unpacked, lua, utils, helper, library, flow]
sources: [unpacked/scripts/utils_obj.script, unpacked/scripts/utils_item.script, unpacked/scripts/utils_data.script, unpacked/scripts/utils_xml.script, unpacked/scripts/utils_ui.script, unpacked/scripts/utils_stpk.script]
related: [INDEX.md, modlayer/gameplay/utils-db.md, flows/db-registry.md, flows/scheme-engine-core.md, flows/script-npc-actions.md, flows/axr-behaviour-addons.md, flows/trade-economy.md, flows/actor-menu-inventory-ui.md, flows/save-load.md, flows/condlist-library.md, flows/hud-ui-load.md, modlayer/gameplay/server-entities-lua.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: utility libraries (`utils_*`)

The six `utils_*.script` files are the **cross-cutting helper layer** almost every other gameplay
script calls into. They are flat function libraries (no shared class hierarchy) — ~10 000 lines, ~500
functions — each scoped to one domain. This flow is a **navigable catalog**: what helper lives where,
the load-bearing ones, and the non-obvious gotchas. It is *not* a call-graph; these are leaves the rest
of the corpus depends on.

**Provenance:** **verified, no agent (catalog read).** Full function-name inventory pulled for all six.
Load-bearing regions READ-IN-FULL: `utils_obj` cover/movement primitives (`validate`/`lmove`/
`find_*_cover`/`try_go_*`); `utils_data` the `cfg_file` class + `parse_*` family; `utils_item`
`get_item_cost`/`get_item_trade_status` + the cost-cache callbacks. `utils_ui`, `utils_xml`,
`utils_stpk` are **structure-surveyed** (inventory + headers + the class skeletons) — their bodies are
mechanical per-field code, not re-read line by line. **No `gamedata/` override of any of the six**
(verified by `find`).

---

## utils_obj — game objects, squads, navigation (931 lines)

Helpers for live `game_object`s and the AI graph.

- **Identity / queries:** `is_actor`, `is_trader`, `in_los` (line-of-sight), `nearest_object` /
  `get_nearest_stalker` / `get_nearest_object`, `graph_distance`, `npc_in_zone`.
- **Online/offline:** `switch_online` / `switch_offline` (force an alife object's state).
- **Lists from config:** `get_rank_list`, `get_reputation_list` / `get_reputation_name`,
  `get_communities_list` (the rank/reputation/faction enumerations the UI and relations code read).
- **Squads:** `get_commander` / `get_squad_count` / `create_squad` / `assign_squad_to_smart` /
  `execute_script_on_squad` (+ `_not_commander`) — the squad-manipulation primitives used by warfare
  (F105), companions (F114), and scripted squads.
- **Cover & movement primitives (load-bearing):** `validate` (is a level vertex accessible and not
  already claimed), `lmove` (move an NPC to a vertex while **reserving** it in
  `db.used_level_vertex_ids` so two NPCs don't claim the same cover), `find_random_cover` /
  `find_close_cover` / `find_cover_in_rand_direction` (wrap `npc:best_cover`), and `try_go_cover` /
  `try_to_strafe` / `try_go_backward` / `try_go_aside_object` / `try_go_position`. **This is the
  movement skeleton the combat schemes reuse** — F113's `axr_fight_from_cover`/`stalker_panic`/etc.
  call straight into these, as do the stock `xr_*` combat schemes.
- **Waypoints:** `parse_waypoint_data` / `path_parse_waypoints` (+ `_from_arglist`) /
  `stalker_at_waypoint` / `get_nearest_waypoint` — parse the `wp00|a=…|p=…` patrol-path strings.
- **Time / sound:** `is_day` / `is_night`, `play_sound`, `time_spent_in_zone` / `is_time_spent_in_zone`.

## utils_item — items, weapons, outfits, trade (1441 lines)

Everything about inventory items and their config.

- **Params:** `get_ammo`, `get_param` / `get_wpn_param` / `get_cond_static`, `get_defined_uses`,
  `item_vector_to_num` / `item_float_to_num`.
- **Addons (scope / silencer / grenade-launcher):** the full `has_/can_/get_attached_/attach_/detach_`
  family (`has_scope`, `can_attach_silencer`, `get_attached_gl`, `attach_addon`, …) +
  `get_addon_flag` / `get_addon_status`.
- **Outfits:** `get_outfit_protection` / `get_outfit_property` / `get_outfit_belt_size`.
- **Inventory location:** `in_actor_inv` / `in_actor_ruck` / `in_npc_inv`, `has_item_by_id`,
  `has_quest_item`, `is_overweight`.
- **Stacks & amounts:** `get_amount` / `collect_amount` / `transfer_amount` / `create_amount` — the
  multi-item-stack manipulation used by trade and reward code.
- **Condition / charge:** `is_degradable` / `degrade` / `discharge` / `get_item_remaining_uses`,
  `switch_on_off` (+ `_steps`) for devices.
- **Pricing (load-bearing):** `get_item_cost(obj, profile)` and `get_item_trade_status(obj, profile)` —
  the **canonical Lua price/tradability functions** the trade UI uses (F101). Cost = base `obj:cost()`
  × trade-config factor × discount × a condition exponent (or per-use/per-round fraction for
  multiuse/ammo); each branch fires the **`on_get_item_cost` script callback** so mods can rewrite the
  final price. `get_item_trade_status` returns 1=can't-trade / 2=too-damaged / 3=hidden / 4=tradable.
  A cost **cache** (`check_cache`/`add_cache`/`get_cache`, invalidated via `server_entity_on_unregister`)
  avoids recomputing on every cell redraw.
- **Upgrades:** `extract_upgrade` / `get_upgrades_tree` / `get_upgrades_tools` / `get_upgrades_installed`
  / `has_upgrades` / `has_this_upgrade` — read the weapon/outfit upgrade tree.
- **Timed events:** `delay_event` / `delay_event_timer` (one-shot deferred callbacks).

## utils_data — math, vectors, time, INI parsing (1279 lines)

The lowest-level grab-bag: numbers, strings, config, serialization.

- **Math / geometry:** `rad2deg` / `deg2rad`, `mean_random`, `string_to_vector` / `vector_to_string` /
  `vector_cmp`, `angle_diff` / `angle_left` / `no_need_to_rotate` (+ `_xz`), `pos_in_rect`.
- **Time (CTime serialization):** `CTime2table` / `table2CTime`, `w_CTime` / `r_CTime` (net-packet),
  `CTimeToSec`, `CTimeAddSec` — how game time is saved/restored and compared.
- **String / path:** `startsWith`, `get_ext` / `get_path` / `get_comment` / `trim_comment`,
  `hex2string` / `string2hex` / `float2hex`, `pairsByKeys`, `to_str`, `get_fsgame` / `fspath` /
  `fsgame_append` (filesystem path helpers).
- **Config parsing (load-bearing):** `parse_params`, `parse_data` (+ `_1v`), `parse_syn_data`,
  **`parse_condlist`**, `parse_ini_section_to_array` (+ `_to_bool_array`), `parse_list_ex`,
  `collect_section` / `collect_sections`, `read_from_ini` / `cfg_get_string` / `prefix_r_string`,
  `get_scheme_by_section`. **These are the primitives the scheme/condlist engine is built on** — the
  `{+info -info =func()} value %effect%` condlist grammar is tokenized here (see
  `flows/condlist-library.md`, `flows/scheme-engine-core.md`).
- **Localization:** `collect_translations` / `file_to_table` / `on_localization_change`.
- **`findfunction(x, tbl)`** — resolve a `"namespace.func"` string to the actual function (the
  string→functor mechanism behind condlist `=func()` calls and dialog/effect dispatch).
- **`cfg_file` class (load-bearing):** a Lua-side **read/write** INI wrapper — the engine `ini_file`
  is read-only, so persisting runtime state needs this. `GetValue`/`GetKeys`/`SetValue`/`ClearValue`/
  `SectionExist`/`KeyExist` plus `Save` / `SaveExt` (the latter rewrites the file **preserving original
  layout + comments**, only updating known keys). This is what `axr_main.config` and the plugin LTX
  stores (keybinds, options, debug toggles — F65/F111/F112/F113) read and write through.

## utils_xml — UI XML, color, resolution scaling (741 lines)

Helpers for loading `.xml` UI layouts and adapting them to screen.

- **XML loading:** `get_hud_xml` / `get_Mhud_xml` (load a HUD/menu xml), `get_icons_texture`.
- **Color:** `get_color` / `get_color_con` / `lerp_color`.
- **Resolution (load-bearing for any custom UI):** `correct_ratio`, `is_widescreen`, `screen_ratio`,
  `adjust_pos_to_element` / `adjust_size_to_element`, `move_element`, `sync_pos` / `sync_size` /
  `sync_element` / `sync_cursor`, `align_to_center` — the aspect-ratio correction every Lua UI screen
  applies so layouts don't stretch.
- **Widgets:** `set_msg`, `set_icon` / `set_upgr_icon`, `set_grid_element`, `get_special_txt`,
  `get_name_cond`.
- **Low-level DOM:** the `XmlParser` class (`ToXmlString` / `FromXmlString` / `ParseArgs` /
  `ParseXmlText` / `loadFile`) + `XmlFindNodeWithAttribute` / `XmlFindNextNodeByName` /
  `XmlGetNodeValue` / `XmlGetNodeAttribute` — navigate/edit the XML tree from Lua.

## utils_ui — the Lua inventory-cell engine (3521 lines)

The biggest library (Tronex). It **reimplements inventory cell rendering in Lua** over the engine
`CUI*` classes, which is what gives the actor menu / trade / loot screens their full control. The
backbone of F104 (`flows/actor-menu-inventory-ui.md`).

- **`UICellItem`** — one item's visual: `Add_Icon` / `Add_Shadow` / `Add_Layers` / `Add_ProgressBar`
  (condition bar) / `Add_Counter` (stack) / `Add_Upgrade` / `Add_Attachements` / `Add_CustomText`,
  plus `Colorize` / `Highlight` / `Check_TradeMode` / `GetCost`, and child-stacking (`AddChild` /
  `PopChild` / `CountChilds` for stacked items).
- **`UICellContainer`** — the grid engine: `AddItem` / `RemoveItem` / `TransferItem`, `FindFreeCell` /
  `Grow` / `TakeRoom`, the full mouse/drag/scroll input (`On_Select` / `On_Drag` / `On_Hover` /
  `On_Mouse1` (+ `_DB`) / `On_Mouse2` / `On_Scroll` / `OnKeyboard`), and sorting
  (`GetSortMethod` + the `sort_by_*` comparators). This is the drag-drop inventory grid.
- **`UIInfoItem` / `UIInfoUpgr`** — the tooltip info rows (item description + upgrade list).
- **`UICellProperties`** — the right-click properties/context menu; **`UIHint`** — the hover hint.
- **Stats:** `prepare_stats_table` / `add_stats_table` / `get_stats_value` (+ string/func/xml variants)
  and the weapon-bar functions `prop_accuracry` / `prop_handling` / `prop_damage` / `prop_rpm` /
  `prop_condition` — the character/weapon stat readouts.

## utils_stpk — server-object packet read/write (2046 lines)

Alundaio's library for reading and editing a **server (cse_alife) object's binary `state_data` blob**
from Lua — the only way to inspect/mutate offline object state. Used by debug spawners (F111), save
editing, and any script that changes a server object's stored properties.

- **High-level `get_*_data` / `set_*_data` pairs** for every server class: `object` / `abstract` /
  `squad` / `weapon` / `item` / `ammo` / `inv_box` / `item_document` / `item_pda` / `actor` /
  `stalker` / `monster` / `smart_cover` / `climable` / `heli` / `lamp` / `physic` /
  `space_restrictor` / `anom_zone` / `visual_zone` / `level_changer` / `item_update`. Each decodes the
  packet into a Lua table (and writes it back).
- **`spawn_smart_cover` / `spawn_heli`** — programmatic spawning of objects that need pre-filled packets.
- **Low-level `parse_*` / `fill_*_properties_packet`** — one pair per class in the full `cse_alife_*`
  inheritance chain (abstract → visual → creature → monster/human → stalker/actor/monster, plus the
  item/zone/restrictor/changer hierarchies). They mirror the C++ `STATE_Read`/`STATE_Write` order, so
  they must stay in lockstep with the engine's server-object serialization (`flows/save-load.md`,
  `modlayer/gameplay/server-entities-lua.md`).
- **Packet primitives:** `data_left`, `read_chunk` / `write_chunk`, `readvu8uN` / `writevu8uN`,
  `readvu32stringZ` / `writevu32stringZ`.

---

## Cross-cutting notes

- **No state of their own (mostly):** these are pure helpers. The exceptions are `utils_item`'s cost
  cache and `cfg_file` instances — both explicit and self-managed.
- **`cfg_file` vs engine `ini_file`:** if a script needs to *write* config (persist a setting), it uses
  `utils_data.cfg_file`; engine `ini_file` is read-only. Knowing which is in play explains why some
  configs round-trip layout/comments (`SaveExt`) and others are read-only.
- **`utils_stpk` is fragile by design** — its `parse_/fill_*` order must match the engine's C++ packet
  layout exactly; an engine change to a cse class's `STATE_Read` silently corrupts these unless updated.
- **`utils_obj` cover reservation** — `db.used_level_vertex_ids` is shared global state; `lmove` claims,
  `validate` checks it. Combat schemes that bypass `lmove` can cause cover-stacking.

## Deeper

- Global state registry these read/write → `flows/db-registry.md` (F69, `db.used_level_vertex_ids`,
  `db.storage`).
- Condlist/scheme grammar built on `utils_data.parse_*` → `flows/condlist-library.md`,
  `flows/scheme-engine-core.md`; combat schemes using `utils_obj` movement → `flows/script-npc-actions.md`,
  `flows/axr-behaviour-addons.md` (F113).
- `get_item_cost` consumers → `flows/trade-economy.md` (F101); `UICell*` consumers →
  `flows/actor-menu-inventory-ui.md` (F104); UI XML scaling → `flows/hud-ui-load.md`.
- Server-object serialization `utils_stpk` mirrors → `flows/save-load.md`,
  `modlayer/gameplay/server-entities-lua.md`.
- Layer map → `modlayer/gameplay/utils-db.md` (M22).
