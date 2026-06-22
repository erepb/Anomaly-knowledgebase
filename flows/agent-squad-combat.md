---
domain: flows/agent-squad-combat
tags: [xrgame, ai, agent-manager, squad, combat, enemy-distribution, flow]
sources: [src/xrGame/agent_manager.cpp, src/xrGame/agent_manager.h, src/xrGame/agent_member_manager.cpp, src/xrGame/agent_enemy_manager.cpp, src/xrGame/agent_corpse_manager.cpp, src/xrGame/agent_location_manager.cpp, src/xrGame/danger_manager.cpp, src/xrGame/group_hierarchy_holder.cpp]
related: [flows/stalker-execution.md, flows/ai-perception.md, flows/ai-planner.md, flows/ai-squad-logic.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: agent / squad combat coordination

How a **group of stalkers coordinates combat** — pooling each member's enemy sightings into a shared
picture and assigning members to enemies, cover, corpses, and grenade threats. This is the group layer
above the per-NPC planner (`flows/ai-planner.md`), perception (`flows/ai-perception.md`), and execution
(`flows/stalker-execution.md`). Note: in Anomaly, high-level squad *spawning/movement* is the Lua
`sim_squad`/smart_terrain layer (`flows/ai-squad-logic.md`); this C++ `CAgentManager` is the **online
tactical combat** coordinator for a group that's loaded in the level.

**Provenance:** spine personally code-verified — `CAgentManager` owns the sub-managers
`m_corpse`/`m_enemy`/`m_explosive`/`m_location`/`m_member`/`m_memory` (+ a group planner), and
`CAgentEnemyManager::fill_enemies`/`assign_enemies`/`distribute_enemies` exist; **there is no danger
sub-manager** (danger is per-NPC). The per-group ownership via the seniority hierarchy, the
member-manager combat mask, the enemy pooling/assignment math, the corpse/explosive/location reactions,
and the (inert) offline combat manager are confirmed homes but *(agent-traced — not personally
re-read)*.

## What the agent manager is (ownership verified)

`CAgentManager` is created **per online group** (one per team→squad→group in the seniority hierarchy,
`group_hierarchy_holder`), on demand when the first stalker of a group registers and destroyed when the
last leaves. Each stalker reaches it via `CAI_Stalker::agent_manager()`. It owns (verified) six
sub-managers + a group planner, updated in order (memory → corpse → enemy → explosive → location →
member → brain):
- **`m_member`** (`CAgentMemberManager`) — the squad roster + combat state;
- **`m_memory`** (`CAgentMemoryManager`) — pooled perception (visibles/sounds/hits) with per-member
  masks;
- **`m_enemy`** (`CAgentEnemyManager`) — the shared enemy list + assignment;
- **`m_corpse`/`m_explosive`/`m_location`** — squad-wide corpse, grenade, and cover/danger-zone
  reactions.

## Member manager (agent-traced)

`CAgentMemberManager` is the roster of living stalkers in the group, indexed so each member maps to a
**bit in a 64-bit `squad_mask_type`** (`mask(stalker)` = `1<<index`). It tracks per-member combat data
in a `CMemberOrder` (assigned enemies/selected enemy, detour/flank state, assigned cover point, corpse/
grenade reaction) and a **combat mask** (`register_in_combat`/`registered_in_combat`/`combat_members`)
of who is actively fighting. The mask is how the whole system addresses "which members" cheaply.

## Shared enemy pooling & assignment (entry verified; math agent-traced)

The core mechanic — one member's sighting informs the squad:
1. each stalker keeps its own enemy list from its perception (`flows/ai-perception.md`).
2. **`CAgentEnemyManager::fill_enemies`** (verified) iterates combat members and merges their seen
   enemies into the squad's `m_enemies` (a `CMemberEnemy` per enemy tagged with `m_mask` = which members
   see it).
3. danger scoring evaluates each member vs each enemy (a victory-probability EF pattern).
4. **`assign_enemies`** (verified) greedily assigns each enemy to the member(s) with the best odds
   (`m_distribute_mask`), and `permutate_enemies` writes each member's `selected_enemy`.
5. **`distribute_enemies`** (verified) is the orchestrator (called from the group planner's kill-enemy
   action). Each member's planner then reads its `selected_enemy` and fights it
   (`flows/stalker-execution.md`).

