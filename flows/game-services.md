---
domain: flows/game-services
tags: [modlayer, gameplay, unpacked, lua, achievements, statistics, rank, reputation, autosave, fast-travel, flow]
sources: [unpacked/scripts/game_statistics.script, unpacked/scripts/game_achievements.script, unpacked/scripts/game_fast_travel.script, unpacked/scripts/game_backpack_travel.script, unpacked/scripts/game_autosave_new.script, unpacked/scripts/game_autosave.script, unpacked/scripts/game_registrator.script, unpacked/configs/plugins/achievements.ltx, src/xrGame/ui/UIRankingWnd.cpp, src/xrGame/ui/UIAchievements.cpp]
related: [flows/game-modes.md, flows/actor-survival-status.md, flows/anomaly-field-binders.md, flows/relations-info.md, flows/pda-map-news.md, flows/trade-economy.md, flows/lua-callback-lifecycle.md, modlayer/gameplay/gamemodes-services.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: game services (F107)

The `game_*` service modules that sit beside the gamemode plugins (F106): the rank/reputation +
counter engine, the achievement detector, autosave, and the two PDA-map travel features. Small
corpus (~2.0K lines across seven files), **all READ-IN-FULL this session, no agent; no
`gamedata/` override of any file** (re-checked on pickup). Two of the seven are effectively dead
(see below), so the live surface is four modules.

## game_statistics — the rank/reputation engine (the spine)

The hub the other services and much of the gameplay corpus feed. Holds five in-memory tables and
turns *events* into *rank/reputation* with band-crossing notifications.

- **`actor_statistics`** — per-event counters, each carrying a `rank` and `rept` reward applied
  on each increment. `increment_statistic(key)` bumps the count and calls `increment_rank` +
  `increment_reputation` (optional custom overrides). Consumers across the corpus call this
  (kills, tasks, stashes, crafting, emissions survived, …).
- **`actor_achievements`** — unlock booleans + a one-time rank/rept reward per achievement; the
  reward is applied by `actor_on_achievement_earned` (below), not by the detector.
- **`npc_statistics`** — the NPC-side mirror. `increment_npc_statistic` rewards an NPC's own rank
  (`set_character_rank`) and reputation, and for stats flagged `save=true` persists a running
  count into the NPC's `m_data` via `alife_storage_manager.get_se_obj_state` (so an NPC's kill
  tally survives offline). This is how NPCs rank up from their simulated kills, not just the
  actor.
- **`actor_visited_levels` / `actor_visited_smarts`** — visited-place tracking. `on_game_load`
  marks the current level; `actor_on_interaction("smarts", …)` marks smarts on approach. These
  are the read surface for fast travel (visit-only mode) and the `tourist` achievement.
- **`actor_artefacts` / `actor_anomaly_maps`** — found-artefact ids and unlocked anomaly maps.

**Rank/reputation mechanics.** `increment_rank` / `increment_reputation` clamp to engine ranges
(rank 0–56000, reputation −4000..4000) and call `check_for_rank_change` /
`check_for_reputation_change`. Those map the raw integer to a *band index* using the `rating` /
`reputation` value lists from `creatures\game_relations.ltx` (parsed once into `rank_table` /
`rept_table`), compare the new band against the snapshot held in `actor_miscellaneous`
(`actual_rank` / `actual_rept`), and fire a `news_manager.send_tip` ("rank increased/decreased")
only on a band crossing. The band config is the F108/F118 relations data tier.

**Artefact take handling** (`actor_on_item_take`): increments `artefacts_found` once per unique
artefact id, and if the artefact is anomaly-associated (looked up in
`bind_anomaly_zone.parent_zones_by_artefact_id`, F90) calls `anomaly:on_artefact_take` and
increments `artefacts_detected`. It teleports the artefact away with `FollowByPath` and guards the
**container exploit** via the `_G.ARTY_FROM_CONT` flag (artefacts pulled from a container set the
flag, which this callback consumes to skip the rank gain).

