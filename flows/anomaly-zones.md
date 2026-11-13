---
domain: flows/anomaly-zones
tags: [xrgame, zone, anomaly, feel-touch, blowout, hit, campfire, flow]
sources: [src/xrGame/CustomZone.cpp, src/xrGame/CustomZone.h, src/xrGame/zone_effector.cpp, src/xrGame/RadioactiveZone.cpp, src/xrGame/GraviZone.cpp, src/xrGame/MosquitoBald.cpp, src/xrGame/ZoneCampfire.cpp, src/xrGame/NoGravityZone.cpp]
related: [flows/damage-hit.md, flows/detector-artefact.md, flows/object-lifecycle.md, flows/anomaly-field-binders.md, flows/zones-anomaly-configs.md, engine/physics-collision.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: anomaly zones

How an anomaly zone detects what's inside it and periodically applies an effect (damage / impulse /
gravity), driven by a four-state pulsation cycle. Zones are `Feel::Touch` objects (the same mechanism
detectors use, `flows/detector-artefact.md`); the effect is delivered as a `SHit`
(`flows/damage-hit.md`). This is the world-hazard half of the artefact ecosystem — note that in this
code path **zones do not spawn artefacts** (see Failure cases).

**Provenance:** spine personally code-verified — the full class hierarchy (headers),
`CCustomZone : CSpaceRestrictor + Feel::Touch`, `CCustomZone::UpdateWorkload` (the state dispatch),
`UpdateCL` (fast-mode), `Power`/`RelativePower` (distance attenuation), `AffectObjects` (per-frame
guarded loop → `Affect`), and the `_G.CCustomZone_BeforeActivateCallback` Lua seam in `shedule_Update`.
The per-subclass `Affect` bodies, `UpdateBlowout` timed sub-events, the `feel_touch_*` bodies, and
`CreateHit`'s packet internals are confirmed homes but *(agent-traced — not personally re-read)*.

## Type tree (verified from headers)

- **`CCustomZone`** ← `CSpaceRestrictor` + `Feel::Touch` — the base: detection, state machine, hit
  generation. (`CSpaceRestrictor` = the shape/area object; see `engine/server-entities.md`.)
  - **`CRadioactiveZone`** — continuous low-power radiation while inside.
  - **`CNoGravityZone`** — physics-only (toggles gravity on enter/exit; no hit).
  - **`CVisualZone`** (intermediate, adds idle/attack animation) → **`CHairsZone`** (motion-triggered),
    **`CAmebaZone`** (`+ CPHUpdateObject`; velocity-limit + impulse).
  - **`CMosquitoBald`** (intermediate) → **`CTorridZone`** (burn + secondary hit), **`CZoneCampfire`**
    (scriptable on/off fire — *not* directly `CCustomZone`).
  - **`CBaseGraviZone`** → **`CGraviZone`** (pull-to-center + blowout throw + telekinesis).
- **`CFryupZone`** ← `CScriptObject` — **not a zone in this hierarchy** (a script object; ignore for
  zone mechanics).

## Detection — Feel::Touch (spine verified; bodies agent-traced)

A zone tracks occupants in `m_ObjectInfoMap` (a vector of `SZoneObjectInfo` — object ptr, time-in-zone,
last-affected time, small/nonalive/ignore flags, attached particles). The `Feel::Touch` callbacks
maintain it: `feel_touch_contact` filters candidates (shape contact; skips other zones / non-kinematics
/ flagged-ignore), `feel_touch_new` creates an `SZoneObjectInfo` + runs the `enter_Zone` hook + entry
particles, `feel_touch_delete` runs `exit_Zone` + removes it. `shedule_Update` calls
`feel_touch_update(center, radius)` to resync the list each tick. A Lua hook
**`_G.CCustomZone_BeforeActivateCallback(zone, object)`** can set an object's `zone_ignore` (verified —
the per-object script veto).

## Exposure / state cycle (state dispatch verified; sub-events agent-traced)

`CCustomZone::UpdateWorkload(dt)` is the engine of the zone: `m_iStateTime += dt`, then dispatch on
`m_eZoneState` —

- **`eZoneStateIdle`** — dormant; idle particles/light. `CheckForAwaking` flips to Awaking when a
  non-ignored object is present (subclasses like Hairs require a motion threshold).
