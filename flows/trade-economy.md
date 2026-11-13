---
domain: flows/trade-economy
tags: [modlayer, gameplay, unpacked, lua, trade, economy, money, npc, flow]
sources: [unpacked/scripts/trade_manager.script, unpacked/scripts/axr_trade_manager.script, unpacked/scripts/ui_inventory.script, unpacked/scripts/utils_item.script, unpacked/scripts/utils_ui.script, unpacked/scripts/actor_menu_inventory.script, unpacked/scripts/game_difficulties.script, unpacked/scripts/xr_logic.script, unpacked/scripts/xr_motivator.script, unpacked/scripts/bind_trader.script, unpacked/configs/items/trade/trade_generic.ltx, unpacked/configs/items/trade/trade_stalker_sidorovich.ltx, unpacked/configs/items/trade/gulag_job_trade_buy_sell.ltx, unpacked/configs/items/trade/presets/trade_presets.ltx, unpacked/configs/items/trade/presets/trade_discounts.ltx, unpacked/configs/beh_gulag_jobs.ltx, src/xrGame/trade2.cpp, src/xrGame/InventoryOwner.cpp, src/xrGame/UIGameSP.cpp]
related: [flows/dialog-trade.md, flows/inventory.md, flows/dialog-system-lua.md, flows/monster-schemes.md, flows/scheme-engine-core.md, flows/condlist-library.md, flows/ai-squad-logic.md, flows/gameplay-boot.md, flows/actor-menu-inventory-ui.md, flows/axr-managers.md, modlayer/gameplay/trade-economy.md, modlayer/gameplay/axr-addon.md, modlayer/gameplay/dialogs-info.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: trade & economy — Lua tier (F101)

Three economies share the `configs/items/trade/` data tier. (1) **Actor↔NPC trade**: a per-NPC
*trade profile* (an LTX naming buy/sell lists, price exponents and discounts) managed by
`trade_manager.script`, consumed by the **Lua trade UI** (`ui_inventory.script`), which computes
prices and performs the whole transaction in Lua — the engine `CTrade` pricing path (F18,
`flows/dialog-trade.md`) is bypassed for the player-facing shop but still calls back into
`trade_manager` for its own queries. (2) The **engine stock spawner**: `buy_supplies` →
`CPurchaseList::process` materializes the trader's restocked inventory. (3) A **simulated NPC↔NPC
economy** (`axr_trade_manager.script`): off-screen-ish gulag jobs where stalkers walk to a trader
to sell surplus loot and buy ammo, and to a mechanic to buy weapon upgrades.

**Provenance:** no agent — written from direct reading. `trade_manager.script` and
`axr_trade_manager.script` READ-IN-FULL; `utils_item.get_item_cost`/`get_item_trade_status`,
`utils_ui.GetCellCost`, `ui_inventory` `start`/`TMode_InitProfile`/`TMode_Buy`/`TMode_Sell`,
`actor_menu_inventory.CUIActorMenu_OnMode_Trade`, `xr_logic` trade assignment,
`xr_motivator` use-callback, `bind_trader:update`, `game_difficulties.setup_eco_factors`,
engine `CUIGameSP::StartTrade` + `CInventoryOwner::buy_supplies` + the `trade2.cpp` functor
asserts all read directly. LTX shapes verified in `trade_generic.ltx`,
`trade_stalker_sidorovich.ltx`, both presets, `gulag_job_trade_buy_sell.ltx`,
`beh_gulag_jobs.ltx`. **No `gamedata/` override of any of these scripts or of
`configs/items/trade/` (verified by listing).** `mob_trader`/`mob_trade`/`bind_trader` scheme
mechanics are documented in F94 (`flows/monster-schemes.md`), not here.

## Profile lifecycle (`trade_manager.script`)

- **Assignment** — when `xr_logic` configures schemes for an alive `stype_stalker` or
  `stype_trader` NPC it reads `trade =` from the NPC's logic section (the per-smart logic LTX —
  87 `configs/scripts/**` files carry such lines), defaulting to `items\trade\trade_generic.ltx`,
  and calls `trade_manager.trade_init(npc, cfg)`. That just records `tm[id].cfg_ltx` and caches
  the parsed ini in a by-name table; nothing is applied yet. `trade_init` early-returns if
  `db.actor` is nil, and a comment warns it runs once per online switch (relevant to disguise
  mods).
