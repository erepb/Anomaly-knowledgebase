---
domain: flows/restrictors-passive-sr
tags: [modlayer, gameplay, unpacked, lua, schemes, zone, flow]
sources: [unpacked/scripts/sr_idle.script, unpacked/scripts/sr_silence.script, unpacked/scripts/sr_no_weapon.script, unpacked/scripts/sr_timer.script, unpacked/scripts/sr_postprocess.script, unpacked/scripts/sr_light.script, unpacked/scripts/sr_particle.script, unpacked/scripts/sr_psy_antenna.script, unpacked/scripts/sr_deimos.script]
related: [flows/scheme-engine-core.md, flows/scheme-walker-patrol.md, flows/camera-pp-effectors.md, flows/anomaly-zones.md, flows/logic-ltx-switching.md, flows/binder-architecture.md, modlayer/gameplay/restrictors-sr.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: passive restrictor schemes (`sr_*`) (F91)

The **passive `sr_*` space-restrictor schemes** — ambient-effect logic that runs while the actor (or an NPC)
is inside a space restrictor: silence zones, weapon-lock zones, timers, post-process tints, light control,
particle emitters, psy-emission antennas, and the Deimos disorientation zone. Each is the **F66 scheme
pattern** (`set_scheme`/`add_to_binder`/`action`) applied to a restrictor (`stype_restrictor`), not a binder:
the action's `update` applies its passive effect each frame and calls
`xr_logic.try_switch_to_another_section` to honor logic-LTX switches. The effects mostly drive the F31
camera/PP-effector + sound APIs.

**Provenance:** **verified** — `sr_idle.script` read in full (the canonical template); the other 8
characterized by class/function structure + their key engine-effect calls (grepped). The bigger ones
(`sr_psy_antenna` 482 lines, `sr_deimos` 183) have their **mechanism** verified (psy-hit/phantom/pp; intensity
+ pp/cam effectors) but full bodies are *(agent-traced)*. **No `gamedata/` override** of any (verified by
listing).

## The scheme template (verified in `sr_idle`)

Every `sr_*` is: `set_scheme(npc, ini, scheme, section, gulag)` → `xr_logic.assign_storage_and_bind` +
`st.logic = cfg_get_switch_conditions` (the switch list, F68); `add_to_binder` → new an `action`,
`xr_logic.subscribe_action_for_events`; the `action:update(delta)` applies the effect then
`xr_logic.try_switch_to_another_section`. `sr_idle` does **only** the switch check — it's the do-nothing
template; the rest add an effect in `update`. This is the same shape as the NPC scheme template (F74), with
the object being a space restrictor instead of an NPC.

## The schemes (verified where noted)

- **`sr_idle`** *(verified)* — do nothing but check for a section switch. The minimal template.
- **`sr_silence`** *(verified structure)* — `CSilence_zone`: a zone that **suppresses combat music/sounds**
  while active (a silence registry the music system honors).
- **`sr_no_weapon`** *(verified structure)* — **locks weapons** while the actor is inside: `zone_enter`/
  `zone_leave` + `switch_state` force the weapon holstered (the "no weapons here" zones).
- **`sr_timer`** *(verified structure)* — `action_timer`: a count-up/down **timer** displayed on the HUD
  (`time2str` formatting), for scripted timed events; `deactivate` on expiry.
- **`sr_postprocess`** *(verified structure)* — applies a custom `PPEffector` (post-process **tint/intensity**)
  + `level.set_snd_volume`; `update_hit` ties damage to intensity. A scripted screen-effect zone.
- **`sr_light`** *(verified structure)* — `action_light` + `check_light`/`check_stalker`: controls
  **artificial light** (NPC torches/lamps) by `level.name`/`level.get_time_hours` — e.g. NPCs light up in
  dark indoor zones.
- **`sr_particle`** *(verified structure)* — plays a `particles_object` (**ambient particle emitter**);
  mode 1 = looped, mode 2 = on actor proximity; `is_end`/`deactivate` manage its lifetime.
- **`sr_psy_antenna`** *(mechanism verified; bodies agent-traced)* — `PsyAntenna`: the **psy-emission zone** —
  `update_psy_hit` (psy damage), `generate_phantoms`, `update_sound` (muffle via `level.set_snd_volume`),
  `update_postprocess` + `level.add_pp_effector` (visual distortion). The classic psy-field hazard.
- **`sr_deimos`** *(mechanism verified; bodies agent-traced)* — `CDeimos`: the **Deimos disorientation zone** —
  an `intensity` that builds, driving `level.add_pp_effector`/`add_cam_effector`; `check_intensity_delta`
  (intensity rises/falls with the actor's movement & view) + `check_disable_bound` — the "keep moving / don't
  look at it" mechanic; removes effectors on exit.

## Function digests (the template)

- **`set_scheme`** — *Does:* `assign_storage_and_bind` + `st.logic = cfg_get_switch_conditions`. *(uniform.)*
- **`add_to_binder`** — *Does:* new the action + `subscribe_action_for_events`. *(uniform.)*
- **`action:update(delta)`** — *Does:* apply the scheme's passive effect (effector/sound/particle/weapon
  lock/psy/intensity) then `try_switch_to_another_section`. *(per-scheme; `sr_idle` = switch only.)*
- **`action:reset_scheme` / `deactivate`** — *Does:* clear signals / tear down the effect on section switch.

## Isolation ledger (the seams)

- **Scheme engine (F66/F68):** all register via `set_scheme`/`add_to_binder`; `st.logic` is the switch list;
  `try_switch_to_another_section` is the per-frame switch check. They run on the `stype_restrictor` scheme
  type, loaded by the scheme registry (F65).
- **Effector/sound seam (F31):** `sr_postprocess`/`sr_psy_antenna`/`sr_deimos` use `level.add_pp_effector`/
  `add_cam_effector`/`remove_*` + `level.set_snd_volume`/`set_snd_volume` (`flows/camera-pp-effectors.md`).
- **Zone seam (F27):** the restrictor object is an engine space restrictor (`se_restrictor`, F84); these
  schemes add Lua behavior on top of the C++ zone (`flows/anomaly-zones.md` for hazard zones).
- **Particle seam:** `sr_particle` drives a `particles_object`.
- **Music/weapon seams:** `sr_silence` feeds the music system; `sr_no_weapon` forces the weapon state.

## Threading / failure / edge cases

- **They're schemes, not binders:** logic comes from a logic-LTX section on the restrictor (F68); a passive
  effect that doesn't fire usually means the wrong/no section is active, not a binder bug. Debug via the
  active section + condlists.
- **Effectors must be removed on exit:** `sr_deimos`/`sr_psy_antenna`/`sr_postprocess` add pp/cam effectors —
  their `deactivate`/exit paths remove them (`level.remove_pp_effector`/`remove_cam_effector`). A stuck screen
  effect indicates a missed teardown on section switch.
- **`sr_deimos` intensity is actor-behavior-driven:** intensity rises when the actor looks at / approaches and
  falls when moving away — the gameplay is "keep moving, don't stare." Not a fixed timer.
- **`sr_no_weapon` forces holster:** scripts elsewhere can't equip a weapon while inside; the lock is the zone,
  not the inventory.
- **No `gamedata/` override** at writing — re-check per session.

## Deeper

- Template: `unpacked/scripts/sr_idle.script` (`set_scheme`/`add_to_binder`/`action_idle`)
- Effect zones: `sr_postprocess.script` (`PPEffector`), `sr_psy_antenna.script` (`PsyAntenna`),
  `sr_deimos.script` (`CDeimos`), `sr_particle.script`, `sr_light.script`
- Gameplay zones: `sr_silence.script` (`CSilence_zone`), `sr_no_weapon.script`, `sr_timer.script`
- Scheme engine: `flows/scheme-engine-core.md` (F66), `flows/logic-ltx-switching.md` (F68); template parallel:
  `flows/scheme-walker-patrol.md` (F74); effectors: `flows/camera-pp-effectors.md` (F31)
- Active `sr_*` (camp/cutscene/teleport/monster/crow_spawner): F92; map `modlayer/gameplay/restrictors-sr.md` (M10)

## Maintenance

- Verified core (the `sr_*` scheme template + each scheme's effect role) is stable. Promote `sr_psy_antenna`/
  `sr_deimos`/`sr_postprocess` bodies from agent-traced when a task needs exact psy/intensity/effector math.
- These are schemes on the F66 engine — if the scheme template changes, re-verify against `sr_idle`.
- No `gamedata/` override at writing; the logic-LTX + zone configs are the tuning surface.
