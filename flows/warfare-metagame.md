---
domain: flows/warfare-metagame
tags: [modlayer, gameplay, unpacked, lua, warfare, faction-war, metagame, simulation, flow]
sources: [unpacked/scripts/warfare.script, unpacked/scripts/warfare_faction_control.script, unpacked/scripts/warfare_factions.script, unpacked/scripts/warfare_levels.script, unpacked/scripts/warfare_names.script, unpacked/scripts/warfare_options.script, unpacked/scripts/smart_terrain_warfare.script, unpacked/scripts/sim_squad_warfare.script, unpacked/scripts/pda_smart_terrain_warfare.script, unpacked/scripts/ui_pda_warfare_tab.script, unpacked/scripts/_g.script, unpacked/scripts/ui_mm_faction_select.script]
related: [flows/ai-squad-logic.md, flows/smart-terrain-core.md, flows/scripted-squad.md, flows/offline-combat.md, flows/helicopter-ai.md, flows/lua-callback-lifecycle.md, flows/relations-info.md, flows/alife-services.md, modlayer/gameplay/warfare.md, modlayer/gameplay/squads-simulation.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: warfare metagame (F105)

The optional faction-war mode: a Lua simulation layered on the stock squad/smart sim that gives
every smart terrain an owner, every faction an economy of bases/resources/respawns, and the
player a commander seat (buy squads/helis, order attacks, spend influence). ~7.2K lines across
12 `warfare*`/`*_warfare` files. **No `gamedata/` override of any warfare file** (checked this
session).

**Provenance:** agent-researched draft, spine personally re-verified: `IsWarfare()` root,
`warfare.on_game_start` + `resetter`, `actor_on_update` head, `squad.__lock` suppression chain
into `sim_squad_scripted`, `override_functions` dead-code proof (+ its heli consequence),
`check_owner` body, `squad_on_npc_death` rewards, `update_faction` locals, both flagged bugs
re-read in context, `warfare_faction_control` head + possess-leader body + purchase costs,
`ui_mm_faction_select` flag write. The faction/level tick internals, invasion/patrol state
machines, fog-of-war map-spot logic, and UI bodies are *(agent-traced with line evidence)*.
**Two agent claims corrected** (registration conditionality; the suppression mechanism — see
below).

## Activation — optimistic registration, reactive dormancy

The root is `_g.IsWarfare()`: true when the new-game option `new_game_warfare` is set in
`axr_main.config` (character creation) **or** `alife_storage_manager` carries
`enable_warfare_mode` — written by `ui_mm_faction_select` when the player picks Warfare at
faction select. There is no separate game-mode module; warfare *is* these scripts.

The wiring is the surprising part (verified): `warfare.on_game_start` registers **all** warfare
callbacks unconditionally — `smart_terrain_on_update`, `squad_on_update`, `actor_on_update`,
`squad_on_npc_death`, `fill_start_position`, `save_state`/`load_state`,
`server_entity_on_unregister` — and optimistically sets `_G.WARFARE = true` in every game.
Dormancy is reactive: each callback early-exits on `not IsWarfare()`, and 60 seconds after
`on_game_load` a `resetter` TimeEvent unregisters the whole set (and clears `_G.WARFARE`) if
warfare is off. So for the first minute of any non-warfare session the warfare handlers run
no-op passes on every smart/squad update. `_G.WARFARE` is the cheap flag other modules test
(`game_relations`, `axr_companions`, PDA tabs).

## Integration with the stock sim — locks, not subclasses

Nothing subclasses. Three mechanisms (all verified):

- **Per-squad lock.** Every `squad_on_update` tick, warfare sets `squad.__lock = true` (and
  clears `scripted_target`) on each non-companion squad — `sim_squad_scripted.generic_update`
  natively honors `__lock` and returns, so stock squad AI is suppressed squad-by-squad while
  warfare drives `current_target_id`/`assigned_target_id`/`current_action` itself
  (0 = en route, 1 = arrived; `rush_to_target` while moving).
- **Method replacement on the smart-terrain class.** `smart_terrain_warfare` replaces
  `se_smart_terrain:show()`/`hide()` wholesale — map spots become faction-colored,
  fog-of-war-gated warfare icons.
- **Bus callbacks**, not engine seams: all simulation runs off `smart_terrain_on_update` /
  `squad_on_update` / `actor_on_update` (F65).

**Dead code with consequences (verified):** `warfare_options.override_functions` — which would
null stock `generic_update`/`specific_update` and hot-swap `heli_alife.update` →
`heli_alife_warfare.update` — has **no caller anywhere**. The squad nulling is redundant (the
`__lock` covers it), but the heli swap never happening means helicopters bought through the PDA
are *created* by `heli_alife_warfare` (`create_heli_weak`/`set_target` are called directly) yet
*updated* by stock `heli_alife` logic (`flows/helicopter-ai.md` F95).

## The simulation tick

Three nested cadences *(agent-traced, heads verified)*:

- **`warfare.actor_on_update`** (10 s): maintains `actor_faction` (faction switches, azazel),
  first-update builds `warfare_levels.level_information` (per-level smart lists, per-faction
  base/resource/territory/lair counts, priorities), applies all-out-war relations if enabled,
  ticks `warfare_levels.update()` + `warfare_factions.update()`, watches surges.
- **`warfare_factions.update_faction`** (random 1–5 min per faction): aggregates per-faction
  stats (bases, resources, power, active/inactive/patrol squads), lerps the economy knobs from
  resource/base ratios — defense respawn time, patrol time, invasion size, patrol population
  factor — applies enemy-of-actor boosts, and runs **resurgence**: a faction with zero bases
  picks a level with empty bases and spawns an invasion from a territory smart at a random
  retry interval.
- **`smart_terrain_warfare.smart_terrain_on_update`** (per smart, 1–5 min): `check_owner` →
  `process_manual_capture` → `process_squads`, then for owned smarts `update_resources` /
  `spawn_defense` / `process_targets` (invasions) / `spawn_patrols` / `process_patrols`; for
  unowned smarts, mutant spawning instead.

Smart roles come from smart props: `base` (spawns defense, launches invasions), `resource`
(feeds the faction economy), `territory` (patrol destinations, invasion staging), `lair`
(mutants), plus per-faction priority flags. Squad composition per spawn comes from
`faction_expansions.get_spawn_section` keyed on faction + resource tier *(agent-traced)*.

## Ownership & capture (verified)

`check_owner` tallies defending squads per faction at the smart (`current_target_id == smart.id`
and arrived; unregistered and companion squads excluded). The subtle rule: **an owned smart does
not flip on a head-count comparison** — the iteration keeps the current owner as the candidate
whenever it has any defender present; count-beats-count applies only to unowned smarts. In
practice a base changes hands when the owner's defenders are eliminated or gone. Refinements:
a "monster" winner is coerced to `none`; a player **manual capture point** forces actor-faction
ownership or nothing; when a faction friendly to the actor takes a smart, any player influence
staked there is refunded; saved owners are sticky for 300 s after load; and a faction whose
`keep_last_base` option is set cannot lose its final base. Owner switches reset patrol targets
and clear invasions, and re-show the map spot. The smart caches `defense_count`/`defense_power`
for target scoring (with a TODO-acknowledged spam message when a smart exceeds 25 squads).

## The player as commander

- **Two currencies.** Money buys units through the PDA (`novice/advanced/veteran_squad_price`,
  `heli_price`, verified debits); **influence points** are earned per enemy kill of a
  warfare-registered squad (`squad_on_npc_death`, verified: money reward + rank-weighted bonus +
  influence) and staked on smarts via the PDA map (`pda_smart_terrain_warfare`,
  `warfare.influence_levels[smart.id]`) to bias capture.
- **`warfare_faction_control` is the player's command deck, not a faction AI** (the name
  misleads): capture point, buy squads/heli, declare war, follow/attack/move orders, send/take
  squads, heli routing — plus **possess leader** (verified body): the actor takes over a squad
  leader, copying name/rank/community/reputation, inheriting the corpse-faction's goodwill and a
  random pocket-money roll, and migrating items. A large commented-out block of further PDA
  orders ("request support", "target selected smart") is admitted-unfinished.
