---
domain: flows/faction-relations-lua
tags: [modlayer, gameplay, unpacked, lua, relations, goodwill, community, faction, dynamic-relations, flow]
sources: [unpacked/scripts/game_relations.script, unpacked/configs/creatures/game_relations.ltx, unpacked/configs/plugins/dynamic_faction_relations.ltx, src/xrGame/relation_registry.cpp, src/xrGame/relation_registry_actions.cpp]
related: [flows/relations-info.md, flows/game-services.md, flows/offline-combat.md, flows/warfare-metagame.md, flows/dialog-system-lua.md, flows/condlist-library.md, flows/lua-callback-lifecycle.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: faction relations (Lua) (F108)

The Lua faction-relations layer — **DoctorX Dynamic Faction Relations** (`game_relations.script`,
1044 lines, READ-IN-FULL; no `gamedata/` override) plus the `creatures\game_relations.ltx` data
tier it shares with the engine. It is the *gameplay* side of the relation system; the engine
attitude math lives in F40 (`flows/relations-info.md`). This file does three things: (1) expose the
getter/setter API the whole corpus uses to query and set goodwill, (2) seed and persist the
faction-to-faction matrix, and (3) run the **dynamic-relations** event system that shifts faction
standings when NPCs kill each other.

## The goodwill model (engine-owned, documented here)

Overall goodwill of stalker A toward stalker B is a **5-component sum** computed by the engine
(F40), each term configured in `game_relations.ltx`:

1. **personal_goodwill** — raw A→B (`set_relation`/`force_set_goodwill`/`change_goodwill`).
2. **reputation_goodwill** — A's reputation vs B's, static from `[reputation_relations]`.
3. **rank_goodwill** — A's rank vs B's, static from `[rank_relations]`.
4. **community_goodwill** — A's faction toward B personally (used by trade stock upgrades).
5. **community_to_community** — A's faction toward B's faction, from `[communities_relations]`.

Enemy/friend cutoffs are `attitude_neutal_threshold` (−999) and `attitude_friend_threshold`
(999) in `[game_relations]`; `set_relation(type, …)` maps to `goodwill_enemy/neutal/friend`
(±1000/0). The Lua constants `FRIENDS=1000 / NEUTRALS=0 / ENEMIES=-1000` are the
**community_relation** thresholds (component 5), distinct from the personal-goodwill cutoffs. The
script's own tutorial block (end of file) is the authoritative API cheat-sheet — note two of its
section labels are swapped (it says reputation_goodwill comes from `[rank_relations]` and
rank_goodwill from `[reputation_relations]`; the code reads them the right way round).

## The API surface (what the corpus actually calls)

A thin wrapper layer over engine `relation_registry` + `game_object` methods. The widely-used
predicates — `is_factions_friends` / `is_factions_enemies` / `is_factions_neutrals` (community
matrix), `is_squad_enemy/friend/neutral_to_actor`, `get_squad_goodwill_to_actor(_by_id)`,
`get_gulag_relation_actor` — are consumed by ~35 scripts (condlists, dialogs, schemes, warfare,
tasks). Setters bridge each goodwill tier:

- **community↔community**: `set_factions_community(_num)` → `relation_registry.set_community_relation`;
  `get_factions_community` → `community_relation`.
- **faction→stalker**: `change_factions_community_num` → `change_community_goodwill`;
  `set_community_goodwill_for_faction`, `set_level_faction_community` (a deferred
  `temp_goodwill_table` applied per-object, the home of the `xr_effects.set_level_faction_community`
  run-string).
- **stalker↔stalker / squad**: `set_npcs_relation` (`force_set_goodwill`), `set_npc_sympathy`,
  `set_squad_goodwill` (`squad:set_squad_relation`), `set_squad_goodwill_to_npc`,
  `set_squad_community_goodwill`, `set_gulag_relation_actor`.

## Seeding & persistence

`reset_all_relations` reads every row of `[communities_relations]` and pushes it into the engine
via `set_factions_community_num`. `on_game_load` (registered on `on_game_load`, MARSHAL only):
on an established save it `load_relation`s each faction pair from `save_var` keys
(`drx_df_<f1>_<f2>_relations`, stored per-pair and per `actor_<faction>` variant); on a brand-new
game (the `alife_storage_manager` `new_game_relations` flag unset) it calls `reset_all_relations`
and sets the flag. `save_relation` writes both directions plus the `actor_`-prefixed variants and
persists them via `save_var(db.actor, …)`.

## Dynamic relations (the event engine)

The headline DoctorX feature: faction standings drift when NPCs kill each other. Registered on
`npc_on_death_callback` (`online_npc_on_death`) and called from the offline simulator
(`offline_npc_on_death`, invoked by `sim_offline_combat`, F87). Both build `{comm, rank, rep}`
tables for victim and killer (online from `db.storage` objects, offline from server objects +
`ranks`/`utils_obj` helpers), skip non-humans (`monster`/`zombied`) and blacklisted factions, and
call `calculate_relation_change`. Gated globally by the `alife/general/dynamic_relations` option.

`calculate_relation_change` is a weighted, randomized propagation:

- If the victim was an **enemy** of faction X → X's relation toward the **killer's** faction rises;
  if the victim was a **friend/neutral** of X → X's relation toward the killer's faction falls.
- Magnitude = `death_value × (victim_rank + killer_rank/5) × (victim_rep ± killer_rep/10)`, where
  rank weights come from a local `rank_table` (novice 0.1 → legend 3) and reputation weights from
  `rep_enemy_table` / `rep_friend_table` (only critically bad/good reputations deviate from 1.0).
  Killer reputation only counts when the victim's reputation was itself remarkable. `give_range`
  adds per-term random jitter.
- **Count limits** (`friend_count_limit` / `enemy_count_limit`, default 3) suppress changes when a
  faction already has too many allies (for positive shifts) or enemies (for negative), damping
  runaway escalation. In **warfare** mode (F105) these limits drop to −1 (always allow), unless
  `fog_of_war` is on, in which case dynamic relations are skipped entirely.

`change_faction_relations` applies the delta to the community matrix, clamps to `friend_limit`
(4100) / `enemy_limit` (−5000) — with a coin-flip reset-to-0 when pegged — saves all directions,
and fires `news_manager` tips on war-footing transitions (declared war / alliance / deteriorating /
cease-fire), resetting the actor's community goodwill only when factions turn hostile (gated by the
`war_goodwill_reset` option).