- **Update** — `trade_manager.update(npc)` runs from two places: the `xr_motivator` use callback
  (every time the player talks to a stalker) and every frame of `bind_trader`'s binder update for
  clsid `script_trader` (Sidorovich-style engine traders). Each call re-picks four condlists from
  the profile's `[trader]` section via `xr_logic.parse_condlist`/`pick_section_from_condlist`:
  `buy_condition`, `sell_condition` (section *names* handed to the engine exports
  `npc:buy_condition`/`sell_condition` → `CTradeParameters`), `buy_item_condition_factor`
  (numeric; script fallback 0.7, and overridable game-wide by the saved
  `trader_buy_condition_override` in `alife_storage_manager` state, stored as percent/100) and
  `buy_item_exponent`/`sell_item_exponent` (fallback 0.75).
- **Restock** — after the condition re-pick, `buy_supplies` is applied only if
  `resupply_time` is older than the `restock` eco factor in game-hours (default 24; a
  `force_refresh` argument zeroes it). The picked supplies section goes to the engine export
  `npc:buy_supplies(config, str)` → `CInventoryOwner::buy_supplies` → `CPurchaseList::process`,
  which spawns the stock. Profiles without `buy_supplies` (e.g. `trade_generic.ltx`, the
  ordinary-stalker profile) simply never restock — normal, the early return is commented as such.
- **Persistence** — `save_state`/`load_state` (registered in `on_game_start`) store the whole
  `tm` table in `m_data.trade_manager`: the chosen condition strings and the `resupply_time`
  (converted CTime↔plain-table by local `c2t`/`t2c` helpers at write time), so restock timers
  survive saves. `on_npc_death` (called from `xr_motivator` death/destroy and
  `bind_trader:net_destroy`) drops `tm[id]`.
- **Engine-called discounts** — `get_buy_discount(npc_id)`/`get_sell_discount(npc_id)` are
  resolved *from C++*: `trade2.cpp` `R_ASSERT`s both functors exist when a `CTrade` is
  constructed (F18 notes this hard dependency). They pick the profile's `discounts` condlist
  (note: condlist evaluated with the npc argument nil), read that section's `buy`/`sell` floats
  from the included `presets/trade_discounts.ltx` (buy 0.1–0.4, sell 0.7–1.5 across the shipped
  tiers), and multiply by the `buy`/`sell` eco factors. The same two functions feed the Lua UI's
  price formula, so engine and Lua paths can't disagree on the discount.

## The data tier (`configs/items/trade/`, 36 profiles + presets)

`trade_<faction/role>.ltx`, each `#include`-ing both presets and carrying one `[trader]` section
whose values are all condlists — so a profile can switch lists by info/goodwill (Sidorovich's
`buy_supplies` ladder picks `supplies_1`…`supplies_5` by `actor_goodwill_ge` thresholds, verified).

- **Buy/sell list sections** (e.g. `[trade_generic_buy]:trasher`) inherit *category* sections from
  `presets/trade_presets.ltx` (~30 of them: `[pistols]`, `[ammo]`, `[artefacts]`, …,
  `[unused]`, `[parts]`, `[money]`; role bundles like `[trasher]:unused,money,parts` and
  `[barman]:unused,money,parts,weapons,…`). Semantics — verified in the Lua status/price code,
  *inverted from what you'd guess*: a line **listed with no value is a blacklist entry** (status
  "can't be traded"); a line listed **with a value** applies `value[1]` as a per-item price
  multiplier (Sidorovich: `isg_patch = 3,3`); an **unlisted** item trades at default price. So
  `trasher` means "won't touch technical items, money sections, or weapon parts".
- **Supplies sections** (`[supplies_1]:common_stock` etc.): `item = count, probability` rows
  consumed by the engine `CPurchaseList` at restock.
- **`gulag_job_trade_buy_sell.ltx`**: `[settings]` toggles for the NPC↔NPC economy plus the
  `[buy_sell]` and `[buy_sell_mechanic]` tables —
  `section = keep, consider, restock, sell_mult, buy_mult` (header comment in the file).

## Opening the shop (the Lua UI takeover)

Dialog "let's trade" → `dialogs.npc_is_trader` action calls `npc:start_trade(db.actor)` (F99
documents the dialog side). Engine `CUIGameSP::StartTrade` first tries the Lua functor
`actor_menu_inventory.CUIActorMenu_OnMode_Trade`, which fires the
`ActorMenu_on_before_init_mode("trade", flags, obj)` callback; `ui_inventory` is subscribed,
sets `flags.ret`, and the engine **returns without opening its own `CUIActorMenu`** — the Lua
window (`ui_inventory.start("trade", npc)` → `TMode_Init`) replaces it. The same interception
pattern covers inventory/loot/repair modes — the window itself (bags, context menus, drag-drop,
the monkey-patch surface incl. the `actor_on_item_sell`/`actor_on_item_buy` synthesis around
`TMode_Sell`/`TMode_Buy`) is documented in `flows/actor-menu-inventory-ui.md` (F104). `TMode_InitProfile` then builds two Lua-side profile
tables straight from `trade_manager.get_trade_profile(id, …)` + the two discount functions:
actor-side bags get the *buy* profile (what the NPC buys from you), NPC-side bags the *sell*
profile.

