---
domain: flows/game-modes
tags: [modlayer, gameplay, unpacked, lua, gamemodes, difficulty, ironman, azazel, saves, flow]
sources: [unpacked/scripts/game_difficulties.script, unpacked/scripts/game_setup.script, unpacked/scripts/gamemode_azazel.script, unpacked/scripts/gamemode_ironman.script, unpacked/scripts/gamemode_agony.script, unpacked/scripts/gamemode_timer.script, unpacked/scripts/ui_mm_faction_select.script, unpacked/scripts/item_artefact.script, src/xrGame/EntityCondition.cpp]
related: [flows/actor-survival-status.md, flows/warfare-metagame.md, flows/trade-economy.md, flows/item-behaviors-lua.md, flows/actor-condition.md, flows/world-object-binders.md, flows/save-load.md, flows/lua-callback-lifecycle.md, modlayer/gameplay/gamemodes-services.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: game modes & difficulty (F106)

The opt-in rule modifiers chosen at new game and the two difficulty-factor families everything
else reads. Small corpus (~1.5K lines, six files), **all READ-IN-FULL this session, no agent;
no `gamedata/` override of any file**. The pattern is uniform: `ui_mm_faction_select` (the
new-game screen) reads `axr_main.config` `character_creation` values and seeds persistent flags
in `alife_storage_manager` state; each mode is then a small callback plugin keyed on its flag.

## The flag registry (seeded by `ui_mm_faction_select`, verified)

`enable_warfare_mode` (F105), `enable_campfire_mode` (save-at-campfire-only — implemented in
`bind_campfire`, F89, not here), `enable_azazel_mode`, `enable_timer_mode` (a *number*: cooldown
minutes), `enable_conditions_mode` (agony), and the `ironman` table (fresh `uuid` from
game-version + randoms, `death_limit`, `death_count`, optional `life_feature`/`life_cycle`,
`granted_lives`, `sleep_hrs`). `_g` exposes the cheap tests (`IsWarfare`, `IsCampfireMode`,
`IsTimerMode`-style checks read the same storage).

## game_difficulties — the two factor families

The central knob store every system polls (`get_game_factor` / `get_eco_factor`). Two families
read from `plugins\difficulty.ltx` presets (`gdiff_N` gameplay, `econ_N` economy) with per-key
player overrides persisted in alife storage (`diff_game` / `diff_eco`):

- **Gameplay**: actor immunities (mapped to vanilla `gd_*` via `g_game_difficulty` console cmd),
  hit power, dispersion base/factor, stamina `power_loss_bias`, max carry weight (engine setters
  + walk weight = weight + ceil(weight/5)×2), and the booleans that toggle whole features —
  thirst/sleep (`actor_status_*.toggle_feature`, F103), radiation-day, geiger/anomaly notify.
- **Economy**: goodwill/rewards/repair/upgrade/buy/sell multipliers (F101 pricing), artefact
  price, loot/money-loot density, stash chance, weapon degradation (console `wpn_degradation`),
  battery consumption, `random_items` (feeds game_setup below), scope/arty chances, trader
  `restock` hours, percentage-parts, limited-bolts, arty/backpack degradation toggles, outfit
  drops.

Application is push-based: `set_*_factor(key, val)` saves, rebuilds the table, and re-applies
the one affected console command / engine setter / feature toggle; `on_game_load` re-applies
everything (and force-disables `ai_die_in_anomaly` with a salty comment). First change of any
factor permanently gives `diff_gameplay_flag_change` / `diff_economy_flag_change` info portions
— the tamper marker achievements and stats can check.

**Verified conditional bug:** `on_game_load` calls
`item_artefact.toggle_feature_arty_degradation` twice in a row — first with the
`arty_degradation` factor, then with `bkpk_degradation` — but it is **one shared toggle** for
one before-hit callback (read in `item_artefact`): the second call wins the registration. With
arty=true + backpack=false on load, the callback is unregistered and artefact degradation is
silently off (the callback itself does gate each half on its own factor internally, so any
*other* combination behaves). Changing either factor in options later re-toggles and can mask
or re-trigger it.

## game_setup — dynamic world items & one-time fixups (misnamed, verified)

Despite the name, this is mostly the **dynamic world-item conservation loop**:
`items\settings\dynamic_item_spawn.ltx` records per-level item positions (`[levels]` →
name → type,x,y,z,lvl_id,gm_id) and weighted section pools (`[categories]`). On the first actor
update of a playthrough it purges listed static items and inventory-box stashes
(`replace_items` / `remove_objects` from `plugins\new_game_setup.ltx`, stash-manager-aware),
then spawns `random_items`-eco-factor × pool-size items at random recorded spots. From then on
**taking a world item spawns a replacement at some other free spot** (preferring other levels),
keeping world density constant. Randomized properties (ammo count from box_size quarter-to-¾,
multiuse uses from a `possible_uses` range, condition 30–70% or part-tier 0.5/0.75/1) are
recorded at spawn but **applied at pickup** via `alife_process_item` — the F102 ItemProcessor
seam. State (`world_itm_on`/`world_itm_num`) rides `save_state`/`load_state`.

Also here: three one-time map fixups (bar medic clutter, darkscape train physics objects,
freedom medic chair+smart-cover teleport) each behind an alife-storage once-flag and an
admitted "TODO: until we can edit all.spawn" comment — the shipped way to patch level geometry
problems from Lua; and **cordon newbie protection**: while a fresh loner actor (≤1 level change,
non-warfare) is on `l01_escape`, any common army squad on the level is removed every 10 s.

## The four gamemode plugins (all READ-IN-FULL)

- **Agony** (`enable_conditions_mode`) — save gating: `on_before_save_input` vetoes
  (`flags.ret = true` + message + menu close) when health < 50% (four severity strings),
  bleeding, radiation > ~100 mSv, surge/psi-storm active, or NPCs are currently fighting the
  actor (`xr_combat_ignore.fighting_with_actor_npcs`).
- **Timer** (`enable_timer_mode` = minutes) — post-save cooldown: a `save` console command
  arms a TimeEvent; until it expires the same save-input veto fires.
- **Ironman** — death accounting with a startling implementation (verified): on
  `actor_on_before_death` it iterates **every `.scoc` save sidecar on disk**, marshal-decodes
  each, and for files matching the playthrough `uuid` increments `death_count` and rewrites
  them — deleting saves outright (`ui_load_dialog.delete_save_game`) once `death_limit` is
  reached. Save-file mutation outside the engine save path. The optional **Second Wind**
  (`life_feature`) grants a death back every `life_cycle` played hours (sleep hours excluded
  via `actor_on_sleep`). Turning ironman off or dying twice permanently gives
  `ironman_flag_off` / `ironman_flag_died` info portions.
- **Azazel** (`enable_azazel_mode`) — death is a body-swap, not a respawn:
  `actor_on_before_death` cancels the death (`flags.ret_value = false`), picks a victim among
  living `sim_default_*` stalkers (non-warfare: uniform random; warfare: candidate classes —
  companions / own faction / allies / neutrals / enemies — each option-gated with min/max
  distance windows, nearest-sort or shuffle), spawns a dead `stalker_azazel` mimic at the old
  position carrying the player's name/icon/visual and inventory (minus a starters ignore-list),
  copies the victim's identity (name/rank/community/reputation/icon + faction goodwill +
  random pocket money), regrants a loadout via `death_manager` (cosmetics/ammo, novice outfit
  fallback) + PDA + backpack, cancels all active tasks, teleports or `ChangeLevel`s to the
  victim, releases it, and stops any surge/psi-storm ("due to bug with killing actor on
  switch"). `heal_everything` includes `db.actor.bleeding = 1` — which *heals*: the property
  setter is `CEntityCondition::ChangeBleeding(percent)`, wound incarnation, verified in C++.
  Warfare's possess-leader (F105) is a sibling of this code.

## Verified quirks

- The arty/backpack degradation double-toggle on load (see above) — the one real conditional
  bug in this domain.
- Azazel computes `local m = tonumber(...dist_mult)` in **five** candidate branches and never
  uses it — the distance-multiplier options are dead.
- Azazel's `squad_mates` table is initialized empty and never filled, so the "don't cancel
  tasks if taking over a squadmate" guard always cancels (the fill code is in a commented-out
  block).
- `db.actor.bleeding = 1` heals (ChangeBleeding = wound incarnation percent) — reads backwards.
- `gamemode_agony`/`gamemode_timer` are not modes of the same kind as ironman/azazel: they're
  pure save-policy filters; campfire mode (the fifth save policy) lives in `bind_campfire`.
- `gamemode_azazel` borrows `warfare.printd` and `warfare_options` even outside warfare — the
  azazel candidate options live in the warfare options table.

## Deeper

- Difficulty factor consumers: trade pricing → `flows/trade-economy.md`; sleep/thirst toggles →
  `flows/actor-survival-status.md`; artefact spawn chance → `flows/anomaly-field-binders.md`.
- The ItemProcessor seam game_setup uses for deferred uses/condition →
  `flows/item-behaviors-lua.md` (F102).
- Save pipeline the ironman sidecar mutation rides on → `flows/save-load.md`.
- Campfire save mode → `flows/world-object-binders.md` (bind_campfire).
- Domain routing → `modlayer/gameplay/gamemodes-services.md` (M19).
