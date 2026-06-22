---
domain: flows/axr-managers
tags: [modlayer, gameplay, unpacked, lua, axr, manager, task, companion, trade, flow]
sources: [unpacked/scripts/axr_task_manager.script, unpacked/scripts/axr_companions.script, unpacked/scripts/axr_trade_manager.script, unpacked/configs/ai_tweaks/axr_companions.ltx]
related: [INDEX.md, modlayer/gameplay/axr-addon.md, flows/axr-behaviour-addons.md, flows/companion-system.md, flows/task-system-lua.md, flows/dynamic-task-types.md, flows/trade-economy.md, flows/relations-info.md, flows/gameplay-boot.md, flows/script-npc-actions.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: AXR managers (task / companion / trade)

The three high-level `axr_*` **managers** — the systems the addon layer builds on top of the stock
A-Life corpus, as opposed to the per-NPC behaviour schemes of F113. Each is the Lua-side driver for a
player-facing system that the engine only half-implements:

- **`axr_task_manager`** — the DoctorX-Storyline (DRX SL) task **generator/extender** that sits on the
  engine task core (F97, `flows/task-system-lua.md`). It is what makes faction honchos hand out
  randomized storyline / mechanic / change-faction tasks, and it owns the **bounty** and **hostage**
  registries the dynamic quest types read.
- **`axr_companions`** — the squadmate **roster + command** layer. The scheme/info-flag model an
  individual companion runs is F82 (`flows/companion-system.md`); *this* flow is the manager half —
  joining/dismissing, the command system that writes the `npcx_beh_*` flags, special/story squads,
  inventory assignment, the command wheel, and the upkeep timers.
- **`axr_trade_manager`** — the autonomous NPC↔NPC gulag-job economy. **Already documented in full
  under F101** (`flows/trade-economy.md`); summarized here only to place it in the family.

All three load + tick through `axr_main` (F65, `flows/gameplay-boot.md`) and hook the
`RegisterScriptCallback` bus. This flow is the second half of the M21 axr-addon map
(`modlayer/gameplay/axr-addon.md`); F113 was the first.

**Provenance:** **verified, no agent.** `axr_task_manager` — callbacks, `on_game_start`, the two honcho
tables, `generate_cache`/`CFG_CACHE`, and `get_first_available_task` READ-IN-FULL; the `drx_sl_*`
generator variants and the `generate_available/ongoing/finished_tasks` dialog builders
structure-surveyed (function list + caller grep across `dialogs`/`xr_conditions`/`xr_effects`).
`axr_companions` — the manager layer READ-IN-FULL (roster lifecycle, special-squad release,
`xr_effects.*` story hooks, inventory transfer, keybinds, command-wheel config); the command-mode
setters and the two `CUIScriptWnd` UI classes structure-surveyed (function list). `axr_trade_manager`
read 1–200 + function-surveyed, but **owned by F101** — see there. **No `gamedata/` override of any of
the three** (verified by `find`). This flow deepens the parts F82 explicitly left agent-traced.

---

## axr_task_manager — the DRX storyline task driver

The engine task core (F97) is a generic shell: it holds active task objects and re-evaluates them per
frame. It does **not** decide *which* tasks an NPC offers. `axr_task_manager` is the DoctorX-Storyline
extender that fills that gap — a flat module (no class) of global tables, callbacks, and `drx_sl_*`
generator functions called from honcho **dialogs**.

### State (module globals, save/load persisted)

- `bounties_by_id` — `task_id → target npc_id`. The kill-target registry for bounty tasks (F98
  `tasks_bounty`). Read by `dialogs`, `dynamic_news_manager` (F87/F105 gossip), `stalker_generic`.
- `hostages_by_id` — `npc_id → smart_id`. Hostage NPCs spawned by companion escort tasks (written by
  `axr_companions.setup_companion_task`, below). The companion roster filters these out so a hostage
  never shows as a commandable squadmate.
- `available_tasks` / `ongoing_tasks` / `finished_tasks` — caches the dialog menu builders fill so the
  honcho's "tasks" dialog can list phrases.
- `skipped_tasks` (file-local) — DoctorX "skip this one" set, cleared by `clear_skipped_tasks`.
- `CFG_CACHE` — every task section's hot fields (`stage_complete`, `sim_communities`, `job_descr`,
  `fetch_descr`, `fetch_func`, `task_complete_descr`, `can_cancel`, `on_job_descr`) pre-parsed from
  `task_manager.task_ini` by `generate_cache`, so the per-frame/per-dialog generators iterate a table
  instead of re-reading ltx. Rebuilt on `load_state`.