`CAgentMemoryManager` similarly pools visibles/sounds/hits (the group's shared perception arrays), with
masks marking which member contributed each percept.

## Corpse / explosive / location reactions (agent-traced)

- **`CAgentCorpseManager`** — registers dead members and assigns the nearest living member to "react"
  (mourn/loot) via its `CMemberOrder`; the member's own planner executes the reaction.
- **`CAgentExplosiveManager`** — registers live grenades as squad danger zones and assigns members to
  react (take cover/flee).
- **`CAgentLocationManager`** — manages tactical cover positions / danger zones, scoring cover points
  (`suitable`/`danger`) so members don't crowd the same spot and avoid known dangers — the squad's
  position distribution.

## Danger is per-NPC (verified by absence)

`CDangerManager` is **not** an agent sub-manager (verified: not in the `CAgentManager` member list).
Each `CCustomMonster`/stalker has its **own** danger memory (bullet ricochet, attack sound, ally
death/corpse, grenade, enemy sound) and reacts individually — the squad coordinates *enemies/corpses/
grenades/positions*, but immediate danger reactions are per-unit.

## Offline (verified inert)

`CALifeCombatManager` is largely commented-out / inert (only `kill_entity` bookkeeping is live, also
noted in `flows/alife-services.md`); offline squad combat is not C++-simulated here — it's the Lua
`sim_squad`/ALife layer (`flows/ai-squad-logic.md`).

## member sees enemy → pool → distribute (sequence)

1. stalker A's perception sees enemy E (`flows/ai-perception.md`).
2. the group planner's kill-enemy action calls `agent_manager().enemy().distribute_enemies()`.
3. `fill_enemies` merges A's enemies into the squad list (E tagged with A's mask).
4. scoring + `assign_enemies` assign E (and others) to the best-positioned members.
5. `permutate_enemies` sets each member's `selected_enemy`; members fight their assigned target
   (`flows/stalker-execution.md`).

## Isolation ledger (the seams)

- **Per-NPC planner/execution:** members read their assigned enemy/cover/corpse/grenade from
  `CMemberOrder` and act via their own GOAP + execution (`flows/ai-planner.md`,
  `flows/stalker-execution.md`).
- **Perception:** member sightings feed the pool via `fill_enemies`/memory masks
  (`flows/ai-perception.md`).
- **Smart cover / positions:** `CAgentLocationManager` distributes cover points (`flows/smart-cover.md`
  for the cover objects themselves).
- **Lua:** **none in the agent manager** — it is pure C++ online coordination. The Lua squad layer
  (`sim_squad`/smart_terrain, `flows/ai-squad-logic.md`) handles squad spawning/macro-movement, not this
  tactical pooling.
- **Config:** none direct; behavior is driven by member perception + the victory-probability scoring.

## Threading

Main thread (group update on the scheduled channel). Single-threaded with per-NPC AI.

## Failure & edge cases

- **Agent manager is per online group:** it exists only while a group's stalkers are loaded; offline,
  the Lua `sim_squad` layer is in charge (`flows/ai-squad-logic.md`). Don't look for offline tactical
  combat in C++.
- **Mask-indexed, 64-member cap:** the `squad_mask_type` is 64-bit, so a group is capped at 64 tracked
  members; the mask is how all assignment works.
- **Danger ≠ squad-coordinated:** grenades/corpses/enemies are pooled, but immediate danger reactions
  (ricochet/sound/hit) are per-NPC (`CDangerManager`) — not distributed by the agent manager.
- **Assignment is victory-probability greedy:** members are sent at enemies they're best positioned to
  beat — odd target focus is the scoring, not a bug.
- **Lua supersedes at the macro level:** Anomaly's squad behavior (where a squad goes, who it fights at
  the strategic level) is `sim_squad` Lua; the agent manager only coordinates the loaded firefight.

## Deeper

- Manager + ownership: `src/xrGame/agent_manager.cpp`/`.h` (`CAgentManager`, sub-managers, `update_impl`),
  `group_hierarchy_holder.cpp` (per-group create/destroy, `register_in_agent_manager`)
- Members: `src/xrGame/agent_member_manager.cpp` (roster, combat mask), `member_order.h` (`CMemberOrder`)
- Enemy pooling: `src/xrGame/agent_enemy_manager.cpp` (`fill_enemies`, `assign_enemies`,
  `distribute_enemies`), `member_enemy.h` (`CMemberEnemy`)
- Reactions: `src/xrGame/agent_corpse_manager.cpp`, `agent_explosive_manager.cpp`,
  `agent_location_manager.cpp`, `agent_memory_manager.cpp`
- Per-NPC danger: `src/xrGame/danger_manager.cpp` (`CDangerManager`), `danger_object.h`
- Offline (inert): `src/xrGame/alife_combat_manager.cpp` — `flows/alife-services.md`
- Members' execution: `flows/stalker-execution.md`; macro squad layer: `flows/ai-squad-logic.md`

## Maintenance

- Keep the "per online group, mask-indexed; pools enemies/corpses/grenades/positions; danger is
  per-NPC; macro squad behavior is Lua `sim_squad`" facts — they bound what this C++ layer does.
- Re-grep `CAgentManager` sub-manager list / `CAgentEnemyManager::distribute_enemies` if the
  coordination changes.
- Promote the *(agent-traced)* enemy-assignment math + corpse/explosive/location bodies to verified
  when a squad-combat-tuning task needs the exact scoring/assignment logic.