- **Price** (`utils_item.get_item_cost`): `cost = base section cost × per-item multiplier from
  the profile list × discount`, then prorated for multiuse items (`remaining/max uses`) and ammo
  (`count/box`), else scaled by condition: `× ((cond·0.9 + 0.1) ^ exponent)`. Every branch offers
  the `on_get_item_cost` callback so mods can replace the number.
- **Tradability** (`utils_item.get_item_trade_status`): nil no object; 1 blacklisted /
  `quest_item` / `can_trade = false`; 2 condition below `buy_item_condition_factor` (ammo and
  multiuse exempt); 3 NPC-side hidden (any non-4 status, zero cost, partial mags/uses, `anim` or
  `fake_ammo` items); 4 tradable. Only status-4 cells count toward the total
  (`utils_ui.GetCellCost`).
- **Transaction** (`TMode_Sell`/`TMode_Buy`): sums the staged cells, checks the buyer's money,
  then does plain `transfer_item` + `give_money` on both parties — *no engine
  `CTrade::TransferItem`*. The engine `GetItemPrice` formula in F18 therefore describes a path
  the shipped game UI no longer exercises for the player shop.

## The simulated NPC↔NPC economy (`axr_trade_manager.script`)

Alundaio/Tronex layer, driven entirely by the callback bus + gulag jobs (F73 dynamic jobs in
`beh_gulag_jobs.ltx`, plus hand-placed smart logic):

1. `npc_on_item_take` → if the picked-up section is in `[buy_sell]` (and not the NPC's best/
   equipped weapon, and the NPC's community isn't excluded — zombied/army/monolith/greh/isg),
   count it; reaching the *consider* threshold sets `st.has_items_to_sell` (mirror:
   `[buy_sell_mechanic]` + a working best weapon → `st.has_tech_items`).
2. The flags are read by `xr_conditions.npc_has_items_to_sell`/`npc_has_tech_items` (registered
   by this script *into* `xr_conditions` at load) inside job condlists, steering the NPC into a
   walker job that ends at the trader/mechanic; `beh_gulag_jobs.ltx` fires
   `%=trade_job_sell_items%` on the `trade` waypoint signal. The trader's/mechanic's own job runs
   `trade_job_give_id`/`tech_job_give_id` to stamp its object id onto the job, so the seller
   knows who to hand items to. A 20-second `job.idle` throttle prevents re-triggering.
3. **Sell**: items transfer to the trader and the NPC is paid `base cost × sell_mult`
   (`give_money` — money is *created*, not moved); if the trader object is missing the item is
   `alife_release`d (destroyed). `items_sold` NPC statistic increments.
4. **Buy**: with remaining money the NPC buys only ammo sections up to the *restock* count —
   the gate is `IsItem("ammo", sec) or bw_ammos[sec]`, so any listed ammo qualifies (the
   best-weapon clause is redundant, despite the comment claiming ammo-for-best-weapon-only) —
   via `alife_create_item` + `transfer_money` to the trader. Dynamic news reports the purchase.
5. **Upgrade**: at the mechanic, after selling, the NPC randomly picks up to
   `max_upgrades_per_visit` (5) affordable upgrades for its best weapon while money > 500
   (respecting installed/opposite-branch constraints), pays, and installs them through
   `inventory_upgrades` with `force_upgrade` set; money is floored to `random(100, 200)` so the
   NPC never goes broke. Dynamic news again.

## Eco difficulty factors (`game_difficulties.script`)

`setup_eco_factors` loads an `econ_<n>` profile (player-overridable per key, persisted in
`alife_storage_manager` state): `buy`/`sell` multiply the trader discounts, `restock` sets the
resupply period, and the rest (`rewards`, `repair`, `upgrade`, `loots`, `money_loots`,
`stash_chance`, `weapon_degradation`, …) parameterize the wider economy. `get_eco_factor(key)` is
the read API used by `trade_manager` and many item/loot scripts.

## Isolation ledger (the seams)

- **Engine→Lua**: `trade2.cpp` → `trade_manager.get_buy_discount`/`get_sell_discount`
  (`R_ASSERT`ed); `CUIGameSP::StartTrade` → `actor_menu_inventory.CUIActorMenu_OnMode_Trade`.
