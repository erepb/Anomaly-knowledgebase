---
domain: flows/per-smart-logic-configs
tags: [modlayer, gameplay, unpacked, config, ltx, data, logic, custom-data, gulag, smart-terrain, flow]
sources: [unpacked/configs/scripts/door.ltx, unpacked/configs/scripts/sr_sleep.ltx, unpacked/configs/scripts/sr_teleport_logic.ltx, unpacked/configs/scripts/camp.ltx, unpacked/configs/scripts/escape/esc_smart_terrain_2_12_smart_logic.ltx, unpacked/configs/scripts/escape/smart/esc_smart_terrain_1_11.ltx, gamedata/configs/scripts/labx8/mod_lx8_door_lock_fix.ltx]
related: [flows/logic-ltx-switching.md, flows/scheme-engine-core.md, flows/condlist-library.md, flows/smart-terrain-core.md, flows/gulag-jobs.md, flows/sim-board.md, flows/scheme-walker-patrol.md, flows/scheme-camper-cover.md, flows/scheme-animpoint.md, flows/monster-schemes.md, flows/physics-object-schemes.md, flows/creatures-actor-configs.md, flows/items-weapons-outfits-configs.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: per-smart logic configs (F121)

The **`configs/scripts/` data tier** — ~2016 `.ltx` files that are the **custom_data / logic-LTX** content
the scheme engine reads: per-object logic FSMs, per-smart gulag job catalogs, smart_terrain respawn
definitions, and the reusable object templates (doors, campfires, boxes, sleep/teleport restrictors). This
is the data side of the framework spine: F68 (logic-LTX format) parses it, F66 (scheme engine) walks it,
F72/F73 (smart_terrain + gulag) consume the job catalogs, F86 (sim_board) reads the respawn configs. The
flow documents **what this tier is, the two file roles, how it's loaded, the gulag-ranking convention, and
who owns each piece** — not a per-file dump (the per-level files are instances of the same handful of
patterns).

**Provenance:** **verified, no agent** — `door.ltx`, `sr_sleep.ltx`, `sr_teleport_logic.ltx`, `camp.ltx`,
`escape/esc_smart_terrain_2_12_smart_logic.ltx` (a full gulag job catalog), `escape/smart/
esc_smart_terrain_1_11.ltx` (a smart respawn def), and the one `gamedata/` DLTX delta read in full;
directory tree + per-level counts listed. The per-level files (jupiter 206, zaton 177, pripyat 130,
sarcofag 107, warlab 78, bar 70, …) are instances of these patterns, not each re-read; the *runtime
meaning* of each scheme/condlist is routed to F66/F68/F67 + the scheme-catalog flows. **One `gamedata/`
override** at writing: `scripts/labx8/mod_lx8_door_lock_fix.ltx` (a DLTX `@`-delta, below).

## Not an `#include` tier — loaded by name (verified)

Unlike `creatures/`, `zones/`, `items/` (which `#include` into `system_ini`), **`configs/scripts/` files
are loaded on demand by name** as an object's or smart's **`custom_data`** (the logic file pointer set in
spawn data / by a binder / by a scheme `cfg=`). So there's no master manifest; the file name *is* the
reference. Organization: **top-level reusable templates** (shared across levels) + **one subdir per level**
(`escape/`, `jupiter/`, `pripyat/`, `zaton/`, …) holding that level's object/smart logic.

## Two file roles (verified)

### 1. Object logic — single `[logic]` + `active =` (F68 format)

A single-object FSM: `[logic]` with `active = <scheme>@<section>`, then the scheme sections with condlist
transitions (`on_info`/`on_actor_inside`/`on_actor_outside`/`on_info2` → `{condlist} target %effects%`).
Verified exemplars:

- `door.ltx` — `[logic] active = ph_door@unlocked` + `[ph_door@unlocked] locked=false closed=true` (the
  `ph_door` scheme, F93).
- `sr_sleep.ltx` — an `sr_idle` tutorial FSM: `sr_idle@wait` → `sr_idle@run %=run_tutorial(...)%` on
  `actor_inside`, back on `actor_outside`/`on_info` (condlist + effect tokens = F67/F68).
- `sr_teleport_logic.ltx` — `sr_idle@wait` → `@run %=sr_teleport%` on `actor_inside` (the per-script
  teleport custom_data referenced by `sr_teleport`, the F92 sibling of the `sr_teleport_sections.ltx`
  level-changer in F119).

These attach to one object via its `custom_data`. The scheme sections (`walker@`, `ph_door@`, `sr_idle@`,
`animpoint@`, `mob_home@`, `meet`) are exactly the schemes documented across Phases G2/G4/G5.

### 2. Gulag / smart logic — many `[logic@<job>]` (F73 job board)

A smart's `<smart>_smart_logic.ltx` is a **job catalog**: many `[logic@<jobname>]` sections, each with
`active = <scheme>@<section>`, a `prior =` rank, an optional `suitable = {condlist} true` gate, and
optional `monster_job = true`. The gulag (F73) ranks these by `prior` + `suitable` and assigns NPCs/
monsters to them; the referenced scheme/path sections live in the same file. Verified on
`esc_smart_terrain_2_12_smart_logic.ltx`:

- **Static jobs:** `logic@walker_1..5` (prior 40, `walker@` patrol) + `logic@guard_1..4` (prior 45,
  `camper@`).
- **Surge-reactive jobs:** `logic@..surge_work_1..11` (`suitable = {=surge_started} true`, prior 60) drive
  `walker@..surge_work_N` (path + `out_restr` hide zone). Built with **in-file section inheritance**:
  `[logic@..surge_work_1]:logic@..surge_def` and `[walker@..work_1]:walker@..surge_def` factor out shared
  fields.
- **Camp jobs:** `logic@..camp_work_1..18` (`suitable = {!surge_started} true` / `{=is_night ...}`, prior
  41) drive `animpoint@` sections (`cover_name`, `use_camp=true`, `turn_on_campfire=true`) with day/night/
  **rain** variants chained via `on_info = {=is_heavy_rain} animpoint@..._10` (F77 animpoint + the
  `[meet_random]` inline meet override, F79).
- **Mutant-swarm jobs:** `logic@faction_base_defense_enemy1..3` (`monster_job = true`,
  `suitable = {=target_squad_name(...)} true`, prior 200) drive `mob_home@` leashes (F94).

So one smart-logic file encodes the entire population behavior of a smart, condlist-gated by world state
(surge / time-of-day / weather / which squad is present).

## Smart respawn defs — the `smart/` subdir (verified)

Each level's `scripts/<level>/smart/<smart>.ltx` is the smart_terrain **definition** (distinct from the
`*_smart_logic.ltx` job catalog): `[smart_terrain]` (`squad_id`, `max_population`, `respawn_params`,
`respawn_idle`) + `[respawn@...]` + a respawn template (`[spawn_all_weak] spawn_squads = …, spawn_num = …`).
This is the **population/respawn** config read by smart_terrain core (F72) + sim_board (F86) — what spawns
where, how often, capped by `max_population`.