### The honcho tables

`drx_sl_main_honcho_table` maps each `community → the faction's main trader story-id` (Sidorovich for
stalker, Petrenko for dolg, Sakharov for ecolog, Sultan for bandit/renegade, …). `drx_sl_honchos_table`
is the fuller list of **every** task-giving NPC — traders, mechanics, and faction leaders — each row a
`{story_id, community[, community2]}`. These are the NPCs whose dialogs expose the storyline-task menu.

### Callbacks (registered in `on_game_start`)

- `npc_on_death_callback` — when any bounty target dies, find its task in `bounties_by_id` and advance
  the engine task stage. Credit requires the kill be the actor's: either the target was
  `tagged_by_actor` (hit by actor/companion) or `who` is the actor / an `npcx_is_companion`. **Design
  quirk:** the non-credited branch used to set `stage = 3` (fail) but now sets `stage = 1`
  ("complete task anyway instead of failing") — bounties no longer fail if someone else gets the kill.
- `npc_on_hit_callback` (also wired to `monster_on_hit_callback`) — marks `db.storage[id].tagged_by_actor`
  when the actor or a companion hits the target, so a later death counts as the actor's bounty kill.
- `squad_on_npc_death` — if the dead NPC was a task giver, re-home `task_giver_id` to the squad
  commander for `simulation_task_*`, else clear it.
- `on_before_level_changing` — GC pass that clears orphaned `task_id` / `_fetch` / `_count` pstor vars
  for tasks no longer active.
- `save_state` / `load_state` — persist `bounties_by_id` + `hostages_by_id`; load rebuilds `CFG_CACHE`.

### Generation pipeline

`get_first_available_task(npc, skip, is_sim)` is the core picker:

1. **Stored-task cache** — each NPC's last offer is kept in `drx_sl_npc_stored_task_<id>` with a
   timestamp; reused for **5400 s (3 in-game hours)** if still valid (prereqs met, not already active).
2. **75 % gate** — outside the cache window there's a 25 % chance the NPC simply has nothing.
3. **Section prefix** — the task pool for an NPC is `<task_section>_task_` (from the NPC's logic
   `task_section` field) → else `<npc:section>_task_` → else `<npc:name>_task_` for unique NPCs like
   `m_trader` → else `simulation_task_` for generic sim stalkers.
4. **Filter + pick** — iterate `CFG_CACHE` for sections matching the prefix, keep those not skipped /
   not active / prereqs satisfied (`xr_conditions.has_completed_task_prerequisites`); for sim tasks
   also require the NPC's `character_community` be in the section's `sim_communities`; pick one at
   random and persist it. (The pre-CFG_CACHE while-loop over `<sec>1, <sec>2, …` is left commented out.)

The `drx_sl_*` siblings are typed entry points over the same machinery:
`drx_sl_generate_random_sl_task` (storyline), `drx_sl_get_mechanic_task` (mechanic upgrades),
`drx_sl_generate_change_factions_task` (defection), and the completion side
`drx_sl_get_finished_task` / `_cf_task` + `drx_sl_set_finished_task_complete` / `_cf_task_complete`.

### Dialog surface

