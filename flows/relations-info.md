---
domain: flows/relations-info
tags: [xrgame, relations, goodwill, reputation, community, info-portion, knowledge, flow]
sources: [src/xrGame/relation_registry.cpp, src/xrGame/relation_registry_inline.h, src/xrGame/relation_registry_actions.cpp, src/xrGame/relation_registry_fights.cpp, src/xrGame/character_community.cpp, src/xrGame/character_rank.cpp, src/xrGame/character_reputation.cpp, src/xrGame/inventory_owner_info.cpp, src/xrGame/InfoPortion.h]
related: [flows/dialog-trade.md, flows/ai-perception.md, flows/damage-hit.md, flows/save-load.md, flows/game-services.md, flows/faction-relations-lua.md, engine/server-entities.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: relations / reputation & info-portions

Two coupled knowledge/standing systems that gate most NPC interaction: **info-portions** (named
boolean knowledge flags an actor/NPC "knows") and **relations** (the goodwill→friend/neutral/enemy
computation between any two characters). Info-portions gate dialog branches (`flows/dialog-trade.md`)
and task progress; relations drive AI enemy selection (`flows/ai-perception.md`) and trade pricing.
Both are saved per object (`flows/save-load.md`).

**Provenance:** spine personally code-verified — `RELATION_REGISTRY::GetAttitude` (the 5-component
sum), `GetRelationType` (thresholds) and `GetRelation` (worst-of-both-directions),
`CInventoryOwner::OnReceiveInfo`/`TransferInfo`/`HasInfo` + the `GE_INFO_TRANSFER` event +
`KNOWN_INFO_VECTOR` storage, and that `OnReceiveInfo` fires **no engine callback**. The
character community/rank/reputation relation **tables**, `relation_registry_actions.cpp` (event
goodwill/rep/rank changes), `relation_registry_fights.cpp` (fight registry), `SetGoodwill` clamping,
the Lua script surface, and the `entity_alive` Hit→fight→Action wiring are confirmed homes but
*(agent-traced — not personally re-read)*.

## Info-portions — the knowledge flags (verified)

An info-portion is just a named flag. **`CInfoPortion`** (`InfoPortion.h`) is a thin stub today
(holding the id; the rich XML-action implementation is commented out). Each `CInventoryOwner` stores a
**`KNOWN_INFO_VECTOR`** (a `vector<shared_str>` of known info ids) in its ALife registry
(`m_known_info_registry`), keyed by object id, so it persists with the save.

- **Give:** `CInventoryOwner::TransferInfo(info_id, add=true)` packages a `GE_INFO_TRANSFER` event
  (`u_EventSend`) and locally calls `OnReceiveInfo`. **`OnReceiveInfo(info_id)`** finds the id in the
  vector (`CFindByIDPred`) and `push_back`s it if absent (else returns false) — **that is all it does.**
- **Disable:** the mirror (`add=false` → `OnDisableInfo`, removes it).
- **Check:** `HasInfo(info_id)` searches the vector.

**Critical seam fact (refined 2026-06-11):** the base `CInventoryOwner::OnReceiveInfo` fires **no
Lua / engine callback** — the old `CInfoPortion` script-action path is commented out — so for
**NPCs** an info-portion is pure storage and scripts react by **polling `has_info`** (xr_logic
conditions, dialog preconditions, task checks). **The ACTOR is the exception:**
`CActor::OnReceiveInfo` (`actor_communication.cpp`, verified) chains the base then fires the
per-object **`callback(GameObject::eInventoryInfo)`** with the info string (only when the info is
*new* — a duplicate returns false before the callback) and refreshes the open talk menu;
`bind_stalker` registers `inventory_info` → `bind_stalker_ext.actor_on_info_callback` →
`SendScriptCallback("actor_on_info_callback", obj, info_id)` on the F1 bus (consumed by e.g.
`info_portions.script` for info-switch condlists). `CActor::OnDisableInfo` fires **no** callback
(talk-menu refresh only). The Lua surface is `obj:give_info_portion`/`disable_info_portion`/
`has_info` (in `script_game_object_inventory_owner.cpp`). Info-portion **definitions** live in
config/XML loaded from gamedata (not in the C++ tree); the C++ side only stores/looks up the id.

## Relations — goodwill → attitude (verified)

`RELATION_REGISTRY` (a global singleton wrapper) computes how character `from` regards character `to`.

**`GetAttitude(from, to)`** (verified) is a **plain linear sum of five components** — no weighting:
- `personal_goodwill` = `GetGoodwill(from_id, to_id)` (per-NPC-pair stored goodwill),
- `reputation_goodwill` = `GetReputationRelation(from->Reputation(), to->Reputation())`,
- `rank_goodwill` = `GetRankRelation(from->Rank(), to->Rank())`,
- `community_goodwill` = `GetCommunityGoodwill(from->Community(), to_id)` (a faction's personal feeling
  toward one character),
- `community_to_community` = `GetCommunityRelation(from->Community(), to->Community())` (the
  faction↔faction matrix).

**`GetRelationType(from, to)`** (verified): reads two static thresholds from `[game_relations]`
(`attitude_neutal_threshold`, `attitude_friend_threshold`); `attitude < neutral → Enemy`,
`< friend → Neutral`, else `Friend` (`NO_GOODWILL → Neutral`). **`GetRelation`** (the symmetric query
used by gameplay) takes the **worst** of `rel(a→b)` and `rel(b→a)`: if either side is Enemy → Enemy,
else if either Neutral → Neutral, else Friend.

### Character attributes (agent-traced tables)

Each `CInventoryOwner` carries `character_info` with **community** (faction index — `character_community`),
**rank** (experience tier — `character_rank`), and **reputation** (`character_reputation`). Each holds a
static `m_relation_table` (loaded from `[game_relations]` matrix sections) consulted by the
corresponding `Get*Relation`. *(table contents agent-traced.)* In normal play the actor's
`character_rank` / `character_reputation` are driven by the Lua `game_statistics` layer (F107,
`flows/game-services.md`): per-event counters apply rank/rept rewards and fire band-crossing news
tips, with the rating/reputation band thresholds parsed from `creatures\game_relations.ltx`. The
Lua faction-relations layer that manipulates the community goodwill matrix and runs DoctorX dynamic
relations (kills shifting faction standings) is `flows/faction-relations-lua.md` (F108).

### Relation changes on events (agent-traced)

`relation_registry_actions.cpp` `Action(from, to, type)` adjusts goodwill/reputation/rank on **attack**,
**kill** (killer gains rank via `CHARACTER_RANK::rank_kill_points`, group goodwill shifts), and
**fight-help** events, using deltas from the `[action_points]` config. `relation_registry_fights.cpp`
maintains a transient `FIGHT_VECTOR` ("who am I fighting and how hard") pruned after a configured
remember-time, giving combat context for those adjustments. These are driven from
`CEntityAlive::Hit` (→ `FightRegister` + `Action(ATTACK)`) and `Die` (→ `Action(KILL)`). *(bodies
agent-traced; the death/hit wiring pairs with `flows/damage-hit.md`.)*

## Isolation ledger (the seams)

- **Dialog/trade:** dialog preconditions gate phrases on `HasInfo` and set infos via `TransferInfo`
  (`flows/dialog-trade.md`); goodwill/relation feeds the trade discount + the price action-factor.
- **AI enemy selection:** `CAI_Stalker::tfGetRelationType` → `GetRelationType` classifies targets
  friend/enemy → drives the enemy manager (`flows/ai-perception.md`).
- **Tasks:** task availability/completion is gated on info-portions (the planned tasks flow) — scripts
  poll `has_info`.
- **Combat:** `CEntityAlive::Hit`/`Die` → fight registry + relation `Action` (rep/rank/goodwill shifts).
- **Save/load:** `KNOWN_INFO_VECTOR` and the relation/goodwill data are per-object ALife registry
  containers serialized with the save (`flows/save-load.md`).
- **Lua surface** *(agent-traced)*: `give_info_portion`/`disable_info_portion`/`has_info`;
  `set_goodwill`/`get_goodwill`/`change_goodwill`/`force_set_goodwill`, `set_relation`/`relation`/
  `general_goodwill`, community/rank/reputation getters & setters. **One-way for NPC info** (no
  engine callback — poll `has_info`); actor-side new infos fire `actor_on_info_callback` (see seam
  fact above).
- **Config:** `[info_portions]` (`files=` → info XML/LTX, gamedata); `[game_relations]`
  (`communities`, `attitude_*_threshold`, the `communities_relations`/`rank_relations`/
  `reputation_relations` matrices, `rank_kill_points`); `[action_points]` (event goodwill/rep deltas).

## Threading

Main thread; the registries are global game state mutated from gameplay/Lua. Single-threaded.

## Failure & edge cases

- **No `on_info` callback for NPCs:** giving an info-portion to an NPC does *not* notify scripts;
  logic that must react polls `has_info` each tick (or checks it in a precondition). **For the actor
  it does fire** — `eInventoryInfo` → `actor_on_info_callback` bus event (new infos only; disable
  never fires). A mod expecting an NPC-side event, or an actor-side event on *disable* or on a
  *duplicate* give, will silently never fire.
- **Linear attitude, no weighting:** one large component (e.g. a big personal-goodwill set) can flip the
  whole relation; there is no normalization. `force_set_goodwill` / `set_relation` from script override
  the computed value.
- **Worst-of-both relation:** `GetRelation` is symmetric-pessimistic — if *either* side sees the other
  as enemy, the pair is hostile. One-sided goodwill doesn't make peace.
- **Stub `CInfoPortion`:** the rich info-portion (dialog/article lists, script actions) is commented
  out; only the id-set behavior exists. Don't document the dead path as live.
- **Thresholds are static-once:** `GetRelationType` caches the thresholds at first call — changing them
  mid-session via config won't take effect.

## Deeper

- Relations: `src/xrGame/relation_registry_inline.h` (`GetAttitude`, `GetRelationType`, `GetRelation`),
  `relation_registry.cpp` (`GetGoodwill`/`SetGoodwill`/`ChangeGoodwill`, `GetCommunityRelation`),
  `relation_registry_actions.cpp` (`Action`), `relation_registry_fights.cpp` (`FightRegister`)
- Character attributes: `src/xrGame/character_community.cpp`, `character_rank.cpp`,
  `character_reputation.cpp` (each `::relation` + `m_relation_table`)
- Info-portions: `src/xrGame/inventory_owner_info.cpp` (`TransferInfo`, `OnReceiveInfo`, `OnDisableInfo`,
  `HasInfo`), `InfoPortion.h` (`CInfoPortion` stub), `InfoPortionDefs.h` (`KNOWN_INFO_VECTOR`)
- Lua surface: `src/xrGame/script_game_object_inventory_owner.cpp`, `script_game_object_use.cpp`
- Consumers: `flows/dialog-trade.md`, `flows/ai-perception.md`, `flows/damage-hit.md`,
  `flows/save-load.md`

## Maintenance

- Keep the refined info-callback fact prominent (NPCs: poll `has_info`; actor: `eInventoryInfo` →
  `actor_on_info_callback`, new infos only); it is the most important and most-misunderstood seam.
- Re-grep `RELATION_REGISTRY::GetAttitude` if the standing computation changes — keep the 5-component
  linear sum exact; it is what every friend/enemy decision rests on.
- Promote the *(agent-traced)* character relation tables, `Action`/`FightRegister`, and the Lua surface
  to verified when a faction-tuning or reputation task needs the exact deltas/matrices.
