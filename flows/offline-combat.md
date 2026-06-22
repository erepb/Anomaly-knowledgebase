---
domain: flows/offline-combat
tags: [modlayer, gameplay, unpacked, lua, alife, squad, simulation, flow]
sources: [unpacked/scripts/sim_offline_combat.script, unpacked/configs/misc/offline_combat_simulator.ltx, unpacked/scripts/game_relations.script]
related: [flows/scripted-squad.md, flows/sim-board.md, flows/relations-info.md, flows/alife-services.md, flows/options-runtime.md, flows/se-others.md, flows/pda-map-news.md, modlayer/gameplay/squads-simulation.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: offline combat simulation (F87)

The **Offline Combat Simulator (OCS)** — `sim_offline_combat.script`: a periodic Lua resolver that makes
**offline enemy squads fight each other statistically** (no engine combat, no online objects). When two
hostile squads are close (full mode) or share a smart_terrain (smart mode), OCS computes each squad's power
from NPC ranks, deals randomized damage, kills NPCs/whole squads, and feeds the result back into faction
relations + news. This is the macro "war happens even where the player isn't" layer on top of the squad sim
(F85/F86). It is a self-contained simulation with no engine seam beyond `on_death`.

**Provenance:** **verified** — `sim_offline_combat.script` spine read this session (the coordinator, both
update hooks, `validate_enemy`, `simulate_battle`, `set_battle_outcome`, the power calcs, settings, boot).
`game_relations.offline_npc_on_death` confirmed present (its comment says "called from
offline_combat_simulator"); the `offline_combat` option (full/on_smarts_only/off) confirmed in
`ui_options.script` (F17); `offline_combat_simulator.ltx` sections `[offline_health]`/`[story_communities]`/
`[ignore_list]` confirmed. **No `gamedata/` override** (verified by listing). P3 / niche.

## Concepts (verified)

- **OCS modes** — the `alife/general/offline_combat` option selects: `full` (state 1, hook
  `squad_on_update` — any two close enemy squads fight), `on_smarts_only` (state 2, hook
  `smart_terrain_on_update` — squads at the same smart fight), or `off` (unregister both hooks). The
  `coordinator` toggles the active callback when the option changes.
- **power** — a squad's combat strength = Σ member `calculate_npc_power`: a stalker contributes its
  `se_obj:rank()`; a monster contributes `m_class[clsid] × m_rank_max(27000)` ± `m_rank_factor` randomization;
  both × the faction's `faction_power` multiplier (warfare only). Cached in `ocs_power[squad.id]`.
- **opponents / level tables** — `opponents[id]` pairs two squads in a fight; `squads_by_level`/
  `count_by_level`/`limit_by_level` scope simulation to crowded levels only.

## Sequence (spine — verified)

### Boot — `on_game_start`
Loads settings (`init_settings` reads `offline_combat_simulator.ltx`), registers `load_state`/`save_state`
(persist `task_squads`), and on `on_game_load` starts a repeating `CreateTimeEvent("cycle","ocs", …,
coordinator)`.

### The driver — `coordinator()` (every `UPDATE_TIME` = 30 real s)
`ResetTimeEvent` to re-arm. Read the `offline_combat` option → set `new_state` (1 full / 2 smart / 0 off)
and **register/unregister the matching callback** when the mode changes (`squad_on_update` for full,
`smart_terrain_on_update` for smart). Then rebuild the working set: clear stale `task_squads`; collect
**task-giver** squads to exclude (from `task_manager`); build `squads_by_level` from `SIMBOARD.squads`
filtered to `common`, **non-story, non-task** squads; compute `count_by_level`; set `limit_by_level[lid] =
true` only when the level's squad count exceeds `pop_factor × MAX_SQUAD_COUNT` (30) — **OCS only simulates
on crowded levels**. Finally pick warfare-vs-normal options (`opt_dist` = `COMBAT_DISTANCE` 20² / warfare
40², `opt_exclude`, `can_fight_mutant`, per-faction power multipliers).

### The hooks — `squad_on_update(squad_1)` / `smart_terrain_on_update(smart)`
Both: for each candidate offline squad on a `limit_by_level` level that is non-task, first try its existing
`opponents[id_1]` (fast path), else scan the level (full) or the smart's squads (smart mode) calling
`validate_enemy` until one returns true (a fight happened). Smart mode shuffles the smart's squad list first.

### Pairing gate — `validate_enemy(sim, lid, squad_1, id_1, id_2, community_1)`
Returns true (and runs a battle) iff: `id_2 ≠ id_1`, `squad_2` exists and is **offline**, its `opponents`
slot is free or already us, it's **not a task / task-giver** squad, communities are **enemies**
(`game_relations.is_factions_enemies`; mutants gated by `can_fight_mutant`), and the squared XZ distance
< `opt_dist`. **Story-squad handling:** both story → skip; exactly one story + **same `assigned_target_id`**
→ the special (non-"not-so-special") story squad **instakills** the other (each member `on_death`); else →
`simulate_battle`.

### The battle — `simulate_battle(id_1=attacker, id_2=victim, …)`
Compute both powers; if either is 0, abort. Mark `opponents` both ways. Pick a random MVP attacker + victim
NPC. **Damage = attacker_power × (HIT_BASE 0.4 + rand × HIT_FACTOR 0.2)**, ceil'd; subtract from victim
power. Then: if victim power ≤ 0 → **whole victim squad dies** (`set_battle_outcome(victory=true)` +
`remove_squad_info` + each member `on_death(attacker)`); else if accumulated damage exceeds the victim
NPC's own power → **kill that one NPC** (`set_battle_outcome(false)` + `se_victim:on_death(attacker)`,
removing the squad if it drops to ≤1). A battle is **one exchange per coordinator tick**, not resolved to
completion instantly — squads whittle down over successive 30 s cycles.

### Outcome — `set_battle_outcome(victory, …)`
`game_relations.offline_npc_on_death(se_victim, se_attacker)` (F40 — relations shift from the offline kill);
if the player's side lost, `dynamic_news_manager…:SOSBattleOffline` (F39 news); if the attacker is a stalker,
improve its **rank** by damage dealt (dominators gain) + record kill stats (verified head; the exact gain
math is spine-read past the cutoff).

### Power / cleanup / persistence
`calculate_squad_power` (Σ `calculate_npc_power`, cached unless `raw`); `get_random_npc` (random member as
MVP); `distance_to_xz_sqr` (XZ-plane squared distance); `remove_squad_info` (drop a destroyed squad from
`ocs_power`/`opponents`/level tables, + warfare removal); `save_state`/`load_state` persist `task_squads`.

## Function digests (spine)

- **`coordinator`** — *Does:* periodic driver; mode toggle + build per-level eligible-squad set + options.
- **`squad_on_update` / `smart_terrain_on_update`** — *Does:* per-squad/per-smart enemy scan → `validate_enemy`.
- **`validate_enemy`** — *Does:* gate a pair (offline, enemy, close, non-task, story rules) → `simulate_battle`.
- **`simulate_battle`** — *Does:* power → randomized damage → kill NPC or whole squad → outcome.
- **`set_battle_outcome`** — *Does:* relations shift + news + attacker rank/kill-stat gain.
- **`calculate_npc_power` / `calculate_squad_power`** — *Does:* rank-based power (monster=`m_class×m_rank_max`,
  stalker=`rank()`, × faction multiplier).
- **`init_settings`** — *Does:* load `offline_combat_simulator.ltx` (`story_communities`, `ignore_list`).

## Isolation ledger (the seams)

- **squad/sim seam (F85/F86):** reads `SIMBOARD.squads`, squad `get_squad_community`/`squad_members`/
  `npc_count`/`assigned_target_id`; kills via the server entity's `on_death` (F84). OCS does **not** use the
  online combat path — it's a pure offline resolver.
- **relations seam (F40):** `game_relations.offline_npc_on_death` applies the relation consequence + the
  `is_factions_enemies` gate; `offline_combat_simulator.ltx` `story_communities` maps story ids → community.
- **options seam (F17):** the `alife/general/offline_combat` (mode) + `alife_stalker_pop`/`alife_mutant_pop`
  (level cap scaling) options.
- **news seam (F39):** `dynamic_news_manager…SOSBattleOffline` on a player-side loss.
- **task seam (F38/F97):** task squads + task-giver squads are excluded so quests aren't broken by attrition.
- **warfare seam (F105):** `_G.WARFARE` swaps distances/exclusions/per-faction power and routes removal
  through `sim_squad_warfare`.
- **Callback bus (F1):** registers/unregisters `squad_on_update`/`smart_terrain_on_update` dynamically;
  `on_game_load`/`save_state`/`load_state`.

## Threading / failure / edge cases

- **Crowded-levels-only:** `limit_by_level` means OCS ignores levels below the pop-scaled squad cap — a
  quiet level has no offline attrition. Tuning `MAX_SQUAD_COUNT`/pop factors changes which levels simulate.
- **One exchange per tick:** a battle isn't instant; squads lose power/NPCs over successive 30 s coordinator
  cycles, so an offline war plays out gradually. `opponents[]` keeps a pair locked across ticks.
- **Story squads are protected:** both-story pairs never fight; a story squad only removes a non-story
  same-target rival (the instakill path), and "not-so-special" communities (`monster`/`zombied`) are exempt.
- **Task squads excluded:** task + task-giver squads are filtered out every coordinator rebuild — a quest
  NPC won't be killed by OCS. Persisted via `task_squads` (`save_state`).
- **Monsters gated by `can_fight_mutant`:** in normal mode mutants fight; in warfare it's an option. A
  `monster` community squad is skipped when disabled.
- **No `gamedata/` override** at writing — re-check per session; the config (`offline_combat_simulator.ltx`)
  is the main tuning surface.

## Deeper

- Simulator: `unpacked/scripts/sim_offline_combat.script` (`coordinator`, `validate_enemy`,
  `simulate_battle`, `set_battle_outcome`, `calculate_*_power`)
- Config: `unpacked/configs/misc/offline_combat_simulator.ltx` (`offline_health`, `story_communities`,
  `ignore_list`); option `alife/general/offline_combat` (F17 `flows/options-runtime.md`)
- Squad/board it consumes: `flows/scripted-squad.md` (F85), `flows/sim-board.md` (F86)
- Consequences: `flows/relations-info.md` (F40, `offline_npc_on_death`), `flows/pda-map-news.md` (F39 news)
- Map: `modlayer/gameplay/squads-simulation.md` (M5)

## Maintenance

- Verified core (the coordinator mode/level gating, `validate_enemy` pairing rules, the
  power→damage→kill battle, relation/news outcome) is stable. The exact attacker rank-gain math + warfare
  branch internals are spine-read past the main cutoff — promote when a task needs precise numbers.
- `game_relations.offline_npc_on_death` is the relations consequence (F40) — cross-check if relation rules change.
- No override at writing; the `.ltx` is the tuning surface.
