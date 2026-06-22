---
domain: modlayer/gameplay/warfare
tags: [modlayer, gameplay, unpacked, lua, warfare, faction-war, map]
sources: [unpacked/scripts/warfare.script, unpacked/scripts/smart_terrain_warfare.script, unpacked/scripts/sim_squad_warfare.script, unpacked/scripts/warfare_options.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/warfare-metagame.md, flows/scripted-squad.md, modlayer/gameplay/squads-simulation.md, flows/smart-terrain-core.md, flows/offline-combat.md, flows/helicopter-ai.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Warfare metagame — L1 map (M18)

The optional faction-war mode: a Lua simulation over the stock squad/smart sim where smarts have
owners, factions have economies, and the player commands. Mechanics live in
`flows/warfare-metagame.md` (F105). Activation root is `_g.IsWarfare()` (new-game option or
persisted `enable_warfare_mode` flag, set by `ui_mm_faction_select`).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| hub | `warfare.script` | unconditional callback registration + reactive `resetter`, actor-faction tracking, level info build, persistence (`m_data.warfare`) | F105 ✅ |
| smart layer | `smart_terrain_warfare.script` | per-smart tick: capture (`check_owner`), defense/patrol/invasion spawning, mutants on unowned smarts, fog-of-war map spots (replaces `se_smart_terrain:show/hide`) | F105 ✅ |
| squad layer | `sim_squad_warfare.script` | per-squad tick: `squad.__lock` suppression of stock AI, target/arrival state machine, kill rewards (money + influence) | F105 ✅ |
| faction tick | `warfare_factions.script` | per-faction economy lerp (respawn/invasion knobs from base/resource ratios), resurgence of eliminated factions | F105 ✅ |
| level data | `warfare_levels.script` | per-level smart/faction stats + cross-level invasion target pick | F105 ✅ (agent-traced) |
| player commands | `warfare_faction_control.script` | PDA command deck: buy squads/heli (money), orders, manual capture, possess-leader body swap — *not* a faction AI despite the name | F105 ✅ |
| options & names | `warfare_options.script`, `warfare_names.script` | the `alife/warfare/*` ui_options knob surface (incl. dead `override_functions`); display names + hidden smarts | F105 ✅ |
| UI | `ui_pda_warfare_tab.script`, `pda_smart_terrain_warfare.script`, `ui_warfare_options_hints.script` | PDA stats tab, influence staking on smarts, option hints | F105 ✅ (agent-traced) |

## How it fits

All simulation rides F65 bus callbacks (`smart_terrain_on_update`, `squad_on_update`,
`actor_on_update`) — no engine seams. Stock squad AI is suppressed per squad via the native
`__lock` honored by `sim_squad_scripted.generic_update`; stock smart map spots are replaced by
method override. Fights resolve through the normal offline-combat path; squad spawn sections come
from `faction_expansions`. `_G.WARFARE` is the cheap cross-module flag (game_relations,
axr_companions, PDA tabs).

## Subsystem deep-dives (delegates to flows)

- Activation, lock mechanism, capture rules, economy tick, commander features, verified bugs
  (linked-level density `level`/`lvl` typo, `patrol_squads` nil store, dead `override_functions`
  + stranded heli update swap) → `flows/warfare-metagame.md` (F105) ✅
- Stock squad simulation being locked out → `flows/scripted-squad.md` +
  `modlayer/gameplay/squads-simulation.md`
- Smart terrain machinery → `flows/smart-terrain-core.md`
- Offline fight resolution → `flows/offline-combat.md`
- Heli purchase/update split → `flows/helicopter-ai.md` (F95)

## Maintenance

- F105 ✅. No `gamedata/` override of any warfare file at writing; re-check on pickup. Promote
  the *(agent-traced)* invasion/patrol state machines and fog-of-war spot logic when a task
  touches them. `faction_expansions.script` (spawn compositions) is unread — document it if a
  task needs spawn tables.
