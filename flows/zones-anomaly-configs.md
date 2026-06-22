---
domain: flows/zones-anomaly-configs
tags: [modlayer, gameplay, unpacked, config, ltx, data, zones, anomaly, teleport, flow]
sources: [unpacked/configs/zones/zones.ltx, unpacked/configs/zones/zone_base.ltx, unpacked/configs/zones/zone_mine_gravitational.ltx, unpacked/configs/zones/zone_field_radioactive.ltx, unpacked/configs/zones/zone_field_psychic.ltx, unpacked/configs/zones/zone_teleport.ltx, unpacked/configs/zones/zone_monolith.ltx, unpacked/configs/zones/zone_campfire.ltx, unpacked/configs/sr_teleport_sections.ltx, unpacked/configs/system.ltx]
related: [flows/anomaly-zones.md, flows/anomaly-field-binders.md, flows/restrictors-active-sr.md, flows/damage-hit.md, flows/creatures-actor-configs.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: zones & anomaly params (F119)

The **`configs/zones/` data tier** — the LTX that defines every anomaly/zone section the engine spawns,
plus the unrelated-but-co-filed **`sr_teleport_sections.ltx`** level-changer table. This is content, not
code: the parameter surface that the C++ `CustomZone` family (engine side, `flows/anomaly-zones.md`, F27)
and the Lua `anomaly_zone_binder` (F90) read. This flow documents the **include graph, the section
inheritance, the section→engine-class mapping, and the parameter groups** — not an exhaustive field dump;
what each field *does* at runtime is owned by F27.

**Provenance:** **verified, no agent** — `zones.ltx`, `zone_base.ltx`, the gravitational mine family, both
radioactive/psychic fields, `zone_teleport.ltx`, `zone_monolith.ltx`, and `sr_teleport_sections.ltx` read
in full; campfire head + the `class=` catalog grepped across all 20 files; the `system.ltx` include
verified. The *runtime meaning* of each parameter is routed to `flows/anomaly-zones.md` (F27), not
re-derived here. **No `gamedata/` override** of `configs/zones/` or `sr_teleport_sections.ltx` at writing
(`gamedata/configs/zones/` empty; the teleport table exists only under `unpacked/`).

## Include graph (verified)

`configs/system.ltx` pulls in `zones\zones.ltx`, which is a pure **#include manifest**: `zone_base.ltx`
first (the inheritance root), then `zone_campfire.ltx`, the special zones (`burningfuzz`, `minefield`,
`nogravity`, `teleport`, `fireball`, `generator`), the four `zone_field_*` files, the four `zone_mine_*`
files, and the named oddballs (`monolith`, `liana`, `hvatalka`, `studen`, `mosquitobald`). So the whole
anomaly catalog assembles into `system_ini` at load, the same way creatures do (`flows/creatures-actor-
configs.md`, F118).

## The inheritance root: `zone_base` (verified)

`[zone_base]` carries only the **shared engine defaults**: volumetric/idle light knobs, `effective_radius`,
`pick_dof_effector`, the R1 idle-light flag — and crucially **`script_binding = bind_anomaly_field.bind`**.
That single line is why every anomaly is also a Lua-bound object: the engine attaches the
`anomaly_zone_binder` (F90, `flows/anomaly-field-binders.md`) to it, which runs the **artefact-spawn
economy** on top of the engine zone. `[zone_base_noshadow]:zone_base` is a shadow-off variant. Every other
zone section is `[…]:zone_base` (or `:campfire_base`, etc.), so it inherits the binding + light defaults
and overrides the rest.

## Section → engine class (verified mapping)

Each section names its C++ implementation via **`class = ZS_*` / `Z_*`** (a clsid resolved by the engine
class registrar to a `CustomZone` subclass — engine tier in `flows/anomaly-zones.md`). The catalog grep:

- **`ZS_MINCE`** — gravitational mincer (the "meat grinder"); **`ZS_GALAN`** — gravitational average.
- **`ZS_MBALD`** (the workhorse, ~13 sections) — Mosquito-Bald springboard physics; used by gravi-weak,
  teleport, minefield, mosquitobald.
- **`ZS_RADIO`** — radioactive/DoT field engine; used by `zone_field_radioactive`, `zone_field_psychic`,
  and `zone_monolith` (psy emitter).
- **`ZS_BFUZZ`** — "burning fuzz" grabber; `burningfuzz`/`liana`/`hvatalka`/`studen`.
- **`ZS_TORRD` / `Z_TORRID`** — thermal; `fireball`/`generator`.
- **`ZS_NGRAV`** — no-gravity; **`Z_CFIRE`** — campfire (harmless unless lit).

The same section also declares spawn metadata: `GroupControlSection = spawn_group_zone`, `$spawn` (the
editor spawn-section name), `$def_sphere`/`$prefetch` (editor sphere + preload), and `ef_anomaly_type` /
`ef_weapon_type` — the AI **evaluation-function types** (same `ef_*_type` family the creature configs use,
F118) by which AI classifies and avoids the zone.

## Variant tiers via inheritance (verified)

The strength tiers are **thin inheritance leaves**: `[zone_field_radioactive]:zone_base` holds the full
parameter block, then `[zone_field_radioactive_weak]:zone_field_radioactive` overrides **only**
`$spawn` + `max_start_power` (0.015 / 0.02 / 0.03 for weak/average/strong); MP variants re-tier again
(`[zone_radioactive]:zone_field_radioactive` → `_weak/_average/_strong`). Read a tier section bottom-up
through its parents to get the effective values — same `[child]:parent` rule as everywhere in the LTX tier
(`modlayer/config-ltx.md`).

## Parameter groups (read in full on the gravi/field exemplars)

A full anomaly section (gravitational mine is the richest) clusters into:

- **Anomaly strength:** `min_start_power`/`max_start_power`, `attenuation`, `period`.
- **Hit:** `hit_type` (`strike` / `radiation` / `telepatic` / `burn`), `hit_impulse_scale`,
  `effective_radius` — `hit_type` is the `SHit` type that feeds the damage chain (`flows/damage-hit.md`,
  F30); fields use `radiation`/`telepatic` (silent DoT), mines use `strike` (impact).
- **Blowout FSM timings:** `awaking_time` → `accamulate_time` → `blowout_time`, plus the staggered
  `blowout_light_time`/`blowout_explosion_time`/`blowout_particles_time`/`blowout_sound_time` and the
  `disable_time`/`disable_time_small`/`disable_idle_time` cooldowns (`-1` = never).
- **Particles & sound:** the parallel `idle_/blowout_/hit_/entrance_/tele_/tearing_/torn_` particle keys
  and `idle_/blowout_/hit_/entrance_/body_tearing_sound` keys.
- **Light:** `blowout_light`/`idle_light` toggles + `light_color`/`light_range`/`light_time`/`light_height`.
- **Impulse physics:** `throw_in_impulse`/`throw_out_impulse`/`throw_in_impulse_alive`/`throw_in_atten`,
  `blowout_radius_percent`/`actor_blowout_radius_percent`.
- **Teleport-corpse:** `tele_height`/`time_to_tele`/`tele_pause` (gravi anomalies fling and relocate
  bodies).
- **Wind:** `blowout_wind` + `blowout_wind_time_start/peak/end` + `blowout_wind_power`.
- **Misc:** `ignore_nonalive`/`ignore_small`/`ignore_artefacts`, `shape_transp_color`/`shape_edge_color`
  (the editor shape tint), and **`postprocess = postprocess_*`** — a screen post-process preset reference
  (preset tier is F122).

## Two families + the named oddballs (verified)

- **`zone_field_*`** — passive DoT fields: tiny/zero `max_start_power`, no blowout, `hit_type`
  radiation/telepatic, persistent `disable_*=-1`. The harmless-looking always-on hazards.
- **`zone_mine_*`** — classic blowout anomalies: the awaking→accumulate→blowout FSM, real impulse, corpse
  fling+teleport. The thing you bolt-test.
- **`zone_monolith`** — `ZS_RADIO` + `hit_type telepatic`: a max-power psy emitter (Brain-Scorcher style),
  artefact economy explicitly **off** (`spawn_blowout_artefacts = off`, `max_artefact_count = 0`),
  `visible_by_detector = on`.
- **`zone_campfire`** (`[campfire_base]:zone_base`, `Z_CFIRE`) — harmless unless lit; carries its own
  `script_binding` (campfire world-object binder, F89, `flows/world-object-binders.md`), distinct from the
  anomaly-field binder.

## The artefact-spawn seam (verified config side)

Anomaly sections **declare** the engine artefact fields (`artefacts`, `BirthProbability`,
`min/max_artefact_count`, `spawn_blowout_artefacts`, `visible_by_detector` — visible on `zone_monolith`),
but in Anomaly the live artefact economy runs in **Lua** via the `anomaly_zone_binder` attached through
`zone_base`'s `script_binding`. So: config = engine defaults + which class; **where artefacts actually come
from = F90** (`flows/anomaly-field-binders.md`). Don't read the config artefact fields as the live spawn
rules.

## `sr_teleport_sections.ltx` — the level-changer table (verified, NOT an anomaly)

Co-filed but unrelated to zones: this is the **inter-level transition** data tier consumed by the
`sr_teleport` **active space-restrictor scheme** (F92, `flows/restrictors-active-sr.md`). One section per
transition, named after the space restrictor (`[<lvl>_space_restrictor_to_<dest>]`), with:

- `enable` — condlist gate (can be a condition list, not just true).
- `spot` — `map_spots.xml` icon for the global-map marker.
- `text` — the Yes/No prompt string shown when the actor enters.
- `hint` — map-spot hint (`st_items_quest.xml`).
- `path` — waypoint the actor teleports to on **Yes**; `reject_path` — waypoint on **No**.
- `autosave` — autosave on transition.

So the flow is: actor walks into the named space restrictor → `sr_teleport` (F92) looks up this section →
Yes/No box → teleport to `path` (the level change) or bounce to `reject_path`. Distinct from the
**per-script** teleport logic files under `configs/scripts/…/*_teleport*.ltx` and `sr_teleport_logic.ltx`,
which are story-level custom_data and belong to the per-smart logic tier (F121).

## Isolation ledger (the seams)

- **Engine zone runtime (F27):** `class = ZS_*` clsid → `CustomZone` subclass; all the strength/hit/
  blowout/particle/light/impulse params are read there (`flows/anomaly-zones.md`).
- **Anomaly-field binder (F90):** `zone_base`'s `script_binding = bind_anomaly_field.bind` → the Lua
  artefact-spawn economy + layers (`flows/anomaly-field-binders.md`).
- **Campfire binder (F89):** `zone_campfire`'s own `script_binding` → world-object/campfire binder.
- **Damage chain (F30):** `hit_type` is the `SHit` type the zone delivers (`flows/damage-hit.md`).
- **AI (F118 family):** `ef_anomaly_type`/`ef_weapon_type` are the same `ef_*_type` evaluation surface as
  creatures — AI zone classification/avoidance.
- **Active restrictors (F92):** `sr_teleport_sections.ltx` is the data for the `sr_teleport` level-changer
  scheme (`flows/restrictors-active-sr.md`).
- **LTX mechanism:** `#include` assembly + `[child]:parent` inheritance + DLTX deltas →
  `modlayer/config-ltx.md`.
- **Presets (F122):** `postprocess = postprocess_*` references a post-process preset.

## Failure & edge cases

- **Read tiers bottom-up:** a `_weak/_average/_strong` section overrides only a couple of keys; effective
  values live in its parent chain back to `zone_base`.
- **Config artefact fields ≠ live spawning** — the binder (F90) drives Anomaly's artefact economy; the
  engine fields are largely declarative here.
- **`sr_teleport_sections.ltx` is not a zone** despite living in `configs/` next to them — it's
  level-changer data for F92.
- **DLTX, not full overrides:** mods patch zones with small `.ltx` deltas; expect **no `gamedata/`
  full copies** (none at writing). If a full `gamedata/configs/zones/` file appears, it's heavy-handed —
  flag it.
- **Two `script_binding` paths:** anomaly fields → `bind_anomaly_field`; campfire → its own binder. Don't
  conflate.

## Deeper

- Include manifest + root: `unpacked/configs/zones/zones.ltx`, `zone_base.ltx`
- Family exemplars: `zone_mine_gravitational.ltx` (richest blowout), `zone_field_radioactive.ltx` /
  `zone_field_psychic.ltx` (passive DoT), `zone_teleport.ltx`, `zone_monolith.ltx`, `zone_campfire.ltx`
- Level-changer table: `unpacked/configs/sr_teleport_sections.ltx`
- Engine zone runtime: `flows/anomaly-zones.md` (F27); artefact economy: `flows/anomaly-field-binders.md`
  (F90); teleport scheme: `flows/restrictors-active-sr.md` (F92); hit type: `flows/damage-hit.md` (F30)
- Map: `modlayer/gameplay/configs-data.md` (M23); LTX mechanism: `modlayer/config-ltx.md`

## Maintenance

- Exemplars read in full; the rest of the catalog reads as variants on the same parameter block. If a mod
  adds a new `class = ZS_*`, confirm the engine subclass exists in `flows/anomaly-zones.md` before
  documenting it.
- Re-check `gamedata/configs/zones/` and `gamedata/configs/sr_teleport_sections.ltx` for overrides per
  session (none at writing).
- Keep the three load-bearing notes: `zone_base.script_binding` = the artefact-binder seam, config
  artefact fields ≠ live spawning (F90 owns it), and `sr_teleport_sections.ltx` is F92 data not an anomaly.
