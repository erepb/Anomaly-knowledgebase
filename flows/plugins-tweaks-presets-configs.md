---
domain: flows/plugins-tweaks-presets-configs
tags: [modlayer, gameplay, unpacked, config, ltx, data, plugins, tweaks, presets, defines, flow]
sources: [unpacked/configs/defines.ltx, unpacked/configs/axr_options.ltx, unpacked/configs/presets/includes.ltx, unpacked/configs/presets/gameplay_diff.ltx, unpacked/configs/ai_tweaks/xr_combat_ignore.ltx, unpacked/configs/ai_tweaks/simulation_objects.ltx, unpacked/configs/plugins/guaranteed_drops.ltx, unpacked/configs/system.ltx]
related: [flows/options-runtime.md, flows/creatures-actor-configs.md, flows/items-weapons-outfits-configs.md, flows/per-smart-logic-configs.md, flows/game-modes.md, flows/sim-board.md, flows/scheme-combat.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: plugins / tweaks / presets configs (F122)

The **tuning-knob & shared-foundation tier**: `configs/defines.ltx` (the inheritance root every item/
weapon/creature descends from), `configs/presets/` (video/difficulty/warfare preset bundles),
`configs/plugins/` (~26 data files for Lua systems), `configs/ai_tweaks/` (~18 AI-scheme tuning tables),
and the empty `axr_options.ltx` stub. Where the other Phase G9 flows document *content* (creatures, items,
zones, per-smart logic), this one documents the **constants**: the base sections everything inherits, the
preset tables, and the externalized tuning of systems documented elsewhere. The flow **owns `defines.ltx` +
`presets/` + the axr_options stub, and routes every `plugins/`/`ai_tweaks/` file to its owning flow**.

**Provenance:** **verified, no agent** — `defines.ltx` (full, 303 lines), `presets/includes.ltx`,
`presets/gameplay_diff.ltx` (head), `ai_tweaks/xr_combat_ignore.ltx` + `simulation_objects.ltx` (heads),
`plugins/guaranteed_drops.ltx` (head), `axr_options.ltx` (confirmed empty), and the `system.ltx` include
read; full `plugins/`/`ai_tweaks/`/`presets/` listings enumerated. Each plugin/tweak file's *runtime
behavior* is owned by the system flow it configures (routed below), not re-derived. **No `gamedata/`
override** of `plugins/`, `ai_tweaks/`, `presets/`, or `defines.ltx` at writing.

## `defines.ltx` — the inheritance root (verified, owned here)

`#include`d **first** in `system.ltx`, so its sections are the bottom of every `[child]:parent` chain in
the item/weapon/creature tiers. It is the home of the base parents the other config flows route here:

- **HUD bases:** `hud_base` (the weapon-HUD offset/strafe/shooting defaults every `wpn_*_hud` inherits,
  F120), `hud_mid_interia`/`hud_low_interia`, `hud_movement_layers`.
- **Immunity presets:** `sect_identity_immunities`, `sect_helm_immunities`, `sect_zero_immunities` (the
  named coefficient blocks items reference via `immunities_sect`).
- **Item base chain:** `global_item_actions` → `identity_immunities` (slot/condition_bar/can_trade/
  snd_on_take defaults) → `booster` (the consumable base: `eat_*`/`boost_*` fields, `class II_FOOD`, F103/
  F102) + `booster_script`/`booster_multi` (script-bound variants with `use1_functor`) + `zero_immunities`.
- **Weapon base parents (F120 routes here):** `weapon_probability` (`hit_probability_gd_*`) +
  `default_weapon_params:identity_immunities` (hit_power/hit_impulse/silencer_*/`use1_functor =
  item_weapon.menu_scope_inv` — the parent of every `wpn_*`).
- **Creature base (F118 routes here):** `common_ph_friction_params_on_npc_death` (ragdoll friction) →
  **`monster_base`** (feel distances, `actor_hit_effect`, `custom_data = default_custom_data.ltx`) — the
  root of every `m_<species>`.

So a claim like "`monster_base`/`default_weapon_params`/`hud_base` lives a directory up in `defines.ltx`"
(from F118/F120) is confirmed here — this file is that directory-up foundation.

## `presets/` — preset bundles (verified, owned here)

`presets/includes.ltx` assembles: **video presets** `video_minimum`/`low`/`default`/`high`/`extreme`
(graphics option bundles → renderer + F17 options), **difficulty presets** `economy_diff` + `gameplay_diff`
(→ F106 game-modes), and **warfare presets** `warfare_default` + `warfare_yoko` (→ F105). A preset is a
**namespaced option-key table** — e.g. `gameplay_diff.ltx [gameplay_easy]` maps
`gameplay/gameplay_diff/<key> = value` (actor_immunities/hit_power/dispersion/weight/thirst/sleep/…). The
options system (F17) + `game_difficulties` (F106) read these to push console/engine settings. (`includes.ltx`
is pulled by the options/preset loader, not directly by `system.ltx`.)

## `axr_options.ltx` — empty stub (verified)

`axr_options.ltx` is **0 lines** — a placeholder. The live options model is the per-feature
`options_modded_exes_*` UI pages (F17, `flows/options-runtime.md`) + `axr_main.config` persistence; this
stub is not the options registry.

## `plugins/` — data for Lua systems (verified listing; route to owners)

26 files, each the externalized config of a Lua system documented elsewhere — **route, don't duplicate**:

- `achievements.ltx` → F107 · `actor_effects.ltx` → F103 · `bounty_squad.ltx` → F114/F98 ·
  `difficulty.ltx` → F106 · `disguise.ltx` → F102 (`gameplay_disguise`) · `dynamic_anomalies.ltx` → F90 ·
  `dynamic_faction_relations.ltx` → F108 · `encyclopedia.ltx`/`encyclopedia_interactivity.ltx` → F100 ·
  `faction_profile.ltx` → F108/F112 · `faction_quick_travel.ltx` → F107 · `guaranteed_drops.ltx` → F102/
  loot (the "after N corpses with no [item], force a drop" counter table) · `inventory_tabs.ltx` → F104 ·
  `mlr.ltx` → MLR addon · `new_game_money`/`new_game_options`/`new_game_setup`/`new_game_start_locations`
  → F106 · `news_tips_icons.ltx` → F112 · `nimble_orders.ltx` → quest/trade · `player_icons.ltx` → UI ·
  `radio_zone_fm.ltx` → F110 · `radioactive_water_settings.ltx` → F103 · `spawner_blacklist.ltx` → F117/
  F123 · `xr_sound.ltx` → F116 · `xrs_debug_tools.ltx` → F111. (`debug/` subdir = debug data, F111.)

## `ai_tweaks/` — AI-scheme tuning tables (verified listing; route to owners)

18 files, the tunable constants of the AI/scheme layer — **route, don't duplicate**:

- `xr_combat_ignore.ltx` → F76 (`combat_inertion` per-faction/difficulty memory + `forbidden_npcs`) ·
  `simulation_objects.ltx` → F85/F86 (the `target_maps` **linked-level** table deciding which levels stay
  simulated — a real performance lever) · `axr_companions.ltx` → F114 · `axr_smartcover.ltx` → F33 ·
  `sr_light.ltx` → F91 · `xr_corpse_detection`/`xr_gather_items.ltx` → F80 · `xr_help_wounded`/
  `xr_eat_medkit.ltx` → F78 · `xr_danger`/`xr_visual`/`xr_weapon_jam.ltx`/`xrs_facer`/`xrs_kill_wounded` →
  F81/scheme tweaks · `rx_ff`/`rx_gl.ltx` → combat (friendly-fire / grenade) · `ai_tweaks.ltx` → general AI.

## The unifying insight

This tier holds the **constants**, not the instances: the base sections everything inherits (`defines.ltx`),
the preset bundles (`presets/`), and the externalized tuning tables of systems documented across the KB
(`plugins/`, `ai_tweaks/`). "Tweaking the game's numbers" without authoring new content almost always means
editing here — but the *behavior* those numbers drive is owned by the system flow, which is why this flow is
mostly a router with two genuinely-owned pieces (`defines.ltx`, `presets/`).

## Isolation ledger (the seams)

- **Inheritance foundation:** `defines.ltx` base sections are the parents of the F118 (creatures), F120
  (items/weapons/outfits), and F103/F102 (booster/consumable) chains.
- **Options/difficulty (F17/F106):** `presets/` bundles + the `gameplay/gameplay_diff/*` namespaced keys.
- **Warfare (F105):** `presets/warfare_*`.
- **Simulation (F85/F86):** `ai_tweaks/simulation_objects.ltx` linked-level table.
- **Per-system plugin/tweak data:** routed above to F76/F78/F80/F81/F90/F100/F102/F103/F104/F106/F107/
  F108/F110/F111/F112/F114/F117.
- **LTX/DLTX mechanism:** `#include` + `[child]:parent` + DLTX deltas → `modlayer/config-ltx.md`.

## Failure & edge cases

- **`defines.ltx` is load-bearing:** it's `#include`d first; a broken section there breaks every item that
  inherits it. Edit base parents with care.
- **`axr_options.ltx` is empty** — don't look for the options registry here (it's F17 + the
  `options_modded_exes_*` pages).
