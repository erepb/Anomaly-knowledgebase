---
domain: modlayer/gameplay/trade-economy
tags: [modlayer, gameplay, unpacked, lua, trade, economy, money, map]
sources: [unpacked/scripts/trade_manager.script, unpacked/scripts/axr_trade_manager.script, unpacked/configs/items/trade/]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/trade-economy.md, flows/dialog-trade.md, flows/dialog-system-lua.md, flows/monster-schemes.md, flows/inventory.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Trade & economy — L1 map (M15)

How money and goods move: the actor↔NPC shop (trade profiles + the Lua trade UI), the engine
stock spawner, and the simulated NPC↔NPC gulag-job economy. Mechanics live in
`flows/trade-economy.md` (F101); the engine `CTrade`/`CTradeParameters` machinery is
`flows/dialog-trade.md` (F18).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| trade profile manager | `trade_manager.script` | per-NPC profile assignment, condlist re-pick, restock timer, save/load, the engine-asserted discount functors | F101 ✅ |
| trade profiles | `configs/items/trade/trade_*.ltx` (36) + `presets/trade_presets.ltx` + `presets/trade_discounts.ltx` | `[trader]` condlists; buy/sell lists are *blacklist + per-item multipliers* via preset category inheritance; supplies sections for restock | F101 ✅ |
| Lua trade UI | `ui_inventory.script` + `utils_item.script` (trade section) + `utils_ui.script` + `actor_menu_inventory.script` | intercepts engine `StartTrade`, computes prices and runs the whole transaction in Lua | F101 ✅ |
| NPC↔NPC economy | `axr_trade_manager.script` + `configs/items/trade/gulag_job_trade_buy_sell.ltx` + `configs/beh_gulag_jobs.ltx` | stalkers sell surplus to traders / buy ammo / get weapon upgrades at mechanics via gulag jobs | F101 ✅ |
| eco difficulty knobs | `game_difficulties.script` (`econ_<n>`) | buy/sell/restock/rewards/loots multipliers consumed across the economy | F101 ✅ (trade-relevant keys) |
| trader binding & performance | `bind_trader.script`, `mob_trader.script`, `mob_trade.script` | `script_trader` binder + scheme (anims/sounds, trade open/close switches) | F94 → `flows/monster-schemes.md` |
| engine tier | `src/xrGame/trade*.cpp`, `InventoryOwner.cpp` | `CTradeParameters`, legacy `GetItemPrice`, `CPurchaseList` stock spawn, discount functor asserts | F18 → `flows/dialog-trade.md` |

## How it fits

Scheme configuration (F66) assigns a profile LTX per NPC (`trade =` in smart logic, default
generic) → talking re-picks the profile's condlists and maybe restocks (engine `buy_supplies`) →
the dialog trade row (F99) calls `start_trade` → engine hands off to the Lua UI, which prices
items from the profile + `trade_manager` discounts × eco factors and transfers items/money
itself. In parallel, item pickups flag NPCs as having surplus, and gulag jobs walk them to
traders/mechanics to sell/buy/upgrade.

## Subsystem deep-dives (delegates to flows)

- Profile lifecycle, data-tier semantics, UI takeover + price formula, NPC↔NPC jobs, verified
  quirks (dead `for i = count, 1` sell loop, blacklist-by-empty-value, non-conservative money) →
  `flows/trade-economy.md` (F101) ✅
- Engine phrase graph + `CTrade`/pricing → `flows/dialog-trade.md` (F18)
- Trade dialog rows / `npc_is_trader` → `flows/dialog-system-lua.md` (F99)
- Trader schemes/binder → `flows/monster-schemes.md` (F94)
- Repair/upgrade item mechanics → `flows/item-behaviors-lua.md` (F102) ✅

## Maintenance

- F101 ✅; F102 (item behaviors) ✅; remaining adjacency is F114 (axr managers).
  Re-check `gamedata/scripts/{trade_manager,axr_trade_manager,ui_inventory,utils_item}` for
  overrides on pickup (none at writing).
