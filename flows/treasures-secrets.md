---
domain: flows/treasures-secrets
tags: [modlayer, gameplay, unpacked, lua, treasure, stash, secret, loot, flow]
sources: [unpacked/scripts/treasure_manager.script, unpacked/scripts/tasks_stash.script, unpacked/configs/items/settings/treasure_manager.ltx, unpacked/configs/misc/secrets.ltx, unpacked/configs/misc/secret_zaton.ltx]
related: [INDEX.md, modlayer/gameplay/_overview.md, flows/item-behaviors-lua.md, flows/ui-news-tips.md, flows/task-system-lua.md, flows/dynamic-task-types.md, flows/pda-map-news.md, flows/utils-libraries.md, flows/db-registry.md, flows/alife-services.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: treasures & secrets (stashes)

How loot caches work: the **procedural** stash generator that fills every inventory-box in the world,
the **predefined** named secrets, the **quest** stashes, and the radio-frequency hint coupling. The
heart is `treasure_manager.script` (Tronex, "based on CoC treasure manager"). This is a P3 standalone
flow (no dedicated map; routed from the M1 gameplay hub). It closes Phase G8.

**Provenance:** **verified, no agent.** `treasure_manager.script` READ-IN-FULL (header docstring,
`init_settings`, the random-stash trio `create_/get_/set_random_stash`, `try_spawn_treasure`, all
callbacks, both persistence paths, `on_game_start`) — only the `get_random_stash` candidate-search body
is structure-surveyed. `tasks_stash.script` head read (the DRX quest-stash effects) + surveyed; the two
config tiers (`treasure_manager.ltx`, `secrets.ltx`) characterized; the `item_radio` stash coupling
confirmed by grep. **No `gamedata/` override** of `treasure_manager`/`tasks_stash` (verified by `find`).

---

## The core idea (procedural stashes)

Per the file's own docstring: on **new game** the manager picks loot for **every `INVBOX`-class object**
in the world, stores it as a comma-separated string in the save, and **spawns it only when the box is
opened**. There is no per-box authored content for these — it's generated to a value budget.

### Item pool (`init_settings`)

Built from `items\settings\treasure_manager.ltx` (the `ini_treasure` handle):

- **`possible_items`** → each item's `cost`/`kind`/`tier` is read from the system ini and bucketed into
  `item_list[kind][tier]`. `kind` is the item group; `tier` is its importance/rarity within the group.
  Multiuse items that are partly used get their tier reduced. Items can be **excluded by economy type**
  (`eco`, from the new-game "economy_treasure" choice or `diff_eco`) via a trailing exclusion list.
- **`settings_common`** / **`settings_rare`** → `min_weight`/`max_weight` (the stash value budget range),
  `num_of_tiers`, and `kind_tier_effect_*` (per-kind base `chance`/`weight` plus per-tier
  `chance_multi`/`weight_multi`). Higher tiers = rarer chance, bigger weight impact.
- **`blacklist_stashes_names`** / **`rare_stashes_names`** → named boxes to skip / to treat as rare.
- **`settings_rare.reward_bonus`** (toolkit list), `completed_task_num`, `start_chance` → the rare-stash
  toolkit reward gate (below). **`stash_bonus`** → extra bonus item pool.

### Generation (`set_random_stash`)

Fills a random target `stash_weight` (between min/max; +200 with the `rag_and_bone` achievement) by
looping: **shuffle kinds → shuffle tiers → roll** `chance*chance_multi + inc`; on a hit, pick a random
item from `item_list[kind][tier]` and subtract `weight*weight_multi` from the budget. An `inc` counter
ramps **up by 10** when a pass adds nothing (anti-stall) and back down on success. Result is stored as
`caches[id] = "sec1,sec2,…"` and a map spot is added (`treasure`, or `treasure_searched` for rare) with
`news_manager.send_treasure(0)` (F112).

### Rare stashes & the toolkit reward

A marked stash can become **rare**: gated on `load_var(actor,"toolkit_reward_count") >= task_num` with a
probability scaled by `start_chance`. When it fires, the next toolkit from `stash_rare.reward_bonus` is
cycled in as a bonus item and the counters reset. Rare stashes exclude multi-toolkits (`i_kit`) from the
random fill and use the `treasure_searched` map spot.

### Lazy spawn (`try_spawn_treasure`)

On `physic_object_on_use_callback` (the box is opened), the cached string is materialized into the box:
**weapons** get a random addon flag (scope/GL/silencer per the section's `*_status`), a random ammo
type, and a random low condition (0–30%) written through `utils_stpk.get_/set_weapon_data` (F115), then
manually `alife():register`-ed; **other items** spawn via `alife_create_item`. `caches[id]` then flips to
`true` (spawned).

### Cache state machine

`caches[id]` is the per-box state: **`false`** = registered, empty · **`"…csv…"`** = content assigned,
not yet spawned · **`true`** = spawned (or looted-empty). On `actor_on_item_take_from_box`, when the box
was `true` and the player takes from it: flip to `false`, remove all treasure map spots, increment the
`stashes_found` statistic, `news_manager.send_treasure(1)`, and `item_radio.clear_stash(lvl,id)`.

## Lifecycle (`on_game_load`)

Scans all server objects (1..65534) for `INVBOX` class, registers each non-blacklisted one as
`caches[id] = false`, then **creates random stashes in half of them** (`create_random_stash(true,"stash")`).
Persistence is dual: a `save_state`/`load_state` callback path (MARSHAL) storing the `caches` table, plus
a `save(pk)`/`load(pk)` net-packet fallback (non-MARSHAL) storing `id`+`bool`.

## Predefined secrets (the other config tier)

`configs/misc/secrets.ltx` (+ per-level `secret_zaton/jupiter/pripyat.ltx` includes) is the **authored**
secret content — a `[list]` master plus `[treasure_name]` sections of `item = count, probability` lines.
This is the legacy CoP-style fixed-content path, consumed by `se_item` (spawn), the task functors
(quest stashes), and stash-hint dialogs — distinct from `treasure_manager`'s procedural fill of generic
boxes. Both tiers coexist: authored secrets for story/specific caches, procedural loot for the rest.

## Quest stashes (`tasks_stash`)

`tasks_stash.script` is the DRX quest-stash type: `xr_effects.drx_sl_create_quest_stash` spawns a quest
item into a random available stash location, and `drx_sl_quest_stash_bonus` picks a random bonus
(high-value ammo) for the reward. Sits on the task system (F97/F98,
`flows/task-system-lua.md`, `flows/dynamic-task-types.md`).

## Stash radio hints

Stashes can carry a radio-frequency source: `item_radio` keeps `RF_stashes[level][id]` and emits a
locatable signal the radio item picks up (the "tune the radio to find a stash" hint). `treasure_manager`
calls `item_radio.clear_stash` when a stash is emptied so its signal stops.

## Verified quirks

- **All loot is new-game-locked** — content for procedural boxes is decided at new game / first load and
  frozen in the save; changing `treasure_manager.ltx` afterwards doesn't re-roll existing boxes.
- **Only half the boxes get stashes** — `on_game_load` fills `floor(caches_count/2)`, so most INVBOXes
  are empty by design.
- **Map-spot "Save as Rare Stash"** right-click tool only registers when `enable_debug` is set (the
  `map_spot_menu_*` callbacks are gated), and it writes back into `treasure_manager.ltx` live via
  `ini_file_ex`.
- **Dual persistence** — both a MARSHAL `caches`-table path and a net-packet `save/load` path exist; only
  one runs depending on `USE_MARSHAL`.

## Deeper

- Item spawning these rely on → `flows/item-behaviors-lua.md`, `alife_create_item`/`se_item`
  (`flows/alife-services.md`); weapon packet writes → `flows/utils-libraries.md` (`utils_stpk`, F115).
- Treasure news + map spots → `flows/ui-news-tips.md` (F112, `send_treasure`), `flows/pda-map-news.md`.
- Quest stashes → `flows/task-system-lua.md` (F97), `flows/dynamic-task-types.md` (F98).
- Save state → `flows/db-registry.md`; gameplay hub → `modlayer/gameplay/_overview.md` (M1).
