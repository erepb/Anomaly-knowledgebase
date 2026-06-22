---
domain: flows/restrictors-active-sr
tags: [modlayer, gameplay, unpacked, lua, schemes, zone, flow]
sources: [unpacked/scripts/sr_teleport.script, unpacked/scripts/sr_crow_spawner.script, unpacked/scripts/sr_monster.script, unpacked/scripts/sr_cutscene.script, unpacked/scripts/sr_camp.script, unpacked/scripts/bind_camp.script]
related: [flows/restrictors-passive-sr.md, flows/scheme-engine-core.md, flows/camera-pp-effectors.md, flows/special-actors.md, flows/world-object-binders.md, flows/anomaly-zones.md, flows/zones-anomaly-configs.md, flows/alife-services.md, modlayer/gameplay/restrictors-sr.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: active restrictor schemes (`sr_*`) (F92)

The **active `sr_*` space-restrictor schemes** — restrictors that **fire events** when the actor enters
(teleport, spawn monsters/crows, play a cutscene) rather than applying a passive ambient effect (F91). Same
F66 scheme template (`set_scheme`/`add_to_binder`/`action`) as the passive ones; the difference is the
payload: an actor-state change, an `alife_create` spawn, or a camera takeover. Plus `sr_camp`, the NPC camp
director (`CCampManager`) that `bind_camp` (F89) instantiates.

**Provenance:** **verified** — `sr_teleport.script` + `sr_crow_spawner.script` read in full; `sr_monster`
(`fake_monster`), `sr_cutscene` (`cam_effector_set`/`action_cutscene`), `sr_camp` (`CCampManager`)
characterized by class/function structure + key engine calls (grepped). The bigger bodies (cutscene 361 L,
camp 360 L) have their **mechanism** verified, full bodies *(agent-traced)*. **No `gamedata/` override** of
any (verified by listing).

## The schemes (verified where noted)

### `sr_teleport` *(verified, full)* — teleport the actor
`action_teleport:update`: when the actor is `inside` the restrictor → add `teleport.ppe` effector + start a
timer; after `st.timeout` → pick a **weighted-random** point from `st.points` (`prob`-weighted),
`db.actor:set_actor_position(patrol(point))` + `set_actor_direction` + a tinnitus sound, reset to idle. The
canonical actor-event scheme. Config: `point1..10`/`look1..10`/`prob1..10` + `timeout`.

### `sr_crow_spawner` *(verified, full)* — ambient crows
`crowkiller:update`: throttled (`time_for_spawn`), suppressed during surge/psi-storm, spawns up to
`max_crows_on_level` (default 7) crows — `check_for_spawn_new_crow` picks a random `spawn_path`, and if it's
far enough from the actor (>100 m) `alife_create("m_crow", …)` there. Feeds `bind_crow.crow_counter`
(`flows/special-actors.md` F36). The crowkiller minigame spawner.

### `sr_monster` *(mechanism verified; body agent-traced)* — scripted monster ambush
`fake_monster:update`: tracks `is_actor_inside`; when the actor enters and `st.monster` is configured, it
**spawns/reveals a scripted monster** + plays an appear sound (e.g. a bloodsucker scare). The scripted-encounter
zone.

### `sr_cutscene` *(mechanism verified; body agent-traced)* — scripted camera cutscene
`cam_effector_set` + `action_cutscene`: plays camera animations via
`level.add_cam_effector("camera_effects\\<anm>.anm", 210408, false, "sr_cutscene.effector_callback", fov)`,
selecting effect sets (`start`/`idle`) by condlist (`enabled_condlist`/`effect_condlist`); `stop_effect` →
`level.remove_cam_effector(210408)`. The in-engine cutscene system (camera takeover, F31).

### `sr_camp` *(mechanism verified; body agent-traced)* — NPC camp director
`CCampManager`: the campfire-gathering social system — NPCs around a camp take turns at **storytelling**
(`story_table` `s_story_1..11`), **guitar** themes, and **harmonica**, choosing a director/listener role and
the next activity. **Instantiated by `bind_camp`** (F89, `camps[id]`), not by the scheme registry — `sr_camp`
is a manager class, not a typical restrictor `action`.

## Function digests (spine)

- **`action_teleport:update`** — *Does:* inside→timer→weighted-random `set_actor_position`+direction+sound.
- **`crowkiller:update` / `check_for_spawn_new_crow`** — *Does:* throttled `alife_create("m_crow")` up to cap,
  away from actor, off during emissions.
- **`fake_monster:update`** — *Does:* spawn/reveal `st.monster` on actor entry + appear sound.
- **`cam_effector_set:start_effect`/`select_effect`** — *Does:* play `.anm` camera effectors, condlist-selected.
- **`CCampManager`** — *Does:* the camp storytelling/guitar/harmonica round-robin (via `bind_camp`).
- **`set_scheme`/`add_to_binder`** — *Does:* the uniform F66 template (storage + switch list + subscribe action).

## Isolation ledger (the seams)

- **Scheme engine (F66/F68):** all but `sr_camp` register via `set_scheme`/`add_to_binder` and `update` →
  `try_switch_to_another_section`; loaded as `stype_restrictor` (F65). `sr_camp` is a manager class run by
  `bind_camp` (F89).
- **Actor seam (F6):** `sr_teleport` mutates the actor (`set_actor_position`/`set_actor_direction`).
- **Effector seam (F31):** `sr_teleport` (pp) + `sr_cutscene` (cam) use `level.add_*_effector`/`remove_*`.
- **ALife spawn seam (F41):** `sr_crow_spawner`/`sr_monster` `alife_create` server objects.
- **Special-actors seam (F36):** crows (`bind_crow.crow_counter`).
- **Zone seam (F27/F84):** the restrictor is an engine space restrictor (`se_restrictor`).

## Threading / failure / edge cases

- **Effector ids are fixed:** `sr_teleport` uses pp id 2006, `sr_cutscene` cam id 210408 — a stuck cutscene
  is usually a missed `remove_cam_effector(210408)`; debug the effector teardown.
- **Crow spawn is gated three ways:** time throttle + crow cap + actor-distance + emission-suppression — crows
  not appearing is usually one of these gates, not a broken scheme.
- **Teleport is probabilistic + has a delay:** the actor isn't teleported instantly on entry — there's a
  `timeout` and a weighted point pick; `prob` weights skew which exit you get.
- **`sr_camp` is a manager, not a scheme action:** it has no `set_scheme`/`add_to_binder` like the others —
  `bind_camp` news it. Looking for it in the scheme registry will fail.
- **`sr_monster` reveals a configured monster:** `st.monster` must be set (a spawned server monster) — without
  it the zone does nothing on entry.
- **No `gamedata/` override** at writing — re-check per session.

## Deeper

- Actor events: `unpacked/scripts/sr_teleport.script` (`action_teleport`), `sr_cutscene.script`
  (`cam_effector_set`/`action_cutscene`)
- Spawners: `sr_crow_spawner.script` (`crowkiller`), `sr_monster.script` (`fake_monster`)
- Camp director: `sr_camp.script` (`CCampManager`) — driven by `bind_camp.script` (F89)
- Passive `sr_*` + template: `flows/restrictors-passive-sr.md` (F91); scheme engine: `flows/scheme-engine-core.md`
  (F66); effectors: `flows/camera-pp-effectors.md` (F31); crows: `flows/special-actors.md` (F36)
- Map: `modlayer/gameplay/restrictors-sr.md` (M10)

## Maintenance

- Verified core (teleport + crow spawner full; monster/cutscene/camp mechanism) is stable. Promote the
  cutscene effect-set selection + `CCampManager` round-robin bodies from agent-traced when a task needs them.
- `sr_camp`/`CCampManager` is shared with `bind_camp` (F89) — keep the cross-link.
- No `gamedata/` override at writing; logic-LTX + camp/cutscene configs are the tuning surface.