- **`eZoneStateAwaking`** — wind-up; after `m_StateTime[Awaking]` → Blowout.
- **`eZoneStateBlowout`** — the active pulse: `UpdateBlowout` fires timed sub-events (particles → light
  → sound → wind ramp → **explosion = `AffectObjects()`**) and after `m_StateTime[Blowout]` → Accumulate.
- **`eZoneStateAccumulate`** — cooldown; after `m_StateTime[Accumulate]` → Blowout again (if still
  active) or Idle.
- **`eZoneStateDisabled`** — off (on/off timer or script).

`UpdateWorkload` also updates the **actor postprocess effector** (`m_actor_effector->Update(distance,
radius, m_eHitTypeBlowout)` — `zone_effector.cpp`, the screen warp/tint near a zone) and the optional
**secondary hit** while non-idle. `UpdateCL` calls `UpdateWorkload` only in **fast-mode** (zone near
camera); far zones tick on the slower scheduled path from `shedule_Update`.

## Applying the effect (verified loop; per-subclass Affect agent-traced)

`AffectObjects()` (verified): guarded to run **once per frame** (`m_dwAffectFrameNum == Device.dwFrame`),
iterates `m_ObjectInfoMap`, and calls **`Affect(&info)`** for each non-destroyed object. `Affect` is the
virtual each subclass overrides; it computes power by distance and emits a hit:

- **`CCustomZone::Power(dist, radius)`** (verified) = `m_fMaxPower * RelativePower(dist, radius)` —
  distance-attenuated (closer = stronger; `RelativePower` is the `1 - attenuation·(dist/radius)²`-style
  falloff clamped to [0,1]).
- the hit is sent via **`CreateHit(...)`** as an `SHit` of type `m_eHitTypeBlowout` (+ a physics impulse
  `m_fHitImpulseScale · power · mass`) into the normal damage pipeline (`flows/damage-hit.md`).
  *(packet body agent-traced.)*

Per-subclass `Affect` *(all agent-traced)*: **Radioactive** — continuous hit throttled to ~0.1 s via
`f_time_affected`; **Hairs** — impulse on blowout, only after motion-wake; **Ameba** — random impulse +
in-zone velocity clamp (`PhTune`); **Gravi** — pull-to-center (`AffectPull`) then outward throw on
blowout (`AffectThrow`) + scheduled telekinesis levitation in idle; **Torrid/MosquitoBald** —
distance impulse + `UpdateSecondaryHit` burn; **NoGravity** — no hit, toggles object gravity in
`enter_Zone`/`exit_Zone`. The hit type is `m_eHitTypeBlowout`, read from the zone's `.ltx` `hit_type`.

## Campfire (agent-traced)

`CZoneCampfire` (← `CMosquitoBald`) is a **scriptable** zone: `turn_on_script`/`turn_off_script` toggle
it; enabling/disabled particles + a smooth idle-light fade over a transition window; `UpdateWorkload`
keeps running while off (for the fade). It is how the campfire-sit / cook mechanic gets a controllable
fire hazard.

## Isolation ledger (the seams)

- **Feel::Touch:** the occupant set — shared mechanism with detectors (`flows/detector-artefact.md`),
  resynced each tick via `feel_touch_update`.
- **Damage:** `CreateHit` → `SHit` of `m_eHitTypeBlowout` into `flows/damage-hit.md` (so outfit/helmet
  protection by that hit type applies — `flows/outfit-helmet.md`). Physics impulse for ragdoll/throw.
- **Postprocess:** `zone_effector.cpp` (`CZoneEffector`) drives the actor screen effect by distance.
- **Physics:** impulse, gravity toggle (NoGravity), telekinesis (Gravi) — `engine/physics-collision.md`.
- **Particles/lights/sound:** idle/awake/blowout/accum sets, blowout wind ramp.
- **ALife/server:** the server-side `CSE_ALifeAnomalousZone` schedules the zone
  (`engine/server-entities.md`); `CScientificDetector` reads zones to draw them
  (`flows/detector-artefact.md`).