The honcho "tasks" dialog (in `dialogs.script`) drives the menu through:
`generate_available_tasks` / `generate_ongoing_tasks` / `generate_finished_tasks` (build the phrase
lists), `get_task_job_description` / `get_fetch_task_description` / `get_task_complete_text` (text),
`npc_give_first_available_ordered_task` (hand it over), and `trigger_fetch_func` / `trigger_job_func`
(run the task's configured fetch/job hooks). This is the bridge: the generators decide *what*, the
engine task core (F97) tracks *progress*, and F98 (`flows/dynamic-task-types.md`) supplies the per-type
world logic (`tasks_bounty` clears `bounties_by_id` on completion; `tasks_defense`/`tasks_faction_control`
consume `hostages_by_id`).

---

## axr_companions — the roster & command manager

F82 (`flows/companion-system.md`) covers what a *single* companion is: a stalker switched to the
`beh_companion.ltx` logic running the `axr_beh` brain (F113), reading `npcx_beh_*` info flags each
frame. This section is the **manager** that maintains the roster and *writes* those flags.

### Roster model

- `companion_squads[squad.id] = squad` — **all** companion squads (voluntary recruits, story escorts,
  hostages).
- `non_task_companions[id] = true` — only **voluntary** recruits; these count against
  `max_actor_squad_size`. `get_companion_count` sizes this table.
- `assigned_items[npc_id][item_id]` — which items the player handed to which companion (ownership for
  give/take-all and weight rules).
- `_current_states` — the default `combat/move/stealth/loot/distance` modes, applied to new recruits
  and persisted across saves.

Config is `ai_tweaks/axr_companions.ltx`: `enable`, `allow_only_friends_as_companions`,
`allow_simulation_squads_as_companions`, `max_actor_squad_size` (default 2). `on_game_load` **doubles**
the cap to `max_actor_squad_size_patriarch` (4) if the `patriarch` achievement is unlocked.

### Join / dismiss

- `can_join_actor` — gate on squad size (NPC's own squad count + current companions ≤ cap) and that the
  NPC is its own squad's commander.
- `add_to_actor_squad` — give `npcx_is_companion`, `se_save_var` the `companion` flag,
  `setup_companion_logic` (unregister from smart terrain, then `xr_logic.configure_schemes` onto
  `beh_companion.ltx` = installs the F113 `axr_beh` scheme), then apply the current command modes.
- `remove_from_actor_squad` — restore the NPC's original scheme/logic, clear the info portions and
  saved vars, drop the level spot; under WARFARE retargets the freed squad to the nearest smart.
- `remove_all_from_actor_squad` — bulk version (uses `execute_script_on_squad` to avoid the
  relations-tab stacking bug).

### Special / story squads (dialog-driven)

For scripted escorts the manager exposes `xr_effects.*` story hooks: `add_task_companion`,
`add_speaker_companion` (the NPC the actor is currently talking to), `add_task_giver_companion`,
`setup_companion_task` (**spawns** a fresh escort squad at a smart terrain, marks it
cannot-dismiss/cannot-teleport, registers hostages into `axr_task_manager.hostages_by_id`, and adds it
to `sim_offline_combat.task_squads` so offline combat ignores it), plus the removal/dismiss/release
counterparts. Releasing plays a `fade_in_out.ppe` screen effect (`releaser`) and stops any in-progress
talk before the squad is removed. `epic_hack` is misnamed — it's a small helper that sets
`xr_danger` on a companion when a helicopter is targeting the actor or that companion.

### Command system → `npcx_beh_*` flags

The whole point of the command layer is to set the info portions the `axr_beh` brain (F113) reads.
Five command axes, each with `set_companion_to_*` primitives, a `get_companion_*_mode` reader, a
per-squad `set_companion_squad_*_mode`, and a `cycle_companions_*_mode` (all companions):

| Axis | Modes |
|------|-------|
| combat | attack / ignore-combat / attack-only-actor-enemy / camper / monolith |
| move | follow / wait / patrol |
| stealth | stealth-substate / relax / default |
| loot | items+corpses / items-only / corpses-only / nothing |
| distance | stay-close / stay-far |

`move_to_point` issues a one-shot "go here" order (the `axr_beh` waypoint target). Keybinds
(`on_key_release` + one `axr_keybind.bind`, F113): **kCUSTOM18** opens the command wheel,
**kCUSTOM1–5** cycle combat/move/stealth/loot modes, **kCUSTOM3** = `move_to_point`.

### Command wheel + list + inventory

`start_CW` opens the **command wheel** (`UIWheelCompanion`, a `CUIScriptWnd`); `cw_prepare` maps each
slice (combat/movement/stealth/distance/loot/trade → `DIK_1..6`) to its get/set/cycle functions and
on/off label strings. `UICompanionList` is the roster panel. Trade opens `ui_companion_inv` via
`start_trade`. Inventory plumbing: `transfer_item` / `transfer_all_item` move items while maintaining
`assigned_items` ownership; `OnItemBeforeMove` / `OnItemAfterMove` hook the loot screen to track
assignment and **veto** drops that would overweight the companion (`is_overweight`, **hardcoded 30 kg**
cap on assigned items regardless of the NPC). `give_all_to_companion` / `take_all_from_companion` /
`can_handle_all_items` back the wheel's trade actions.

### Upkeep timers

`on_game_load` arms two `CreateTimeEvent` cycles: `healing_timer` (every 10 s, regen any companion
below full health by +0.1) and `unstuck` (every 30 s — a deliberate hack that teleports a companion
to its own position when it's *following + catching-up + peaceful + standing-still + far* from the
actor, i.e. stuck on geometry). `on_level_changing` teleports companion squads to the actor on map
change, unless `companion_squad_can_teleport` is false (a `companion_cannot_teleport` var, or the squad
is in `npcx_beh_patrol_mode` / `npcx_beh_wait`).

---

## axr_trade_manager — see F101

The third member of the family is the **autonomous NPC↔NPC economy**: gulag-job stalkers walk to a
trader to sell surplus loot (and buy ammo) and to a mechanic to upgrade weapons. It hooks
`npc_on_item_take` to accumulate per-NPC surplus, exposes `npc_has_items_to_sell` (condition) +
`trade_job_sell_items` / `trade_job_give_id` (effects) for gulag job logic, and reads
`items/trade/gulag_job_trade_buy_sell.ltx` (with `no_trade_comm` excluding zombied/army/monolith/
greh/isg). It is **read in full and documented under F101** — `flows/trade-economy.md` (economy #3).
It lives here only so the `axr_*` manager trio is complete in one place.

---

## Verified quirks

- **Bounties never fail** — `npc_on_death_callback`'s no-credit branch was changed from `stage = 3`
  (fail) to `stage = 1` (complete); a bounty target killed by a third party still completes the task.
- **Hardcoded 30 kg companion carry cap** in `is_overweight` — the assigned-item weight limit ignores
  the NPC's actual strength/inventory.
- **`unstuck` is an acknowledged hack** (its own comment) — teleport-in-place to free companions stuck
  on geometry; relies on a five-predicate heuristic, not real pathing.
- **Backward-save cruft** — `on_game_load` migrates obsolete `companion_N` vars; `load_state` /
  `squad_on_first_update` repopulate `companion_squads` (saved as `false`, rebuilt live). Marked
  "remove me later" but still present.
- **`se_load_var(item_id, nil, "strapped_item")` guard is dead** (in `transfer_item`) — part of the
  nil-global `se_load_var` family documented under F102 (`flows/item-behaviors-lua.md`); the
  strapped-weapon exception path never fires.
- **`epic_hack`** — name is a leftover; it is just a heli-danger helper, unrelated to companions
  generally.

## Deeper

- Engine task core this extends → `flows/task-system-lua.md` (F97); per-type world logic →
  `flows/dynamic-task-types.md` (F98).
- Single-companion scheme + info-flag model → `flows/companion-system.md` (F82); the `axr_beh` brain
  the flags drive → `flows/axr-behaviour-addons.md` (F113).
- NPC↔NPC economy → `flows/trade-economy.md` (F101, economy #3).
- Boot/loader + callback bus → `flows/gameplay-boot.md` (F65); scheme interface → `flows/script-npc-actions.md` (F37).
- Layer map → `modlayer/gameplay/axr-addon.md` (M21).
