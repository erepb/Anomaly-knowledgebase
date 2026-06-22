---
domain: flows/scheme-camper-cover
tags: [modlayer, gameplay, unpacked, lua, schemes, camper, cover, flow]
sources: [unpacked/scripts/xr_camper.script, unpacked/scripts/xr_cover.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-combat.md, flows/scheme-engine-core.md, flows/state-manager-core.md, flows/smart-cover.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: camper / cover schemes (F75)

Two position-holding schemes — **deltas on the walker template** (F74 `flows/scheme-walker-patrol.md`).
**camper** = an NPC that holds a patrol point and scans/fires (with a sniper sub-mode), transitioning to
the engine combat planner when an enemy breaches its radius. **cover** = an NPC that finds and moves to a
nearby cover vertex, then plays a pose. Read F74 first; only the deltas are here.

**Provenance:** **cover is verified** in `xr_cover.script` (`set_scheme`, `action_cover:activate_scheme`
with the `best_cover` call). **camper is agent-traced** *(not personally re-read this session)* — its
config + sniper-scan + combat-handoff are reported from the research draft and follow the F74 pattern.
No `gamedata/` overrides of either file (verified by listing).

## Cover — verified

- **`xr_cover.script:set_scheme`** (verified) reads: `smart`/`path_center` (the base point), `anim`
  (a condlist pose, default "hide"), `radius_min`/`radius_max` (search ring, default 3/7), `sound_idle`.
  No `move_mgr` setup; the action drives navigation directly.
- **`xr_cover.script:action_cover:activate_scheme`** (verified) selects a cover vertex: resolve a base
  point (`path_center` patrol vertex / `smart` level vertex / current position), pick a random radial
  direction at a `radius_min..radius_max` distance (`level.vertex_in_direction`), then
  `self.object:best_cover(self_random_position, enemy_random_position, cover_dist, 1, 150)` over cover
  distances; fall back to the random vertex if none; correct to the nearest accessible vertex; then
  `set_dest_level_vertex_id` + `state_mgr.set_state("assault")` to run there.
- **`action_cover:execute`** (verified entry) — on arrival (≤ ~0.4 m²) pick the pose via
  `xr_logic.pick_section_from_condlist(..., st.anm)` and `state_mgr.set_state(pose)`; else keep moving.
- Binder delta: a single `evaluator_need_cover` (→ `xr_logic.is_active`) + `action_cover`, with
  preconditions `property_danger=false`/`property_enemy=false` (so cover-seeking yields to combat) and
  the alife-gating precondition. Navigation is **engine pathfinding** (no `move_mgr`), unlike walker.

## Camper — agent-traced

- **`xr_camper.script:set_scheme`** reads walker's `path_walk`/`path_look` plus camper extras: `sniper`
  (bool — use `path_look` flags as scan targets, `sniper_update_rate(true)`), `radius` (enemy lock
  distance, default 20), `shoot` ("always"|"terminal"), `no_retreat`, `scantime_free`, `enemy_idle`,
  and `def_state_campering`/`_campering_fire`/`_moving_fire` state variants.
- **`action_patrol:execute`** runs two modes: no-enemy → `process_danger` + (sniper) interpolated scan
  between `path_look` waypoints, else `move_mgr` patrol; enemy-present → stance eval (stand/crouch via
  `high_cover_in_direction`) + `state_mgr.set_state` to a `threat_fire`/`hide_fire`/`*_sniper_fire`
  state, with sniper scatter-fire on a memory triangle.
- **Combat handoff:** an `evaluator_close_combat` (enemy within `radius` or `locked`) gates the engine
  `action_combat_planner` — when an enemy is close, the NPC drops camping and the engine combat AI takes
  over (ties to F76 `flows/scheme-combat.md`). The companion combat style `xr_combat_camper` (fire from
  cover, look-around) is documented in F76.

## Function digests (spine)

- **`xr_cover.script:set_scheme`** — *Does:* read base point + radius + anim. *(verified.)*
- **`xr_cover.script:action_cover:activate_scheme`** — *Does:* random-radial + `best_cover` vertex
  selection → navigate. *(verified.)*
- **`xr_cover.script:action_cover:execute`** — *Does:* arrive → pose; else move. *(verified entry.)*
- **`xr_camper.script:set_scheme` / `action_patrol:execute` / `evaluator_close_combat`** — *Does:* hold +
  scan + fire + combat handoff. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `db.storage[id][scheme]` (camper: scan_table, locked, scantime; cover: cover_vertex,
  anm). Same scheme framework (F66/F74).
- **Engine seams:** `best_cover` / `vertex_in_direction` / `high_cover_in_direction` (engine cover
  queries); the GOAP `action_combat_planner` (camper gates it, F76); `state_mgr` (F70).
- **Smart-cover:** distinct from F33 smart-covers — these use ad-hoc cover vertices, not registered
  smartcovers (animpoint F77 uses those).

## Threading / failure / edge cases

- **Cover yields to combat:** `property_enemy/danger=false` preconditions mean cover-seeking stops under
  threat; it is a peace-time reposition, not a combat behavior.
- **Camper radius lock (agent-traced):** once an enemy enters `radius` the camper stays put and fires
  rather than retreating (more so with `no_retreat`).
- **Sniper mode (agent-traced):** uses `path_look` waypoint flags as scan arcs; `sniper_update_rate`
  reduces AI tick cost.

## Deeper

- Cover: `unpacked/scripts/xr_cover.script` (`set_scheme`, `action_cover:activate_scheme`/`execute`)
- Camper: `unpacked/scripts/xr_camper.script` (`set_scheme`, `action_patrol:execute`,
  `evaluator_close_combat`)
- Template: `flows/scheme-walker-patrol.md`; combat handoff: `flows/scheme-combat.md`; animation:
  `flows/state-manager-core.md`

## Maintenance

- Cover spine verified; promote camper to verified on next read (the sniper scan + combat handoff are the
  parts to confirm).
- Check `gamedata/scripts/` for overrides before trusting (none at writing).