- Companion squads are excluded from the sim everywhere (with an in-code credit to "xQd's fix"
  for dismissal).

## Data modules & persistence

`warfare_factions` (participant list — 13 factions, `monster` tracked but non-participatory;
per-faction aggregate table), `warfare_levels` (per-level smart/faction stats + cross-level
target selection over `level_targets.level_links`), `warfare_names` (display names + smarts
hidden from the map), `warfare_options` (the whole knob surface read from
`ui_options.get("alife/warfare/...")` — per-faction participation/respawn/invasion/priority
weights, global fog-of-war / all-out-war / auto-capture / keep-last-base / prices / azazel
compatibility). Persistence rides the standard `save_state`/`load_state` m_data callbacks under
`m_data.warfare`: influence, manual point, invasions, patrol tables, faction information,
`registered_squads` (faction → squad id → target smart) and `smart_owners` — squad runtime
fields are *not* saved; they're reconstructed from `registered_squads` on load.

## Verified bugs & quirks

- **Linked-level density bug** (`calculate_target_density`): the linked-levels loop iterates
  `lvl` but the body indexes `level_information[level]` — the home level's smarts are re-added
  once per linked level and linked-level smarts are never collected. Target density scoring is
  skewed toward (duplicated) home smarts.
- **`patrol_squads` nil store** (`update_faction`): the aggregate write reads an undefined
  global `patrol_squads` (the sibling locals `respawn`/`patrol_respawn`/`invasion_size`/
  `patrol_pop_factor` all exist) — nil overwrites the field initialized to 0 earlier in the
  same function.
- `override_functions` dead (see integration) — and with it the heli update swap.
- One minute of no-op warfare callbacks in every non-warfare game before `resetter` fires.
- `max_population` forced to 1 when a smart reports 0 — in-code comment admits "hackish and
  will result in the squatting squads, but meh".
- Sizeable admitted-unfinished surface: hostility-weighting TODO in `warfare_levels`, hardcoded
  1-in-10 off-level patrol chance, the commented PDA order set.

## Deeper

- Stock squad sim being locked out → `flows/scripted-squad.md` (+ the
  `modlayer/gameplay/squads-simulation.md` map); the smart-terrain machinery underneath →
  `flows/smart-terrain-core.md`; offline combat that resolves the actual fights →
  `flows/offline-combat.md`.
- Helicopter purchase/update split → `flows/helicopter-ai.md` (F95).
- Relations machinery behind all-out-war and enemy checks → `flows/relations-info.md` (F40).
- Domain routing → `modlayer/gameplay/warfare.md` (M18).
