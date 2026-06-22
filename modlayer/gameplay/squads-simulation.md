---
domain: modlayer/gameplay/squads-simulation
tags: [modlayer, gameplay, unpacked, lua, alife, squad, simulation, map]
sources: [unpacked/scripts/sim_squad_scripted.script, unpacked/scripts/sim_board.script, unpacked/scripts/simulation_objects.script, unpacked/scripts/sim_offline_combat.script, unpacked/scripts/sim_squad_bounty.script, unpacked/scripts/sim_squad_warfare.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/scripted-squad.md, flows/smart-terrain-core.md, flows/gulag-jobs.md, flows/alife-online-offline.md, flows/alife-services.md, flows/ai-squad-logic.md, flows/se-others.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Squad simulation (`sim_*`) — L1 map

The **macro ALife layer**: how groups of NPCs (squads) decide *where to go* across the offline world.
Where the scheme engine (F66) + smart_terrain/gulag (F72/F73) decide what an NPC does *at* a location,
this layer decides which location a squad heads to. The pieces: the **squad server class** (`sim_squad_scripted`,
F85), the **board** that supplies + scores targets and assigns squads to smarts (`sim_board`, F86), the
**target registry/scorer** (`simulation_objects`), and **offline combat** resolution between squads that
never come online (`sim_offline_combat`, F87). This map routes the family; mechanics live in the flows.

**Read first:** `flows/scripted-squad.md` (F85) — the squad class + its update→target→assign loop. Always
grep `gamedata/scripts/` for an override; **`sim_squad_scripted` HAS a runtime override** (the assignment
path is routed through `sim_board`).

## The pieces (verified by listing + F85)

| File | Role | Flow |
|------|------|------|
| `sim_squad_scripted` | the squad server class (`cse_alife_online_offline_group` subclass); update loop, target FSM, members, switch, target interface | F85 ✅ (runtime override in gamedata) |
| `simulation_objects` | target registry (`object_registry`/`available_by_id`) + `evaluate_prior` scorer + level-link checks + props from `simulation_objects_props.ltx` | F85 ✅ (covered with the squad) |
| `sim_board` | the `simulation_board` singleton (`SIMBOARD`/`get_sim_board`): smart registry, `get_squad_target`, `assign_squad_to_smart`, `setup_squad_and_group`, `general_squad_precondition`, start positions | F86 ✅ |
| `sim_offline_combat` | resolves combat between offline squads (no online objects) via `offline_combat_simulator.ltx` | F87 ✅ |
| `sim_squad_bounty` | **separate manager** — spawns bounty hit-squads on high rank/bad reputation (NOT a squad subclass) | F105/tasks-adjacent ⬜ |
| `sim_squad_warfare` | **separate manager** — warfare-metagame squad helper (NOT a subclass) | F105 ⬜ |

## How it fits (verified in F85)

1. Each squad `update` calls `get_script_target` → if a scripted/condlist/random/hostage target exists →
   `specific_update`; else `generic_update` asks `SIMBOARD:get_squad_target` for the best candidate.
2. `SIMBOARD:get_squad_target` (F86) ranks sim objects with `simulation_objects.evaluate_prior` (type gates →
   props × behaviour weights × 1/graph-distance).
3. On arrival the squad assigns its members to the target smart's job board (F72/F73) — at runtime via
   `sim_board:assign_squad_to_smart` (the gamedata delta).
4. The squad is itself a sim target (implements the F84 `se_actor` target interface) so other squads can path
   to it.
5. `check_online_status` decides `forced_online_status`, honored by `can_switch_*` for the F16 ALife switch.

## Subsystem deep-dives (delegates to flows)

- Squad class + target scoring → `flows/scripted-squad.md` (F85) ✅
- The board (target supply, smart registry, assignment, world population) → `flows/sim-board.md` (F86) ✅
- Offline combat → `flows/offline-combat.md` (F87) ✅
- Smarts/jobs members land on → `flows/smart-terrain-core.md` (F72), `flows/gulag-jobs.md` (F73)
- ALife online/offline + services → `flows/alife-online-offline.md` (F16), `flows/alife-services.md` (F41)

## Maintenance

- This map is the squad-sim routing hub; `ROADMAP-gameplay.md` (Phase G3, F85–F87) is the authoritative
  plan/status. **F85+F86+F87 all done — the squad-simulation domain is fully documented** (only the
  separate bounty/warfare managers route to F105).
- `sim_squad_bounty`/`sim_squad_warfare` are managers, not subclasses — keep them under the warfare/tasks
  domains, not here.
- Re-check `gamedata/scripts/sim_*` for overrides (currently `sim_squad_scripted` is overridden).