**Persistence.** MARSHAL `save_state` / `load_state` only. It deliberately saves achievement
*unlock booleans* and statistic *counts* — **not** the `rank`/`rept` reward values baked into the
table literals — so that editing a reward in this script takes effect on existing saves instead of
being frozen in. The visited/artefact/anomaly tables are saved whole.

**Quirks (verified).** `articles` and `killed_stalkers` counters are incremented but unused for
rewards (rank=0/rept=0; a comment points kill-reward at the `rank_kill_points` config instead).
`increment_reputation` early-returns during `bar_arena_fight` (arena fights must not move
reputation). `on_game_load` sets `debug_mode_flag_on` when `DEV_DEBUG` — a permanent cheater flag
that disqualifies the `invictus` achievement.

## game_achievements — the detector (pull-based, UI-driven)

Pure functor/requirement/reward triplets — 24 achievements, each a `<name>_functor` that: bails
unless `can_unlock()` (`db.actor.afterFirstUpdate`), returns `true` if already unlocked, else tests
`<name>_requirements()` and on success runs `<name>_rewards()`. Rewards fire
`SendScriptCallback("actor_on_achievement_earned", name, msg)` — which lands in
`game_statistics.actor_on_achievement_earned` (the recorder: news tip + flip unlock bool + apply
the table's rank/rept reward).

**What invokes the functors (the spine, verified in C++).** There is **no Lua poll** and no
periodic timer. The PDA *Ranking* tab drives it: `CUIRankingWnd` reads the `[achievements]` section
list, builds one `CUIAchievements` widget per entry, and `SetFunctor`s the config's `functor=`
string. `CUIAchievements::Update` calls the Lua functor `f()` via `R_ASSERT(... functor ...)` each
refresh; truthy *and* not-already-shown (and not `repeatable`) adds it to the displayed list. So an
achievement is only *evaluated and unlocked while the player has the ranking PDA tab open and it
ticks* — the functor's unlock+reward is a side effect of that UI check. (`tourist_functor` works
around level-transition false-fires by gating itself behind a `tourist_achievement_delay`
info-portion across two evaluations.)

**Config** (`plugins\achievements.ltx`): each `[name]` gives `icon`/`hint`/`name`/`desc` +
`functor=game_achievements.<name>_functor` + optional mode tags `story` / `ironman` / `warfare` /
`ignore`. `get_achievements_count` scans the section and counts only achievements applicable to the
current mode (story requires `not story_mode_disabled`, ironman requires `IsHardcoreMode`, warfare
requires `_G.WARFARE`); `completionist` needs `get_achievements_locked_count() <= 1` (the ≤1
tolerates completionist itself still being locked).

**Rewards are distributed, not central.** A `_rewards()` body only records the unlock. The actual
*effect* lives at each consumer site gated by `game_achievements.has_achievement("X")` — verified
across ~15 files: artefact spawn chance (`bind_anomaly_zone`, F90), trade prices and PDA-delivery
pay (`dialogs`), companion squad size (`axr_companions` / `dialogs_axr_companion`), disguise
suspicion (`gameplay_disguise`), disassembly/craft/repair bonuses (`item_parts` / `item_recipe` /
`item_repair` / `ui_workshop`), emission survival (`level_environment`), box/mutant loot
(`xr_box` / `ui_mutant_loot`), task-stash loot (`treasure_manager`, F117), and condlist gates
(`xr_conditions`). A few rewards *do* act directly in the body: `tourist` spawns 3 treasure stashes
(`treasure_manager.create_random_stash`); `wishful_thinking` / `absolver` / `collaborator` unlock
factions by writing `axr_main.config` `unlocked_factions` and saving; `iron_curtain` gives 40000 RU.

**Quirks (verified).** `bookworm_food` is **commented out of the `[achievements]` list** (and
`ignore=true`), so `CUIRankingWnd` never builds its widget → its functor is never polled → it is a
**dead, unreachable achievement** despite having a full triplet and a table row. Several
requirement comments have drifted from the code: `murky_spirit` says "Lost to the Zone" but checks
`operation_afterglow_complete`; `wishful_thinking` and `invictus` both say "Living Legend" but
check `living_legend_done` and `operation_afterglow_complete` respectively. `invictus` is the
strictest gate in the corpus (story + afterglow + hardcore + ironman-alive + max gameplay *and*
economy difficulty + no difficulty tampering + no `debug_mode_flag_on`).

