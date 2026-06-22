---
domain: flows/item-behaviors-lua
tags: [modlayer, gameplay, unpacked, lua, items, inventory, crafting, flow]
sources: [unpacked/scripts/itms_manager.script, unpacked/scripts/item_device.script, unpacked/scripts/item_weapon.script, gamedata/scripts/item_weapon.script, unpacked/scripts/item_parts.script, unpacked/scripts/item_cooking.script, unpacked/scripts/item_repair.script, unpacked/scripts/item_artefact.script, unpacked/scripts/item_radio.script, unpacked/scripts/item_recipe.script, unpacked/scripts/item_backpack.script, unpacked/scripts/item_money.script, unpacked/scripts/item_tent.script, unpacked/scripts/item_mine.script, unpacked/scripts/item_map_kit.script, unpacked/scripts/item_knife.script, unpacked/scripts/_g.script, unpacked/configs/items/settings/]
related: [flows/inventory.md, flows/trade-economy.md, flows/binder-architecture.md, flows/gameplay-boot.md, flows/scheme-engine-core.md, flows/characters-info-encyclopedia.md, flows/dialog-system-lua.md, flows/actor-survival-status.md, flows/treasures-secrets.md, modlayer/gameplay/items-lua.md, modlayer/gameplay/trade-economy.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: item Lua behaviors (F102)

What happens when the player *uses, combines, or drags* an item — the `item_*` plugin family plus
the `itms_manager` hub. Architecture: every file is a **self-registering callback plugin** on the
F65 bus (`actor_on_item_use` and friends), dispatching per section through the **`_ITM` item-type
registry** that `_g.Parse_ITM` builds at boot from `ini_sys` keys — so "is this a repair kit?"
is `IsItem("repair", sec)` against a table precomputed from each section's `repair_tool` flag, not
a string match. The one structural exception is `item_device`, which is a real **object binder**
(F88): device sections carry `bind = item_device.bind`, giving them a per-frame `device_binder`
for battery drain. One **live `gamedata/` override** exists: `item_weapon.script` (a "demonized"
fork — see below); everything else runs from `unpacked/`.

**Provenance:** agent-researched, spine personally verified — `_g.Parse_ITM`/`IsItem` registry,
`itms_manager` registrations + `actor_on_item_before_use` (boost gating) + `actor_on_item_use`
(hardcoded-section handlers, `multiuse_r` recharge) + `actor_on_item_take`/`drop` (strapped-item
bookkeeping) + `ItemProcessor` ammo branch, `item_cooking` `OnCook`/`GetAvailFuel`,
`item_parts.disassembly_weapon`, `item_device` binder class + consumption-table build,
`item_knife` whole file, `item_mine` header + registrations, both `item_weapon` function censuses
+ the override's `ammo_aggregation_full`/overheat diff — all read directly. Registration lists of
every file verified by grep. Per-file mechanics of artefact/radio/repair/recipe/backpack/money/
tent/map_kit beyond their trigger surfaces are *(agent-traced)* where marked. Two agent claims
were rejected on verification: the `item_knife` "operator precedence bug" is not a bug (`/` binds
tighter than `-` in Lua; `cond-num/4` is the intended `cond - factor/4`), and the `item_cooking`
`is_campfire` global typo is benign (see Failure cases). **Only `item_weapon.script` has a
gamedata override (verified by listing).**

## The type registry (`_g.Parse_ITM`)

At boot `Parse_ITM` scans every `ini_sys` section and fills `_ITM[type][section]` from per-section
keys: `repair` ← `repair_tool` (value = its `repair_only` list), `cook` ← `cooking_tool`,
`map` ← `use_map`, `recipe` ← `recipe`, `tool` ← kind `i_tool`/`i_kit`, `multiuse` ← `max_uses`,
`device` ← `bind = item_device.bind`, and many more. `IsItem(type, sec)` / `GetItemList(type)`
are the read API used by every plugin's dispatch — *adding a new repair kit is a config change,
not a code change*. Conditions/values support string/bool/float/list reads with defaults.

## The hub (`itms_manager.script`)

Registers the central handlers on `_g.start_game_callback` (F65):

- **`actor_on_item_before_use`** — *can* this be consumed: skips `tool` items; reads the
  `boost_time`/`boost_*_restore` keys and refuses if a currently-running booster of the same kind
  is stronger; blocks anabiotics in combat; enforces `required_tool` (e.g. a can opener) — the
  veto travels back through the callback's `flags.ret_value`.
- **`actor_on_item_use`** — hardcoded-section specials (Strelok's note → `txr_routes.open_route`,
  deployable mgun, hand watch, chocolate, `bolts_pack` → 20–30 random bolts via
  `utils_item.delay_event`, empty-syringe returns, weather radar) plus the generic `multiuse_r`
  rule: rechargeable multiuse items get one use restored on use.
- **`ActorMenu_on_item_drag_drop`** — the combine engine: artefact dragged onto a container makes
  the `<artefact>_<container>` section (lists from `items\settings\arty_container.ltx`); generic
  pair combos come from `items\settings\craft.ltx` `item_combination` (`sec1:sec2 → result`),
  consuming both inputs.
- **`ItemProcessor`** (lazy singleton class) — deferred property application for spawned items:
  queues condition/uses/ammo-count writes per object id and applies them once the game object
  exists (`alife_create_item(sec, npc, {ammo=n, cond=c, uses=u})` rides this). Used everywhere
  items are spawned with non-default state (F101's NPC economy, loot, disassembly).
- Also: bolt-count persistence (`m_data.bolts`, capped by `items\settings\itms_manager.ltx`
  `bolt_max_num`), item pickup sounds, a weighted reward-item selector
  (`items\settings\item_rewards.ltx`), new-game equipment damage, focus-highlighting of
  compatible items (ammo/scopes/parts/kits) in the inventory UI.

## The plugin catalog (trigger surfaces verified; details as marked)

| Plugin | Trigger | Mechanic |
|--------|---------|----------|
| `item_repair` | `actor_on_item_use` on `IsItem("repair")` | repair-kit UI: targets filtered by the kit's `repair_only` list + `repair_min/max_condition` window; new condition = old + `repair_add_condition` + per-part bonus (`repair_part_bonus`, achievement-boosted); degrades/discharges the kit *(math agent-traced)* |
| `item_parts` | drag tool onto item, `actor_on_item_take` | disassembly (verified): attachments at status 2 collected, mag force-unloaded, each part rolls `random(100) <= disassembly_chance + condition*100`, spare parts at half chance, part conditions persisted per item via `se_save_var(... "parts")` and re-distributed when item condition drops |
| `item_cooking` | `actor_on_item_use` on `IsItem("cook")` | cooking UI (verified): campfire mode via `bind_campfire.get_nearby_campfire` or fuel-tier mode from `items\settings\cook.ltx` `[fuel]`/`[meal_list]`; recipes gated by encyclopedia notes (F100); ingredients counted in uses; fuel discharged only in non-campfire mode |
| `item_recipe` | `actor_on_item_use` on `IsItem("recipe")` | reading a recipe unlocks its encyclopedia note via the `actor_on_interaction("notes", …)` bus (F100) and shows a craft-requirements viewer over `craft.ltx` *(UI agent-traced)* |
| `item_artefact` | `actor_on_before_hit` (only when the `arty_degradation` eco flag is on, toggled live by `game_difficulties`), belt/ruck-move callbacks | artefact condition loss per hit type (immunity-section mapping), ruck-carried artefact radiation leak, and the belt-artefact HUD *(internals agent-traced)* |
| `item_device` | **binder** (`bind = item_device.bind`) + key/drag/trade callbacks | `device_binder:update` → `process_power`/`process_torch`/`process_glitch`: stage-based battery drain from `items\settings\devices.ltx` `[power_consumption]` (csv per stage) + `[power_consumption_event]` (event drains, e.g. geiger on `actor_on_feeling_anomaly`); battery drag-onto-device recharge; NV/torch toggles; surge/psi PDA glitch; `m_data.device_condition` persistence |
| `item_weapon` | drag-drop (scope attach), key release (silencer toggle), ammo wheel UI | scope attach/detach against `[addons_table]`, TKGP silencer toggle, Tronex ammo wheel, weapon overheat; **overridden in gamedata** (below) |
| `item_money` | `actor_on_item_use`/`take` | money *items*: corpse money → denomination items by rank tables × eco factor; wallet denominations; auto-absorb on pickup *(agent-traced)* |
| `item_backpack` | `actor_on_item_use`, `actor_on_item_take_from_box` | deployable player stash: spawns `inv_backpack` invbox + map spot, name UI, `m_data.player_created_stashes`; emptying it returns the backpack and fires `actor_on_stash_remove` *(agent-traced)* |
| `item_tent` | `actor_on_item_use`, `physic_object_on_use_callback` | placeable tent/sleep bag (`ph_tent`/`ph_sleepbag` physics objects, F93 adjacency); pack-up by proximity; settings from `itms_manager.ltx` *(agent-traced)* |
| `item_map_kit` | `actor_on_item_use` on `IsItem("map")` | anomaly-map viewer; first use unlocks `txr_routes` guide routes for the level *(agent-traced)* |
| `item_radio` | key hold/release, `actor_on_update` | RF receiver: frequency tuning, stash/artefact/psy-source detection by clsid/section database, 3D digit HUD, `m_data.RF_*` persistence *(agent-traced)* |
| `item_knife` | **none** — pure library | per-knife degradation factors + `degradate()` (quarter-factor per hit) + `can_loot(monster)` tier gates for butchering; callers live elsewhere (verified whole file) |
| `item_mine` | `actor_on_item_use` + npc/monster/actor update | placeable mines/IEDs with priming timer, proximity blow, witness-turns-hostile; **header says `-- UNUSED` but the callbacks DO register** — `axr_main` auto-runs any script with `on_game_start`, so the code is live and only the items are unobtainable |

## The `item_weapon` gamedata override (the one live fork)

`gamedata/scripts/item_weapon.script` shadows the unpacked one (393 changed diff lines, verified
by diff). Same scope/silencer/ammo-wheel surface, two real changes ("demonized" comments):

1. **Ammo aggregation rewritten** — base `ammo_aggregation(obj, npc)` (per-picked-box sorting,
   damaged-ammo replacement) becomes a thin wrapper that batches via a next-frame
   `CreateTimeEvent` into the new `ammo_aggregation_full(npc_id)`, which re-sorts **all** of the
   owner's alife ammo objects in one pass (tolerates being handed objects instead of ids).
2. **Overheat reworked** — per-weapon smoke tracking (`add_weapon`/`remove_weapon_by_id`/
   `npc_on_update` — NPC weapons participate too) replacing the actor-active-item-only logic, and
   `get_object_by_id` wrappers replacing raw `level.object_by_id`.

Any KB statement about ammo aggregation or overheat must be checked against the *gamedata* file.

## Isolation ledger (the seams)

- **Callback bus (F65)**: consumes `actor_on_item_use`/`before_use`/`take`/`drop`/
  `to_belt`/`to_ruck`/`to_slot`, `ActorMenu_on_item_drag_drop`/`focus_receive`,
  `actor_on_feeling_anomaly`, key events, `save_state`/`load_state`; sends
  `actor_on_stash_create`/`remove`, `actor_on_frequency_change`, `on_get_item_cost` consumers
  overlap with F101.
- **Type registry**: `_g.Parse_ITM`/`IsItem`/`GetItemList` (boot, F65).
- **Binder tier (F88)**: `item_device.bind` (`device_binder`); tent/sleepbag spawn `ph_*`
  physics objects (F93).
- **Shared helpers**: `utils_item` (`discharge`/`degrade`/`delay_event`/`get_upgrades_*`),
  `utils_stpk` (netpacket item data), `actor_effects` (use animations/FX), `ui_item` (names).
- **Adjacent systems**: encyclopedia notes gate cooking/recipes (F100); eco factors
  (`game_difficulties`) scale degradation/loot money (F101); achievements/statistics increment
  on disassembly/repair; `treasure_manager` feeds the radio's stash list.
- **Data tier**: `configs/items/settings/` — `itms_manager.ltx`, `devices.ltx`, `parts.ltx`,
  `craft.ltx`, `cook.ltx`, `arty_container.ltx`, `item_rewards.ltx`, `backpack_stash.ltx` +
  per-section `ini_sys` keys (`boost_*`, `repair_*`, `cooking_*`, `max_uses`, `use_map`, …).

## Threading

All Lua, main game thread, callback-driven; the only per-frame work is `device_binder:update`
(bound devices), the radio scan (5s cadence), and the override's overheat/smoke updates.

## Failure & edge cases (verified unless noted)

- **`item_id` nil-global family (latent bug)**: `se_load_var(item_id, …, "strapped_item")` with
  an undefined `item_id` appears in `itms_manager.actor_on_item_take`, twice in
  `axr_companions`, and in `death_manager` — all four read `nil` and the branches never fire.
  Only `ui_inventory` passes the real id. Net effects: the strapped-weapon flag set on drop is
  never cleared on pickup, and `death_manager`'s "don't modify weapons strapped from the player"
  guard is dead (it always proceeds).
- **`item_cooking` global typo, benign**: `OnCook` tests the global `is_campfire` (always nil →
  always "not campfire") instead of `self.is_campfire`, but in campfire mode `GetAvailFuel(tier,
  true)` returns `false`, so no fuel is found to discharge — the bug is masked downstream.
  Touching `GetAvailFuel`'s campfire branch would un-mask it.
- **`ItemProcessor` debug-only nil**: the ammo branch's success printf references `new_con`
  (copy-paste from the condition branch); only evaluated with `self.Debug` on.
- **`item_mine` is live-but-dormant**: the `-- UNUSED` header does not stop registration (any
  script with `on_game_start` is auto-run); spawning its sections by console/mod re-activates
  the whole mechanic.
- **Booster veto path**: a use can be silently refused by `actor_on_item_before_use` (stronger
  booster active, missing `required_tool`, anabiotic in combat) — mods adding consumables must
  set the `boost_*` keys or the gate mis-compares.
- **Override drift**: only `item_weapon` is forked; diffs against `unpacked/` are meaningless
  for ammo aggregation/overheat — read `gamedata/` first (METHODOLOGY rule).

## Deeper

- Hub: `unpacked/scripts/itms_manager.script` (handlers up front; `ItemProcessor` class in the
  back half).
- Type registry: `unpacked/scripts/_g.script` (`Parse_ITM`, `IsItem`, `GetItemList`).
- Device binder: `unpacked/scripts/item_device.script` (`device_binder`, consumption tables).
- Disassembly: `unpacked/scripts/item_parts.script` (`disassembly_weapon`, `get_parts_con`).
- The fork: `gamedata/scripts/item_weapon.script` (`ammo_aggregation_full`, overheat block).
- Data: `unpacked/configs/items/settings/`.

## Maintenance

- `gamedata/item_weapon.script` is the family's only override **at writing** — re-list
  `gamedata/scripts/item_*` on pickup; item plugins are a prime mod target.
- The *(agent-traced)* internals (repair math, radio scan, artefact HUD, money tables) are good
  faith but unverified — promote on first task that touches them.
- F103 landed (`flows/actor-survival-status.md`): the actor-side half of the use pipeline lives
  there — `actor_effects` use-animation scheduler (the engine room behind `play_item_fx` /
  `play_continuous_effect` this family calls), plus the `eat_sleepiness`/`eat_thirstiness` meter
  effects and the arszi_psy anabiotic reset.
