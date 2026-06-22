---
domain: flows/anomaly-field-binders
tags: [modlayer, gameplay, unpacked, lua, lifecycle, zone, artefact, flow]
sources: [unpacked/scripts/bind_anomaly_zone.script, unpacked/scripts/bind_anomaly_field.script, unpacked/configs/plugins/dynamic_anomalies.ltx]
related: [flows/binder-architecture.md, flows/world-object-binders.md, flows/anomaly-zones.md, flows/zones-anomaly-configs.md, flows/detector-artefact.md, flows/db-registry.md, flows/alife-services.md, flows/save-load.md, flows/relations-info.md, modlayer/gameplay/binders.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: anomaly-field binders (F90)

The two `bind_anomaly_*` files — the Lua layer over engine anomaly zones (C++ `CCustomZone`, F27). They
contain **three distinct things**: the **artefact-economy binder** `anomaly_zone_binder` (the Lua path that
actually **spawns artefacts** in anomaly zones), the **per-field binder** `anomaly_field_binder` (a thin
binder that registers a field so zones can enable/disable it), and the **Dynamic Anomalies system** (module
functions in `bind_anomaly_field` that periodically spawn/relocate anomalies across the map from
`dynamic_anomalies.ltx`). Plus the `anomaly_on_before_activate` veto (the Lua side of F27's activation hook).

**Resolves F27's open point:** `flows/anomaly-zones.md` (F27) noted the C++ zone does **not** spawn artefacts
(only commented ALife). **F90 is where artefacts actually spawn** — in `anomaly_zone_binder:spawn_artefact_randomly`.

**Provenance:** **verified** — `anomaly_zone_binder` (`__init`, `spawn_artefact_randomly`, `refresh`,
`respawn_artefacts_and_replace_anomaly_zone`, `disable_anomaly_fields`, `turn_on/off`, `on_artefact_take`,
`net_spawn`) read; `anomaly_field_binder` read in full; the `dyn_anomalies_spawn/suffle/update/refresh`
system + `anomaly_on_before_activate` read. Pulse-anomaly functions *(agent-traced)*. **No `gamedata/`
override** of either file (verified by listing). Both attach via the F88 `bind` hook.

## Concepts (verified)

- **anomaly zone (Lua)** — `anomaly_zone_binder`, the manager over a zone's **layers** of anomaly fields +
  its **artefact economy**. A zone picks one random `layer_N` per session; each layer defines its anomaly
  fields, mines, artefact list (+ chance coeffs), spawn patrol-paths (`artefact_ways`), and caps
  (`max_artefacts`/`respawn_tries`). *(verified in `__init`.)*
- **anomaly field** — `anomaly_field_binder`, the individual field object. Registers in
  `bind_anomaly_field.fields_by_names[name]`; `set_enable(b)` → engine `enable_anomaly()`/`disable_anomaly()`.
  The damage logic is the C++ `CCustomZone` (F27); this binder is just registration + the on/off toggle.
- **dynamic anomalies** — anomalies that **spawn and relocate across the map** after each emission, driven by
  `plugins/dynamic_anomalies.ltx` (positions per level) — a system of module functions, **not** a binder.

## Sequence (spine)

### Artefact spawning — `anomaly_zone_binder` (verified)
`__init` parses the `anomal_zone` cfg into per-layer tables (artefacts/coeffs/ways/fields/mines/forces/caps)
and picks `cur_layer = layer_random`. `net_spawn` → `db.add_anomaly` + `db.add_obj` + `refresh`. **`refresh`**
spawns up to `max_artefacts − spawned_count` artefacts (when `respawn_artefacts` is set) then
`disable_anomaly_fields`. **`spawn_artefact_randomly`** is the economy core: a spawn-chance =
`game_difficulties.get_eco_factor("arty_chance")` × surge/psi-storm multiplier (+geologist achievement); if
it passes, pick an artefact type from the layer's list → a concrete artefact from `arty_collection[type]` →
`alife_create_item` at a random point on a random `artefact_ways` patrol path (+1 m up); register the new
artefact in `artefact_ways_by_id`/`parent_zones_by_artefact_id`. **`on_artefact_take`** decrements
`spawned_count` and clears the registries when the player takes one (F24 detector/artefact). **`turn_off`**
releases all spawned artefacts (`alife_release_id`); **`turn_on`** re-enables. **`disable_anomaly_fields`**
enables only the *current* layer's fields/mines (via `fields_by_names[name]:set_enable`) and disables the
rest — so a zone presents one layer's anomaly arrangement at a time.

### Layer replacement — `respawn_artefacts_and_replace_anomaly_zone` (verified)
On a surge / psi-storm: bump `s_chance`, disable the current layer's fields/mines, pick a **new random
layer**, enable its fields/mines, update caps/forces, `refresh`. This is how emissions reshuffle a zone's
anomalies + restock its artefacts.

### Per-field binder — `anomaly_field_binder` (verified, full)
`reinit` → `db.storage[id]`; `net_spawn` → `db.add_zone` + `db.add_obj` + register in `fields_by_names[name]`;
`net_destroy` clears all. `set_enable(b)` toggles the engine anomaly. (The commented `set_restrictor_type`
shows the disabled "AI dies in anomaly" path — left off for FPS.) Thin: the field's hazard is C++ (F27).

### Dynamic anomalies — module functions in `bind_anomaly_field` (verified)
`ini_settings` loads `dynamic_anomalies.ltx` into `dyn_ano_info` (positions per level) + `dyn_ano_type` (type
→ candidate sections). **`dyn_anomalies_spawn`** `alife_create`s an anomaly per configured slot (setting its
shape via `utils_stpk`), recording `dyn_anomalies[lvl][id]`. **`dyn_anomalies_suffle`** randomly flips each to
active/inactive (`dyn_ano_chance`) — the relocation. **`dyn_anomalies_update`** enables/disables the online
objects by state (only when the actor is far enough, `dyn_ano_safe_dist`) + adds map markers.
**`dyn_anomalies_refresh(force)`** first-time spawns+shuffles (gated by the `dynamic_anomalies_spawned` info),
or reshuffles on `force` (after an emission), then schedules `dyn_anomalies_update` via `CreateTimeEvent`.

### Activation veto — `anomaly_on_before_activate(zone, obj, flags)` (verified)
The Lua side of F27's `CCustomZone_BeforeActivateCallback`: dead stalkers/monsters don't trigger anomalies
(`flags.ret_value = false`); among non-creatures only `obj_bolt` activates (bolts probe anomalies), everything
else is vetoed.

## Function digests (spine)

- **`anomaly_zone_binder:spawn_artefact_randomly`** — *Does:* chance-gated artefact spawn at an
  `artefact_ways` point; registers it. *Reads:* `arty_collection`, `game_difficulties`, surge/psi state.
- **`anomaly_zone_binder:refresh` / `respawn_artefacts_and_replace_anomaly_zone`** — *Does:* restock to cap /
  surge-driven layer swap + restock.
- **`anomaly_zone_binder:disable_anomaly_fields`** — *Does:* enable only current-layer fields/mines.
- **`anomaly_zone_binder:on_artefact_take` / `turn_off` / `turn_on`** — *Does:* take bookkeeping / release-all /
  re-enable.
- **`anomaly_field_binder:net_spawn` / `set_enable`** — *Does:* register in `fields_by_names`; engine
  enable/disable.
- **`dyn_anomalies_spawn/suffle/update/refresh`** — *Does:* the map-wide dynamic-anomaly spawn/relocate cycle.
- **`anomaly_on_before_activate`** — *Does:* veto anomaly activation (dead creatures / non-bolt objects).

## Isolation ledger (the seams)

- **Binder base (F88):** both subclass `object_binder`, attached via `bind`.
- **Engine zone seam (F27):** the field hazard/blowout is C++ `CCustomZone` (`flows/anomaly-zones.md`);
  `enable_anomaly`/`disable_anomaly`, `anomaly_on_before_activate` veto. **F90 owns the artefact-spawn +
  layer + dynamic-relocation that the C++ side lacks.**
- **Artefact seam (F24):** spawned via `alife_create_item` from `arty_collection`; `on_artefact_take` pairs
  the detector/artefact take (`flows/detector-artefact.md`).
- **Registry seam (F69):** `db.add_anomaly`/`add_zone`/`add_obj`; module registries `fields_by_names`,
  `artefact_ways_by_id`, `parent_zones_by_artefact_id`, `dyn_anomalies`.
- **ALife/emission seam (F41):** surge/psi-storm chances drive `spawn_artefact_randomly` + layer replacement;
  dynamic-anomaly reshuffle on emission.
- **Config seam:** zone cfg from each zone's `anomal_zone/cfg`; dynamic anomalies from
  `plugins/dynamic_anomalies.ltx` + the `dynamic_anomalies` option (F17).
- **Callback bus (F1):** `anomaly_on_before_activate`, `actor_on_first_update`/`_update`/`_interaction`,
  `save_state`/`load_state`.

## Threading / failure / edge cases

- **Artefacts are spawned by Lua, not the engine zone:** if artefacts stop appearing, debug
  `anomaly_zone_binder` (chance math, `artefact_ways`, `arty_collection`), **not** the C++ zone (F27).
- **Layers = alternate arrangements:** only the current layer's fields are enabled; a field that "isn't
  working" may belong to an inactive layer. Emissions (`respawn_…replace`) swap the layer.
- **`spawned_count` can desync:** `on_artefact_take` floors it at 0; a wrong count caps respawns. The
  `parent_zones_by_artefact_id` map links an artefact back to its zone for take handling.
- **Dynamic anomalies wait for online registration:** `dyn_anomalies_refresh` schedules `update` with a delay
  (`CreateTimeEvent` n=10 first time) because online objects don't exist instantly after server spawn.
- **Activation veto is global:** `anomaly_on_before_activate` makes only bolts (and live creatures) trip
  anomalies — dropped items/corpses don't, by design.
- **No `gamedata/` override** at writing — re-check; the `.ltx` (zone cfg + `dynamic_anomalies.ltx`) is the
  tuning surface.

## Deeper

- Artefact economy: `unpacked/scripts/bind_anomaly_zone.script` (`anomaly_zone_binder`,
  `spawn_artefact_randomly`, `respawn_artefacts_and_replace_anomaly_zone`, `on_artefact_take`)
- Field binder + dynamic anomalies: `unpacked/scripts/bind_anomaly_field.script` (`anomaly_field_binder`,
  `fields_by_names`, `dyn_anomalies_*`, `anomaly_on_before_activate`); config
  `unpacked/configs/plugins/dynamic_anomalies.ltx`
- Engine zone (hazard/blowout): `flows/anomaly-zones.md` (F27); artefacts/detectors: `flows/detector-artefact.md` (F24)
- Pattern: `flows/binder-architecture.md` (F88); other world binders: `flows/world-object-binders.md` (F89)
- Map: `modlayer/gameplay/binders.md` (M7)

## Maintenance

- Verified core (the artefact-spawn economy, layers, dynamic-anomaly cycle, activation veto) is stable. The
  pulse-anomaly functions are agent-traced — promote when a task needs them.
- This flow resolves F27's "where do artefacts spawn" — keep the cross-link if either changes.
- No `gamedata/` override at writing; the zone cfg + `dynamic_anomalies.ltx` are the tuning surface.