## game_autosave_new — the live autosave (29 lines)

The only working autosave. `actor_on_first_update` seeds `last_save` and a 60s `actor` TimeEvent;
each tick reads `ui_options.get("other/autosave_timer_new")` (minutes) and, if the interval has
elapsed, runs `exec_console_cmd("save <user> - tempsave")`. `option_updated` resets the timer when
the setting changes mid-game. **`game_autosave.script` is entirely commented out** — its header
says "now unused"; it is the superseded interval/rotation implementation kept only as reference.

## game_fast_travel — PDA-map fast travel to smarts

PDA map-spot travel to a faction-whitelisted set of smart terrains. The whitelist `tsmarts` comes
from `plugins\faction_quick_travel.ltx` keyed by the actor's community; settings come from
`ui_options` (`gameplay/fast_travel/*`). `enabled` is a tristate:

- **0 off** — nothing.
- **1 visit-only** — markers appear only for whitelisted smarts the actor has visited
  (`game_statistics.has_actor_visited_smart`); `actor_on_interaction` discovers a marker on
  approach (and optionally announces it).
- **2 all** — every whitelisted smart gets a marker immediately.

It registers `map_spot_menu_add_property` / `map_spot_menu_property_clicked` (the PDA-map context
menu seam, F39): the add-callback appends a "Fast travel to …" item, the click-callback runs the
travel. Travel is gated by a battery of toggles (`combattravel` / `weighttravel` / `damagetravel`
/ `stormtravel`) — refusing while fighting, overweight (carry minus suit/belt-artefact bonuses),
bleeding/irradiated, or during a surge/psi-storm — then optionally forwards game time
(`level.change_game_time` by distance, plus `surge_manager`/`psi_storm_manager` `time_forwarded`
and a forced weather change) and finally either `set_actor_position` (target online) or
`ChangeLevel` (target offline / cross-level). `DEV_DEBUG` exposes travel to every smart.

## game_backpack_travel — PDA travel to deployed backpacks

Same machinery as fast travel, but the targets are the player's **deployed backpacks** (server
objects named `inv_actor_backpack` / `inv_backpack`), settings under
`gameplay/backpack_travel/*`. Adds two map-spot menu items: **bp_to** (travel to the backpack,
same guard battery + time-forward + position/ChangeLevel logic) and **bp_open** (if the backpack
is online, `:use(db.actor)` to open it remotely). The carry-weight check shares the same
`bbp_extra_weight` belt/backpack accounting helper as fast travel.

## game_registrator — vestigial MP scaffolding

`register(object_factory)` is empty; `get_game_clsid(mode, is_server)` maps a multiplayer game
mode string ("single"/"deathmatch"/…) to the engine game-class clsid (`SV_*` / `CL_*`). Only
`"single"` is reachable in Anomaly; the DM/TDM/AHUNT/CTA branches are inherited multiplayer
plumbing. Effectively dead for single-player gameplay but on the engine's game-class boot path.

## Where this sits

These are bus plugins on the F65 lifecycle, siblings of the F106 gamemode plugins in the same M19
map. `game_statistics` is the rank/reputation authority the corpus increments; `game_achievements`
is a thin detector recorded back through it; fast/backpack travel are PDA-map (F39) consumers that
read `game_statistics` visited-tracking; autosave is a console-cmd timer. The rank/reputation band
tables come from the relations data tier (F108/F118).

## Deeper

- The gamemode plugins and difficulty store that share this map → `flows/game-modes.md` (F106).
- The PDA-map context-menu seam these travel features hook → `flows/pda-map-news.md` (F39).
- Anomaly artefact economy that `actor_on_item_take` feeds → `flows/anomaly-field-binders.md` (F90).
- Rank/reputation band config and goodwill → `flows/relations-info.md` (F40; F108/F118 own the data).
