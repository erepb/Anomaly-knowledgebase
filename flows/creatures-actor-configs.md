---
domain: flows/creatures-actor-configs
tags: [modlayer, gameplay, config, ltx, creatures, monster, actor, data, flow]
sources: [unpacked/configs/creatures/monsters.ltx, unpacked/configs/creatures/stalkers.ltx, unpacked/configs/creatures/m_dog.ltx, unpacked/configs/creatures/actor.ltx, unpacked/configs/creatures/immunities.ltx, unpacked/configs/creatures/damages.ltx, unpacked/configs/creatures/game_relations.ltx, unpacked/configs/creatures/spawn_sections.ltx, unpacked/configs/defines.ltx]
related: [INDEX.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, flows/monster-schemes.md, flows/monster-ai-control.md, flows/damage-hit.md, flows/relations-info.md, flows/faction-relations-lua.md, flows/actor.md, flows/helicopter-ai.md, flows/game-services.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: creatures & actor configs (data tier)

The `configs/creatures/` data tier: the LTX that defines every monster species, the actor, and the
shared damage / immunity / relations profiles the engine reads at spawn. This is config, not code — a
**catalog flow** mapping the include structure, the section-inheritance chain, and which parameter
groups live where. The first flow of Phase G9 (the config/data tier), under map M23.

**Provenance:** **verified, no agent (config read).** `monsters.ltx` / `stalkers.ltx` /
`spawn_sections.ltx` (the master includes) read in full; `m_dog.ltx` read as the representative species
(structure + the inline immunity/damage/protection/vision sub-section references); `actor.ltx`,
`immunities.ltx`, `damages.ltx`, `game_relations.ltx` heads read; `monster_base` located in
`defines.ltx`. The ~30 `m_*.ltx` and ~27 `spawn_sections_*.ltx` files are leaves on the same pattern
(not each re-read). **No `gamedata/` override** of `configs/creatures/` (verified by `find`).

---

## Include & inheritance structure

Three master files pull everything in:

- **`monsters.ltx`** — `#include`s every `m_<species>.ltx` (flesh, bloodsucker, dog, boar, pseudodog,
  giant, controller, poltergeist, snork, crow, tushkano, phantom, chimera, burer, zombie, fracture,
  cat, rat, borya, bibliotekar, + Tronex additions karlik/lurker/psysucker/gigant_jumper) and defines
  the shared `[monsters_common]` section (corpse/death timers, etc.).
- **`stalkers.ltx`** — `#include`s `m_stalker.ltx`, `m_person.ltx`, `immunities.ltx`, `damages.ltx`,
  `m_stalker_zombied.ltx`, `m_stalker_monolith.ltx`.
- **`spawn_sections.ltx`** — `#include`s the ~27 per-level `spawn_sections_<level>.ltx` (+ a `_general`).

**Section inheritance** uses LTX `[child]:parent` syntax. The chain is:
`monster_base` (defined in **`configs/defines.ltx`**) → `[m_<species>_<variant>]:monster_base` (in the
`m_*.ltx` files) → the per-level spawn-section variants (in `spawn_sections_*.ltx`) → the actual object
that ALife spawns. So a monster's stats are assembled by walking that parent chain; a spawn section is
usually a thin override of a species variant tuned for a level.

## A monster section (`m_dog.ltx` as representative)

Each species file defines one or more variants (`[m_dog_e]:monster_base`, etc.). The fields group into:

- **ALife / engine identity:** `GroupControlSection`, `$npc`/`$prefetch`/`Scheduled`/`Human`, `visual` /
  `corpse_visual` / `icon`, `cform` (collision class) + `class` (the AI class id, e.g. `SM_DOG_S`),
  `script_binding = bind_monster.bind`.
- **Stats:** `Health` / `MaxHealthValue`, `MinSpeed`/`MaxSpeed`/`going_speed`/`search_speed`,
  `DynamicObjectsCount`, `smart_terrain_choose_interval`, `panic_threshold`.
- **Evaluation-function types:** `ef_creature_type` / `ef_weapon_type` / `ef_detector_type` (used by the
  AI selectors).
- **Loot:** `Spawn_Inventory_Item_Section` + `_Probability` (e.g. `mutant_dog_tail`).
- **Bones:** `bone_torso` / `bone_head` / `bone_fire`.
- **Influences (environmental absorption):** `fire_/psy_/radiation_*` max-distance/power/linear/quadratic
  factors — how the creature emits or resists those fields (e.g. a dog's radiation pp-effector).
- **Physics:** `ph_box*_center`/`_size` collision capsules; `ph_collision_damage_factor`.
- **Inline profile references (key):** a species names its own sub-sections and defines them in the same
  file — `damage = m_dog_damage`, `immunities_sect = dog_immunities`, `protections_sect = dog_protections`,
  `vision_free_section` / `vision_danger_section`, `hit_fraction_monster`. So **monster immunities are
  per-species inline**, not from `immunities.ltx`.
- **Sound/anim** sections follow further down.

## Shared profile files

- **`immunities.ltx`** — named **resistance profiles** (`no_resistance`, `low_/medium_/high_resistance`,
  …) each giving per-damage-type immunity coefficients: `burn_/strike_/shock_/wound_/radiation_/`
  `telepatic_/chemical_burn_/explosion_/fire_wound_immunity`. These are the **stalker** resistance
  presets (referenced by NPC sections); monsters mostly define immunities inline (above). A coefficient
  multiplies incoming damage of that type (lower = more resistant). See `flows/damage-hit.md` (F30).
- **`damages.ltx`** (~1150 lines) — named **bone-damage profiles** (`body_damage_*`): per-bone
  `(bone_armor, damage_multiplier)` maps that scale a hit by where it lands (headshot multipliers, etc.).
  Referenced by a section's `damage = <profile>`. Damage math: see the pastebin link in the file header.
- **`game_relations.ltx`** — the rank/reputation/rating **threshold tables** (`rating`, `reputation`,
  `monster_rating`, `goodwill_names`, `attitude_*_threshold`) and the faction-relations matrix.
  **Owned by F40/F108** (`flows/relations-info.md`, `flows/faction-relations-lua.md`) — routed here, not
  duplicated; its rank/reputation tables are also read by game services (F107, `flows/game-services.md`).

## actor.ltx

The `[actor]:…` section is the player's creature definition: `class = S_ACTOR`,
`script_binding = bind_stalker.actor_init`, `visual`, `cform`. Then the **player tuning** groups:
movement coefficients (`run_coef`/`sprint_koef`/`jump_speed`/`crouch_coef`/`climb_coef`/`walk_accel`/…),
`max_item_mass`, grenade-feel radius/time, `ef_creature_type`, `attachable_items` (torches, hand radio),
night-vision sounds, physics boxes, and (further down) the survival/condition parameters that pair with
the actor survival system (`flows/actor.md`, F103 actor-survival). `actor_hud_hand_animations.ltx` is the
HUD hand-anim sibling.

## spawn_sections & helicopter

- **`spawn_sections_<level>.ltx`** — the per-level catalog of ready-to-spawn creature sections (thin
  overrides of species variants), the bridge into the spawn/ALife tier (F123). `spawn_sections.ltx`
  includes them all; `_general` holds cross-level ones.
- **`helicopter.ltx`** — the Mi-24/heli object params (health, weapons, movement); pairs with the
  helicopter AI (`flows/helicopter-ai.md`, F95).

## Notes & gotchas

- **Two immunity mechanisms** — stalkers use the named profiles in `immunities.ltx`; monsters define
  immunity sub-sections **inline** per species. Don't expect a mutant's resistances in `immunities.ltx`.
- **`monster_base` is in `defines.ltx`**, not in `creatures/` — the inheritance root lives a directory up.
- **Spawn sections vs species** — to retune a monster everywhere, edit its `m_*.ltx` variant; to retune
  it on one level, edit that level's `spawn_sections_*.ltx` override.
- **DLTX applies** — mods patch these with `.ltx` deltas rather than full overrides
  (`flows/dxml-xml-interception.md`, `modlayer/config-ltx.md`); hence no `gamedata/` full-file copies here.

## Deeper

- Monster AI/behaviour that reads these stats → `flows/monster-schemes.md`, `flows/monster-ai-control.md`
  (`class`/`ef_*_type`); damage application → `flows/damage-hit.md` (F30, immunities + bone profiles).
- Relations/rank tables → `flows/relations-info.md` (F40), `flows/faction-relations-lua.md` (F108),
  `flows/game-services.md` (F107 rank/rep); heli → `flows/helicopter-ai.md` (F95).
- Actor survival pairing → `flows/actor.md` (F103); LTX/DLTX mechanism → `modlayer/config-ltx.md`.
- Config-tier map → `modlayer/gameplay/configs-data.md` (M23).