## Config: `game_relations.ltx` is shared, not Lua-private

This LTX is read by the engine, this script, **and** other systems — editing it affects all of
them:

- `[game_relations]` — `rating` / `reputation` band thresholds (the rank/reputation tier ladder
  read by game_statistics, F107, for level-up news tips), `attitude_*_threshold`, `goodwill_*`,
  the `communities` name→index map, `monster_rating`.
- `[communities_relations]` — the 31-wide faction-vs-faction starting matrix (engine loads it
  natively at boot; `reset_all_relations` re-pushes it on new game). `[rank_relations]` /
  `[reputation_relations]` — the static rank/rep goodwill matrices (engine sums + this script's
  `get_rank_relation` / `get_reputation_relation`). `[rank_kill_points]` — rank points awarded for
  a kill (engine `relation_registry_actions`, F40; the kill-reward source game_statistics defers
  to). `[action_points]` — per-action goodwill/reputation deltas + goodwill clamps (engine F40).
- `[communities_sympathy]` — goodwill-spread coefficients (all 0.0 in vanilla = disabled).
- `[ranks_properties]` — engine-referenced rank interpolation coeffs; its own comment warns
  removing it crashes the game though "the section is not used by the game" logically.
- `[smart_terrain_rank_change]` — rank bump on smart registration (F72). `[monster_communities]`
  / `[monster_relations]` — the monster faction index + monster-vs-monster matrix (F34/F94).

## Verified bugs & quirks

- **`factions_table_all` order does not match the shipped `[communities_relations]`.** The Lua list
  is 30 entries ordered `actor, bandit, dolg, ecolog, …`; the LTX matrix is 32 rows/columns ordered
  `actor, monster, trader, army_npc, greh_npc, bandit, …` (and includes `army_npc`/`greh_npc`,
  which the Lua list omits). The script's own ATTENTION comment requires them to match because
  `reset_all_relations` indexes columns *positionally* (`tbl[fta[i]][j]` assigned to
  `relation(fta[i], fta[j])`). So the new-game Lua reseed maps mismatched cells (e.g. it reads the
  `monster` column when assigning the `bandit` pair) and does not reproduce the matrix as authored.
  This is a file-level comparison fact; the in-world effect (the reseed overwrites the engine's
  native load on a new game) was not runtime-observed.
- **`reset_goodwill` precedence bug.** `character_community(db.actor) == faction_1 or ("actor_" ..
  faction_1)` parses as `(== faction_1) or "string"` — the trailing string literal is always
  truthy, so the first branch always runs regardless of the actor's community; faction_2's goodwill
  is reset and faction_1's never is.
- **`friend_num` dead branch.** In `calculate_relation_change` the faction-tally loop has two
  identical `elseif (rel > -1000) and (rel < 1000)` arms, so the `friend_num` arm is unreachable —
  friends are counted as `natural_num`. Only affects the `natural_num + friend_num` count-limit
  test, so the practical effect is friends count toward the friend-limit as naturals.
- **`set_squad_goodwill_to_npc` reads an undefined global.** It computes `tonumber(new_relation)`
  but the parameter is `new_goodwill`; `new_relation` is nil, so the goodwill always resolves to 0
  (neutral) — the string forms "enemy"/"friend" never take effect.
- `get_squad_relation_to_actor_by_id` returns `"friends"` (plural) where its siblings return
  `"friend"`, and dereferences `squad.player_id` without a nil guard if the squad is missing.

## Deeper

- Engine attitude math, `relation_registry`, info-portion knowledge → `flows/relations-info.md` (F40).
- Rank/reputation band ladder this LTX feeds (level-up tips, rewards) → `flows/game-services.md` (F107).
- The offline death feed → `flows/offline-combat.md` (F87); warfare overrides → `flows/warfare-metagame.md` (F105).