## Reusable top-level templates (verified)

Shared `custom_data` referenced by many objects across all levels: the door family (`door.ltx`,
`door_metal_*`, `door_wood_*`, `safe_door_metal.ltx`, …), `ph_campfire.ltx`/`ph_sleepbag.ltx`/`ph_tent.ltx`,
the box family (`inventory_box.ltx`, `treasure_inventory_box.ltx`, `actor_inventory_box.ltx`,
`slide_box_small.ltx`), `sr_sleep.ltx`, `sr_teleport_logic.ltx`/`sr_teleport_no_cofirm_logic.ltx`,
`bribes_logic.ltx`, `beh_companion.ltx`, `scenes.ltx`, `dummy.ltx`. **Special:** `camp.ltx` is **not** an
FSM — its `[camp]` section is the campfire-storyteller registry (`stories`/`guitar_themes`/
`harmonica_themes`) read by the `sound_manager` `sm` scheme (F116). `tasks/` holds quest task logic (→
F97/F98).

## DLTX deltas apply here too (verified)

The one `gamedata/` override at writing is `scripts/labx8/mod_lx8_door_lock_fix.ltx`, a **DLTX `@`-delta**:
`@[ph_button@unlocked] anim = idle / tooltip = st_lx8_button_press_true`. So this tier is patched the same
way as the rest — surgical section/key deltas, not full-file copies. Re-check `gamedata/configs/scripts/`
per session.