- **Lua callback:** `_G.CCustomZone_BeforeActivateCallback(zone, obj)` → per-object ignore veto.
- **Config read** (`.ltx` zone section): `hit_type` (→ `m_eHitTypeBlowout`), `max_start_power`
  (→ `m_fMaxPower`), `attenuation`, `effective_radius`, `hit_impulse_scale`, the state durations
  (`awaking_time`/`blowout_time`/`accamulate_time`), `ignore_nonalive`/`ignore_small`/`ignore_artefacts`,
  `disable_time*`, `use_secondary_hit`/`secondary_hit_power`, on/off timers, `postprocess`, and the
  particle/light/sound/grass-bend sets. Subclass-specific: Gravi `blowout_radius_percent`/`tele_*`,
  Ameba `max_velocity_in_zone`, Hairs `min_speed_to_react`, Campfire `enabling_particles`/
  `disabled_particles`/`disabled_sound`.

## Threading

Main thread: `shedule_Update` (deferred scheduler channel) and `UpdateCL`/`UpdateWorkload` (fast-mode,
near camera). Feel::Touch contact runs during update. No worker-thread interaction.

## Failure & edge cases

- **Floating artefacts (2026.10.11)** — `CBaseGraviZone::exit_Zone` deactivates telekinesis on a
  non-living physics object leaving the zone (and stops its tele particles), so objects don't stay
  suspended after exiting (OGSR port).
- **Zone volumes don't block NPC sight** — zone spatials carry `STYPE_FEELVISIONIGNORE`
  (`flows/ai-perception.md`).

- **Zones do NOT spawn artefacts in this code path.** The only artefact-spawn logic is commented-out
  ALife server code (`alife_anomalous_zone.cpp`); zone subclasses have no `SpawnArtefact`. Artefact
  placement is ALife/game-mode driven, not a zone effect — a common misconception. (The reverse —
  an *artefact* spawning a *zone* on activation — is real: `flows/detector-artefact.md`.)
- **Fast vs slow update:** far zones don't run `UpdateWorkload` from `UpdateCL`; they tick on the
  scheduled path. A zone that "does nothing" off-screen may simply be on the slow path.
- **`AffectObjects` is once-per-frame:** the `m_dwAffectFrameNum` guard means multiple call sites in one
  frame apply the hit only once; per-subclass throttling (Radioactive's 0.1 s) is separate.
- **`zone_ignore`:** flags (`ignore_nonalive`/`small`/`artefacts`) and the Lua veto can exclude objects;
  a zone that "won't hurt" something is often an ignore flag.
- **Hit type must match intent:** damage is entirely `m_eHitTypeBlowout`; outfit protection for that
  type fully governs survivability (a radiation zone vs radiation-protection).

## Deeper

- Core: `src/xrGame/CustomZone.cpp` (`UpdateWorkload`, `UpdateCL`, `shedule_Update`, `AffectObjects`,
  `Power`/`RelativePower`, `CreateHit`, `feel_touch_new`/`_delete`/`_contact`, `CheckForAwaking`,
  `BlowoutState`/`UpdateBlowout`), `src/xrGame/CustomZone.h` (`SZoneObjectInfo`, `m_ObjectInfoMap`,
  the `eZoneState*` enum, `m_eHitTypeBlowout`)
- Postprocess effector: `src/xrGame/zone_effector.cpp` (`CZoneEffector`)
- Subclass effects: `src/xrGame/RadioactiveZone.cpp`, `GraviZone.cpp` (`AffectPull`/`AffectThrow`),
  `MosquitoBald.cpp` (`UpdateSecondaryHit`) + `TorridZone.cpp`, `AmebaZone.cpp`/`ZoneVisual.h`,
  `HairsZone.cpp`, `NoGravityZone.cpp` (`enter_Zone`/`exit_Zone`), `ZoneCampfire.cpp`
- Server entity: `CSE_ALifeAnomalousZone` (`engine/server-entities.md`); detector-side zone draw:
  `flows/detector-artefact.md`

## Maintenance

- Re-grep `CCustomZone::UpdateWorkload` if the state machine changes — it is the single dispatch for
  the idle→awaking→blowout→accumulate cycle.
- Keep `Power = m_fMaxPower × RelativePower` and "hit is `m_eHitTypeBlowout`" accurate — they tie the
  zone to `flows/damage-hit.md` and `flows/outfit-helmet.md`.
- Keep the "zones don't spawn artefacts (this path)" correction — it is a frequently-wrong assumption.
- Promote the *(agent-traced)* per-subclass `Affect`, `UpdateBlowout`, and `CreateHit` bodies to
  verified when a zone-tuning task needs exact impulse/timing math.