- **Presets are option-key tables, not sections of game objects** — `gameplay/gameplay_diff/<key>` paths are
  consumed by the options/difficulty system, not read as an item section.
- **plugins/ and ai_tweaks/ are config, not code** — the file tells you the *numbers*; the *logic* is in the
  owning Lua flow. Read both together when tuning.
- **DLTX, not full overrides:** no `gamedata/` copies at writing. A full override here would be unusual —
  flag it.

## Deeper

- Foundation: `unpacked/configs/defines.ltx` (`hud_base`, `identity_immunities`, `booster`,
  `default_weapon_params`, `monster_base`)
- Presets: `unpacked/configs/presets/includes.ltx` + `gameplay_diff.ltx`/`economy_diff.ltx`/`video_*.ltx`/
  `warfare_*.ltx`
- Tuning exemplars: `ai_tweaks/xr_combat_ignore.ltx`, `ai_tweaks/simulation_objects.ltx`,
  `plugins/guaranteed_drops.ltx`
- Options model: `flows/options-runtime.md` (F17); difficulty: `flows/game-modes.md` (F106)
- Map: `modlayer/gameplay/configs-data.md` (M23); LTX mechanism: `modlayer/config-ltx.md`

## Maintenance

- `defines.ltx` read in full; `presets/` + plugin/tweak files enumerated with one exemplar each. When a
  task needs a specific plugin/tweak's *meaning*, go to its owning flow (routed above); the file format is
  plain LTX.
- Re-check `gamedata/configs/{plugins,ai_tweaks,presets}/` + `defines.ltx` for overrides per session (none
  at writing).
- Keep the load-bearing notes: **`defines.ltx` = the inheritance root (the directory-up home of
  monster_base/default_weapon_params/hud_base)**, **`axr_options.ltx` is an empty stub**, and **plugins/
  ai_tweaks are tuning data routed to their owning system flows**.
