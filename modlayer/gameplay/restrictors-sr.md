---
domain: modlayer/gameplay/restrictors-sr
tags: [modlayer, gameplay, unpacked, lua, schemes, zone, map]
sources: [unpacked/scripts/sr_idle.script, unpacked/scripts/sr_silence.script, unpacked/scripts/sr_no_weapon.script, unpacked/scripts/sr_timer.script, unpacked/scripts/sr_postprocess.script, unpacked/scripts/sr_light.script, unpacked/scripts/sr_particle.script, unpacked/scripts/sr_psy_antenna.script, unpacked/scripts/sr_deimos.script, unpacked/scripts/sr_camp.script, unpacked/scripts/sr_cutscene.script, unpacked/scripts/sr_teleport.script, unpacked/scripts/sr_monster.script, unpacked/scripts/sr_crow_spawner.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/restrictors-passive-sr.md, flows/scheme-engine-core.md, flows/scheme-walker-patrol.md, flows/camera-pp-effectors.md, flows/anomaly-zones.md, flows/se-others.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Restrictor schemes (`sr_*`) — L1 map

The **space-restrictor scheme** family. An `sr_*` is the F66 scheme pattern (`set_scheme`/`add_to_binder`/
`action`) applied to an engine space restrictor (`se_restrictor`, F84) rather than an NPC — Lua behavior on a
volume of space. They split into **passive** (ambient effects while inside: silence/no-weapon/timer/
post-process/light/particle/psy/deimos — F91) and **active** (drive events/scripts: camp/cutscene/teleport/
monster-spawn/crow-spawn — F92). This map routes the family; the scheme template + each scheme's role live in
the flows.

**Read first:** `flows/restrictors-passive-sr.md` (F91) — the `sr_*` scheme template (`sr_idle`) + the
passive catalog. Always grep `gamedata/scripts/` for an override.

## The family (verified by listing `unpacked/scripts/sr_*`)

| Scheme | Kind | Role | Flow |
|--------|------|------|------|
| `sr_idle` | passive | do-nothing template (switch check only) | F91 ✅ |
| `sr_silence` | passive | suppress combat music/sounds | F91 ✅ |
| `sr_no_weapon` | passive | lock weapons (force holster) | F91 ✅ |
| `sr_timer` | passive | HUD count-up/down timer | F91 ✅ |
| `sr_postprocess` | passive | post-process tint + sound vol | F91 ✅ |
| `sr_light` | passive | NPC artificial-light control | F91 ✅ |
| `sr_particle` | passive | ambient particle emitter | F91 ✅ |
| `sr_psy_antenna` | passive | psy-emission hazard (psy hit/phantoms/distortion) | F91 ✅ |
| `sr_deimos` | passive | Deimos disorientation zone (intensity → pp/cam fx) | F91 ✅ |
| `sr_camp` | (manager) | NPC camp director (`CCampManager`, used by `bind_camp` F89) | F92 ✅ |
| `sr_cutscene` | active | scripted camera cutscene (`cam_effector_set`) | F92 ✅ |
| `sr_teleport` | active | teleport the actor (weighted-random point) | F92 ✅ |
| `sr_monster` | active | spawn/reveal scripted monster on entry | F92 ✅ |
| `sr_crow_spawner` | active | spawn ambient crows | F92 ✅ |

## How it fits (verified in F91)

1. A logic-LTX section on a restrictor names an `sr_*` scheme; the scheme registry (F65) loads it as
   `stype_restrictor`.
2. `set_scheme` binds storage + the switch list (F68); `add_to_binder` subscribes the `action`.
3. Each frame `action:update` applies the scheme's effect (passive) or fires its event (active), then
   `xr_logic.try_switch_to_another_section` honors logic switches.
4. Effect schemes drive the F31 effector/sound APIs (`level.add_pp_effector`/`add_cam_effector`/`set_snd_volume`).

## Subsystem deep-dives (delegates to flows)

- Passive `sr_*` (effects/hazards) → `flows/restrictors-passive-sr.md` (F91) ✅
- Active `sr_*` (camp/cutscene/teleport/monster/crow_spawner) → `flows/restrictors-active-sr.md` (F92) ✅
- Scheme engine + template → `flows/scheme-engine-core.md` (F66), `flows/scheme-walker-patrol.md` (F74)
- Effectors → `flows/camera-pp-effectors.md` (F31); zones → `flows/anomaly-zones.md` (F27); restrictor server
  object → `flows/se-others.md` (F84, `se_restrictor`)

## Maintenance

- This map is the `sr_*` routing hub; `ROADMAP-gameplay.md` (Phase G4, F91/F92) is the authoritative
  plan/status. **Both F91 (passive) + F92 (active) done — the `sr_*` family is fully documented.**
- `sr_camp` is the camp manager used by `bind_camp` (F89) — it's a manager, not a typical restrictor scheme.
- Re-list `unpacked/scripts/sr_*` + check `gamedata/` overrides per session.
