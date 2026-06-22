---
domain: flows/actor-menu-inventory-ui
tags: [modlayer, gameplay, unpacked, lua, ui, inventory, actor-menu, context-menu, flow]
sources: [unpacked/scripts/ui_inventory.script, unpacked/scripts/actor_menu.script, unpacked/scripts/actor_menu_inventory.script, unpacked/scripts/utils_ui.script, unpacked/scripts/aaaa_script_fixes_mp.script, gamedata/scripts/callbacks_gameobject.script, src/xrGame/UIGameSP.cpp, src/xrGame/UIGameCustom.cpp, src/xrGame/ui/UIActorMenu_script.cpp, src/xrGame/Inventory.cpp]
related: [flows/inventory.md, flows/trade-economy.md, flows/item-behaviors-lua.md, flows/actor-survival-status.md, flows/upgrade-system.md, flows/lua-callback-lifecycle.md, flows/hud-ui-load.md, flows/gameplay-ui-screens.md, flows/utils-libraries.md, modlayer/gameplay/actor-survival.md, modlayer/gameplay/items-lua.md, modlayer/gameplay/trade-economy.md, modlayer/gameplay/ui-scripts.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: actor menu & inventory UI (F104)

How Anomaly replaced the engine's `CUIActorMenu` with a Lua window: the functor seam
(`actor_menu_inventory`), the mode mirror (`actor_menu`), the `UIInventory` script window
(`ui_inventory`), and the `utils_ui` cell framework underneath it. F101 documented the *trade*
half of this window; this flow is the window itself — lifecycle, bags, context menus, drag-drop,
loot/repair modes, and the monkey-patch surface mods (and the modded exes themselves) hook.
**Override status (checked this session):** none of the four core scripts is forked in
`gamedata/`; the two patch files that hook the window (`aaaa_script_fixes_mp`,
`callbacks_gameobject`) *are* forked, but their UIInventory-relevant patches are identical in
both copies (the forks' additions belong to other domains).

**Provenance:** agent-researched draft, spine personally re-verified: the whole entry chain
(header/enable_feature, `on_game_start` subscription, `start`, `overrides`, `change_last_mode`,
`hide_*`), `actor_menu.script` and `actor_menu_inventory.script` READ-IN-FULL, the engine callers
grep-read in C++, the `self.properties` table + `InitProperties` body, `Name_/Mode_/Cont_/DB_/
Action_Custom`, the helmet-stat block, `LMode_TransferInfo`, quick-slot read/write,
`npc_on_use`/`physic_object_on_use` auto-open, `Picker_Refresh`/`On_CC_Hover` heads, the
stats-table guards in `utils_ui`, and every monkey-patch site. Bag parsing, mode bodies,
drag-drop branch list, RMode delegation, and the UICellItem/UICellContainer anatomy are
*(agent-traced with line evidence, structure-confirmed)*. **Two agent claims corrected:**
`utils_ui.add_stats_table` is not a usable public extension point (see quirks), and external
callers of `ui_inventory` do exist (the monkey-patch surface below).

## The takeover seam — engine → functor → veto

`actor_menu_inventory.script` is a pure functor surface the engine calls directly (verified in
C++): `UIGameCustom.cpp` calls `CUIActorMenu_OnMode_Inventory` (and `OnHideActorMenu`),
`UIGameSP.cpp` calls `OnMode_Trade` / `OnMode_Upgrade` / `OnMode_DeadBodySearch`, and
`UIActorMenu_action.cpp` calls `OnItemDropped` (with the `EDDListType` source/target enum) plus
focus receive/lost. Each mode functor broadcasts `ActorMenu_on_before_init_mode(mode, flags, obj)`
on the F65 bus and returns `flags.ret` — **returning true tells the engine to skip its own
ActorMenu**. Two always-true policy functors (`CInventory_ItemAvailableToTrade`,
`CUIActorMenu_DonateCurrentItem`) are consulted from `Inventory.cpp`. A fourth functor the engine
R_ASSERTs — `CInventory_BeltWidth` — is **not defined in this file at all**: `callbacks_gameobject`
monkey-patches it into the namespace at load, wrapping it in an `on_belt_size_get` script callback
(same pattern for artefact count). Mode names at this seam: `inventory`, `loot`, `trade`,
`repair`.

`ui_inventory` subscribes (gated on a hardwired `enable_feature = true`) and answers every mode
with `start(mode, obj)` + `flags.ret = true` — the engine ActorMenu still exists in the binary but
never opens in the shipped game. `start` lazily constructs the singleton `GUI = UIInventory()`,
dispatches `IMode_Init` / `LMode_Init` / `TMode_Init` / `RMode_Init`, shows the dialog and
registers it in the script GUI registry (`Register_UI`). Quirk: holding shift while using a corpse
(not an inventory box) makes `start` return early — the veto still fires, so *neither* UI opens.
`overrides()` completes the takeover by replacing `_G.hide_hud_inventory`, the engine's
hide-functor, `_G.GetActorMenu` (now returns the Lua window), and `ui_companion_inv.start`
(companion loot reuses this window's loot mode).

## actor_menu — the mode mirror and the HUD message API

The engine R_ASSERTs `actor_menu.actor_menu_mode` (`UIActorMenu_script.cpp`) and reports mode
ints: 0 closed, 1 inventory, 2 trade, 3 upgrade, 4 dead-body search, 10/11 talk show/hide. The
script mirrors them into `last_mode`, fires `ActorMenu_on_mode_changed` (plus
`ActorMenu_on_trade_started`/`_closed`), and toggles the info portions condlists key off:
`inventory_wnd_open`, `trade_wnd_open`, `upgrade_wnd_open`, `body_search_wnd_open`,
`dialog_wnd_open` (aggregated by `inventory_opened()`; `is_hud_free()` additionally checks the
PDA and the script GUI registry). Because the engine window never opens, **the Lua UI feeds this
mirror itself**: `ui_inventory.change_last_mode` calls `actor_menu.actor_menu_mode` with the
matching int (loot→4, trade→2, repair→3), so every consumer of the mode/infos keeps working.
The `xr_meet_*_closed` flags it also sets are **dead state** — written here, read nowhere
(grep-verified).

The file's second half is the corpus-wide HUD message API used by dozens of scripts: `set_msg`
(timed text on a named custom static), `set_notification` (icon variant), `set_item_news`
(formatted news tips) and `set_fade_msg` — a queued fade-in/out pipeline driven by a
`CreateTimeEvent` tick that pauses while dialog is open and renders, of all things, onto the
`not_enough_money_mine` custom static.

## The window — bags, parsing, slots

`UIInventory` is a `CUIScriptWnd` built from `ui_inventory.xml` (the cell framework reads
`utils.xml`). Inventory real estate is a set of named `UICellContainer` bags mirroring the
engine's `EDDListType`: `actor_bag` (ruck), `actor_equ` (the 13 equipment slots, manually
placed cells), `actor_belt` (artefact belt, blockers shown past the outfit's
`get_artefact_count`), `actor_quick` (4 quick-use slots), `npc_bag` (corpse/box/companion), the
four trade offer/stock bags (F101), and the `picker` popup. `ParseInventory` fills them via
`iterate_ruck` / `iterate_inventory` / `iterate_inventory_box` with kind filters;
`ParseInventory_Companion` restricts a living companion's loot view to items assigned through
`axr_companions` plus strapped weapons *(agent-traced)*. Equipment changes show
slot/helmet/backpack blockers when the worn outfit forbids them.

**Quick slots are console state, not inventory state:** cells render whatever sections
`slot_0`…`slot_3` console variables hold (greyed out when the actor doesn't own one), and
drag-dropping a consumable executes `slot_N <section>` — persistence and key handling are
engine-side.

## The context menu — a declarative, LTX-extensible framework

`self.properties` is a declarative table of ~25 rows; each row has an `index` (menu order), a
label (`name` string or `name_func` method ref), visibility filters (`mode`/`cont` static sets or
`mode_func`/`cont_func` methods), chained `precondition1..N` method refs, and an `action` method
ref. Right-click (`On_CC_Mouse2`) → `InitProperties` walks rows in index order, filters by
bag/mode, AND-chains preconditions, collects label+action pairs and pops the menu. The naming
convention is the API: `Cond_*` gates (use/equip/move/attach/detach/unload/quest), `Name_*`
labels, `Action_*` executors (which call the engine — `move_to_slot`, `attach_addon`, transfers —
then mark the UI dirty). Built-ins cover use, attach ×3, equip/unequip, move/move-all, donate,
unload, detach silencer/scope/GL, drop/drop-all.

**The modding surface is the ten `custom_N` rows**, resolved per item section from ini_sys at
menu-open time: `useN_functor` (module.function returning the label — it doubles as the
precondition: returning nil/false hides the row), `useN_action_functor` (executed on click),
`useN_modes` (default: inventory only), `useN_containers` (default: actor bags),
`useN_allow_db` (also fire on double-click). This is how F102's repair kits, cooking, recipes
etc. put "Use / Repair with…" rows on items without touching this file — config keys, not code.

## Drag-drop, mouse, picker

`On_CC_DragDrop` branches by target: trash, ruck→slot equip, artefact→belt, consumable→quick slot
(console exec), dropped-onto-item attachment (silencer/scope/GL via the same Cond_/Action_Attach
pair `itms_manager` drag-combos don't handle — the *combine* path is the
`ActorMenu_on_item_drag_drop` callback consumed by `itms_manager`, F102), and bag↔bag transfers
(ctrl = whole stack) *(agent-traced branch list, heads verified)*. Double-click uses the same
custom-row `DB_Custom` flag; hover drives highlight + the stats tooltip. The **picker**
(`enable_item_picker`) is a popup sub-container that explodes a stacked cell into its individual
instances (each with condition/ammo sorting via `sort_by_props`) so the player can act on a
specific copy.

## Modes

- **IMode** — plain actor inventory.
- **LMode (loot)** — three flavors keyed at init: dead stalker (full `iterate_inventory`; also
  `LMode_TransferInfo`, verified: reads the corpse's `custom_data` → `[logic] known_info` section
  and moves each info portion the corpse holds onto the actor — dialog-relevant intel transfers
  on body search), inventory box (`iterate_inventory_box`, items may spawn post-open), companion
  (assigned-items filter). `TakeAll`/`PutAll` run every item through the same `Cond_Move` /
  `Action_Move` gates; `PutAll` pre-checks the receiving NPC's carry weight for living stalkers.
  Loot auto-opens from the bus: `npc_on_use` (dead stalker) and `physic_object_on_use_callback`
  (invbox — with a nil-`who` guard, "sometimes who isn't passed").
- **TMode (trade)** — pricing, profile, transaction: all in `flows/trade-economy.md` (F101).
- **RMode (repair/upgrade at mechanics)** — pure orchestration, zero local logic: eligibility,
  affordability, question text and effects all delegate to `inventory_upgrades.*`, tree shape to
  `utils_item.get_upgrades_tree`/`get_upgrades_installed` (`flows/upgrade-system.md` F26 owns the
  mechanics). Repair completion sets condition to 0.99999 *(agent-traced)*.

## Staying in sync — the callback mirror

The window registers *itself* (`RegisterScriptCallback(name, self)` — object subscription, method
dispatch) for a dozen bus events: actor item to ruck/slot/belt, drop, use, box put/take, NPC
take/drop, use events and `actor_on_net_destroy` (closes and nils the singleton so death rebuilds
it). These keep cells live while the engine moves items underneath the UI; the slot callbacks also
do equipment sanity (unequip artefacts when the outfit leaves, helmet/backpack when the new outfit
forbids them) *(agent-traced)*.

## utils_ui — the cell framework under everything

`UICellItem` is the drawable item cell: icon + offset shadow, conditional overlay layers, a
progress bar (condition), counters (ammo/uses), upgrade mark, attached scope/silencer/GL icon
overlays, custom text, and a `childs` table for stacking. `UICellContainer` is the grid: occupancy
matrix, id→index and section→index lookups, auto-placement (`AddItem`) vs manual placement
(`AddItemManual`, used by slots/belt/quick), and pluggable sort (`sizekind` default; kind order
from the ini_sys `item_kind_order` section). The hover stats tooltip pulls from a module-level
`stats_table` with five groups (weapon/outfit/artefact/booster/backpack) of declarative stat rows
(type, icon, magnitude, track-bar vs text, compare flag, optional `value_functor`) — built once by
`prepare_stats_table` *(anatomy agent-traced; guards verified)*.

## The monkey-patch surface (verified)

Because `UIInventory` is a global class table in a global namespace, the shipped game itself
patches it at runtime — the pattern mods should copy:

- `callbacks_gameobject` (live gamedata fork, identical in this region) wraps `TMode_Sell`,
  `TMode_Buy` and `On_Item_Exchange` with flag-passing closures to synthesize the modded-exe
  callbacks **`actor_on_item_sell` / `actor_on_item_buy`** (with computed price) — those events
  exist only via this wrapper. It also injects `CInventory_BeltWidth` into `actor_menu_inventory`.
- `aaaa_script_fixes_mp` (forked, patch identical) wraps `Action_Move` with a nil-item guard,
  version-gated to ≤1.5.2.
- `item_weapon` checks `ui_inventory.is_enabled()`; `utils_ui` reads `ui_inventory.GUI.mode`
  directly (a circular peek — the framework inspects its host).

## Verified quirks & failure cases

- `On_CC_Trasfer` — the missing "n" is load-bearing; `UICellContainer` invokes the callback by
  this exact name.
- `utils_ui.add_stats_table` looks like an extension point but is **unusable**: it and
  `prepare_stats_table` share the same "return if stats_table non-empty" guard, so calling it
  before preparation suppresses *all* default stats, and after preparation it's a no-op. No
  caller exists in the corpus. Mods that extend tooltips write `utils_ui.stats_table[...]`
  directly after preparation.
- The helmet stat block: on a failed `cast_Helmet` it logs using `outfit:name()` (wrong variable
  — `outfit` is the *outfit* local from the previous block) and then continues into
  `c_helm:GetDefHitTypeProtection` calls without bailing — a crash if the cast ever failed (in
  practice slot 12 always holds helmets).
- `quick_item` in the drag-drop quick-slot branch is an accidental global (the `UpdateQuick`
  twin is a proper local).
- `actor_menu.xr_meet_dialog_closed` / `xr_meet_trade_closed` / `xr_meet_upgrade_closed` are
  written but never read anywhere — vestigial state.
- `UICellItem:Add_Shadow` calls `Show(true)` twice in a row (harmless).
- Shift+use on a corpse opens nothing at all (Lua UI skips, engine menu still vetoed).

## Deeper

- Engine inventory machinery (slots/ruck/belt rules the actions ultimately hit) → `flows/inventory.md` (F8).
- Trade mode internals, pricing, the F101 takeover history → `flows/trade-economy.md`.
- The drag-combine consumer and the `useN_*` row providers → `flows/item-behaviors-lua.md` (F102).
- Upgrade/repair mechanics behind RMode → `flows/upgrade-system.md` (F26).
- Domain routing → `modlayer/gameplay/actor-survival.md` (M17 hosts this flow's domain).