- **Lua→engine exports**: `npc:buy_condition`/`sell_condition`/`buy_item_condition_factor`/
  `buy_item_exponent`/`sell_item_exponent` (→ `CTradeParameters`), `npc:buy_supplies`
  (→ `CPurchaseList`), `transfer_item`/`transfer_money`/`give_money`, `obj:cost()`/`condition()`.
- **Callback bus (F1/F65)**: `npc_on_item_take` (axr intake), `ActorMenu_on_before_init_mode`
  (UI takeover), `ActorMenu_on_trade_started`/`closed` (`mob_trade`, F94), `on_get_item_cost`
  (price mod hook), `save_state`/`load_state` (persistence).
- **Scheme engine (F66/F73)**: profile assignment inside `xr_logic` scheme configuration; the
  axr conditions/effects injected into `xr_conditions`/`xr_effects` for job condlists.
- **Adjacent flows**: engine pricing + dialog machinery F18; the trade dialog row F99; trader
  binder/schemes F94; squads/jobs F12/F73; upgrades data `flows/inventory.md` (F8 family).

## Threading

All Lua, main game thread. The "NPC economy" is not a background simulation — it only runs for
online NPCs via callbacks and job logic; offline NPCs neither sell nor restock (trader restock
timers only advance against game time and are checked on talk/update).

## Failure & edge cases (verified unless noted)

- **Trader/stash drop events for antifreeze-pending items (2026.11.09)** — `CAI_Trader` and
  `CInventoryBox` ignore drop events for items spawn antifreeze has not spawned yet
  (`ai_trader.cpp`, `InventoryBox.cpp`). Sidorovich/Forester binders also fire
  `sid_forester_on_item_take` / `sid_forester_on_item_drop` (`callbacks_gameobject.script`).

- **Dead sell loop (latent bug)**: both sell loops iterate `for i = count, 1 do` *without* a
  `-1` step — Lua's default step is +1, so the body runs only when `count == 1`. An NPC that
  accumulated 2+ surplus items of one section sells *none* of them that visit, yet the counters
  are cleared (`empty_table`) afterward, so the surplus is forgotten. The keep-in-stock `break`
  on param 1 is mostly dead code for the same reason. Practical effect: NPC→trader sales happen
  in ones, and over-accumulated stacks silently stop selling.
- **Items destroyed without a trader**: if the seller object can't be resolved at sell time the
  goods are `alife_release`d and the money is conjured by `give_money` — the "economy" is not
  conservative in either direction.
- **Restock is per-talk, not per-frame** for ordinary stalkers: a stalker's profile is only
  re-applied when used (talked to); engine `script_trader`s re-check every binder update.
- **Blacklist-by-empty-value**: an item listed in a buy/sell section *with no value* is
  untradeable; deleting the value (not the line) is how presets ban categories. Misreading this
  as a whitelist is the classic modding error here.
- **Wrong-condition goods hidden, not refused**: NPC-side items below full mags/uses or with
  zero cost get status 3 and never render; actor-side too-damaged items render with a "too
  damaged" note (status 2).
- **Engine assert**: deleting/renaming `trade_manager.get_buy_discount`/`get_sell_discount`
  crashes at `CTrade` construction (`R_ASSERT`, F18).
- **`trade_init` before actor**: returns silently if `db.actor` is nil; the NPC then has no
  profile until the next scheme configuration.

## Deeper

- Profile manager: `unpacked/scripts/trade_manager.script` (all of it — it's 239 lines).
- NPC economy: `unpacked/scripts/axr_trade_manager.script` (`check_trade_item`,
  `npc_trade_buy_sell`, `npc_tech_upgrade_sell`).
- Pricing: `unpacked/scripts/utils_item.script` (trade section), `unpacked/scripts/utils_ui.script`
  (`UICellContainer:GetCellCost`), `unpacked/scripts/ui_inventory.script` (`TMode_*`).
- Data: `unpacked/configs/items/trade/` (profiles, `presets/trade_presets.ltx`,
  `presets/trade_discounts.ltx`, `gulag_job_trade_buy_sell.ltx`).
- Engine seams: `src/xrGame/trade2.cpp` (functor asserts), `src/xrGame/InventoryOwner.cpp`
  (`buy_supplies`), `src/xrGame/UIGameSP.cpp` (`StartTrade` interception).

## Maintenance

- Re-verify `gamedata/scripts/` for overrides of `trade_manager`/`axr_trade_manager`/
  `ui_inventory`/`utils_item` on pickup (none at writing — but trade UI is a popular mod target).
- The dead `for i = count, 1` loop is upstream Anomaly behavior; if a future merge adds the
  `-1` step, the "sells in ones" edge case above changes materially.
- If repair/upgrade pricing gets its own flow (F102 items), move the mechanic-upgrade detail
  there and leave a pointer.