## Isolation ledger (the seams)

- **Logic-LTX format (F68):** `[logic]`/`active=`/`[scheme@section]` + `on_*` condlist transitions — the
  parser that turns these files into the switch list.
- **Scheme engine (F66):** walks the active section + switch conditions; the `cfg=`/`custom_data` pointer
  is how a file gets loaded.
- **Condlist library (F67):** the `{+info -info =func(args) !func}` predicates + `%=effect%` in `suitable`
  / `on_*` lines.
- **Gulag job board (F73):** `[logic@job]` + `prior`/`suitable`/`monster_job` = the rankable job catalog.
- **Smart_terrain core (F72) + sim_board (F86):** the `smart/<smart>.ltx` `[smart_terrain]` + `respawn@` +
  `spawn_squads` population config.
- **Scheme catalog:** `walker@` (F74), `camper@` (F75), `animpoint@` + `meet` (F77/F79), `mob_home@` (F94),
  `ph_door@`/`ph_button@` (F93), `sr_idle@`/`sr_teleport` (F91/F92).
- **Sound (F116):** `camp.ltx [camp]` storyteller registry.
- **Tasks (F97/F98):** `scripts/tasks/`.
- **LTX/DLTX mechanism:** parse + `[child]:parent` (in-file inheritance) + `@`-deltas →
  `modlayer/config-ltx.md`.

## Failure & edge cases

- **No `#include` — reference by name:** a logic file does nothing until an object/smart's `custom_data`
  points at it; a typo in the pointer = no logic (object idles on engine defaults).
- **`smart/<smart>.ltx` ≠ `<smart>_smart_logic.ltx`:** the former is the respawn/population definition, the
  latter the gulag job catalog. Don't conflate.
- **`suitable` + `prior` decide assignment, not `active`:** within a smart-logic file, the gulag picks
  which `[logic@job]` an NPC gets; `active=` only names that job's starting scheme section.
- **In-file inheritance is local:** `[walker@x]:walker@y` resolves within the same file (factoring shared
  job fields), distinct from cross-file `#include` inheritance in the system-ini tiers.
- **`camp.ltx` is a registry, not an FSM** — editing it changes campfire stories (F116), not behavior
  switching.
- **DLTX deltas exist here** (the labx8 door-lock fix) — full `gamedata/` copies are rare; expect surgical
  `@`-patches.

## Deeper

- Object logic exemplars: `scripts/door.ltx`, `scripts/sr_sleep.ltx`, `scripts/sr_teleport_logic.ltx`
- Gulag job catalog: `scripts/escape/esc_smart_terrain_2_12_smart_logic.ltx`
- Smart respawn def: `scripts/escape/smart/esc_smart_terrain_1_11.ltx`
- Storyteller registry: `scripts/camp.ltx`
- DLTX delta: `gamedata/configs/scripts/labx8/mod_lx8_door_lock_fix.ltx`
- Format/engine: `flows/logic-ltx-switching.md` (F68), `flows/scheme-engine-core.md` (F66),
  `flows/condlist-library.md` (F67); job board: `flows/gulag-jobs.md` (F73), `flows/smart-terrain-core.md`
  (F72), `flows/sim-board.md` (F86)
- Map: `modlayer/gameplay/configs-data.md` (M23); LTX mechanism: `modlayer/config-ltx.md`

## Maintenance

- Exemplars of both file roles + a smart def read in full; the ~2000 per-level files are instances of the
  same patterns. When a task needs a *specific* smart/object, read that file directly — the format is
  documented here, the content is per-instance.
- Re-check `gamedata/configs/scripts/` for DLTX deltas per session (one at writing: the labx8 door-lock
  fix).
- Keep the three load-bearing notes: **loaded by name as custom_data (not `#include`)**, **`smart/` def vs
  `*_smart_logic` job catalog**, and **`suitable`/`prior` gulag ranking**.
