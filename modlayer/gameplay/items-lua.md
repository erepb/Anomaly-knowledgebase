---
domain: modlayer/gameplay/items-lua
tags: [modlayer, gameplay, unpacked, lua, items, crafting, map]
sources: [unpacked/scripts/itms_manager.script, unpacked/scripts/item_device.script, gamedata/scripts/item_weapon.script, unpacked/configs/items/settings/]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/item-behaviors-lua.md, flows/inventory.md, flows/trade-economy.md, flows/binder-architecture.md, modlayer/gameplay/trade-economy.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Item Lua behaviors — L1 map (M16)

The `item_*` plugin family + the `itms_manager` hub: everything that happens when an item is
used, combined, dragged, repaired, cooked, disassembled or deployed. Mechanics live in
`flows/item-behaviors-lua.md` (F102); the engine inventory machinery (take/place/slots) is
`flows/inventory.md` (F8).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| type registry | `_g.script` (`Parse_ITM`/`IsItem`) | ini-driven `_ITM[type][section]` tables; the family's dispatch backbone | F102 ✅ |
| hub | `itms_manager.script` | use veto (boosts/required tools), hardcoded specials, drag-drop combining, `ItemProcessor` deferred item properties, bolts, rewards | F102 ✅ |
| economy cluster | `item_repair`, `item_parts`, `item_cooking`, `item_recipe` | repair kits, disassembly + part conditions, campfire/fuel cooking, recipe learning | F102 ✅ |
| equipment cluster | `item_device` (**binder**), `item_weapon` (**gamedata fork**), `item_artefact`, `item_knife` | battery drain/NV/torch/PDA, scopes/silencer/ammo-wheel/overheat, artefact degradation+belt HUD, knife library | F102 ✅ |
| deployables & misc | `item_backpack`, `item_tent`, `item_mine` (dormant), `item_map_kit`, `item_radio`, `item_money` | player stash, tent/sleep bag, mines, anomaly maps, RF receiver, money items | F102 ✅ (several *(agent-traced)*) |
| data tier | `configs/items/settings/*.ltx` + per-section `ini_sys` keys | `itms_manager`/`devices`/`parts`/`craft`/`cook`/`arty_container`/`item_rewards`/`backpack_stash` | F102 ✅ |

## How it fits

Engine inventory events (F8) surface on the Lua bus (F65) → `itms_manager` gates the use
(`actor_on_item_before_use`) → the owning plugin dispatches by `IsItem(type, sec)` from the
`Parse_ITM` registry → mechanics consume `utils_item`/`actor_effects` and the
`configs/items/settings/` data tier. Devices alone are bound objects (`bind = item_device.bind`,
F88) with a per-frame binder. Items spawned with non-default condition/uses/ammo ride the hub's
`ItemProcessor`.

## Subsystem deep-dives (delegates to flows)

- Registry, hub, per-plugin catalog, the `item_weapon` gamedata fork, verified quirks
  (`item_id` nil-global family, benign cooking typo, live-but-dormant mines) →
  `flows/item-behaviors-lua.md` (F102) ✅
- Engine inventory (take/place/slots/belt) → `flows/inventory.md` (F8)
- Trade pricing of multiuse/condition items → `flows/trade-economy.md` (F101)
- Binder pattern → `flows/binder-architecture.md` (F88)

## Maintenance

- F102 ✅ — Phase G6 complete. Re-list `gamedata/scripts/item_*` on pickup (only `item_weapon`
  forked at writing). Promote *(agent-traced)* plugin internals when tasks touch them.
