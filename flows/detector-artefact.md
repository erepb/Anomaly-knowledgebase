---
domain: flows/detector-artefact
tags: [xrgame, detector, artefact, feel-touch, belt, restore, activation, flow]
sources: [src/xrGame/CustomDetector.cpp, src/xrGame/CustomDetector.h, src/xrGame/CustomDevice.cpp, src/xrGame/SimpleDetector.cpp, src/xrGame/AdvancedDetector.cpp, src/xrGame/EliteDetector.cpp, src/xrGame/Artefact.cpp, src/xrGame/Artefact.h, src/xrGame/Actor.cpp, src/xrServerEntities/inventory_space.h]
related: [flows/inventory.md, flows/actor.md, flows/outfit-helmet.md, flows/weapon-state-machine.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: detectors & artefacts

How an artefact detector finds nearby artefacts (and zones) and drives its display, and how an
artefact behaves as a belt item — passive restore/immunities while worn, plus its optional
spawn-from-anomaly activation. The two halves meet at the **Feel::Touch** proximity system: artefacts
are touch-feelable objects; a detector subscribes to them within a radius, gated by rank. Belt restore
shares the same actor condition tick as outfits (`flows/outfit-helmet.md`, `flows/actor.md`).

**Provenance:** spine personally code-verified — the detector hierarchy (headers),
`CCustomDetector::shedule_Update`/`UpdateWork`, `CAfList::feel_touch_contact` (rank+section gate),
`CAfList : CDetectList<CArtefact>` + `m_af_rank`, the `DETECTOR_SLOT`/`ARTEFACT_SLOT` constants, the
three subclass `UpdateAf` overrides (existence), the `CArtefact` member model (restore speeds,
`m_ArtefactHitImmunities`, `m_af_rank`, `m_bCanSpawnZone`, activation/detector-support objects),
`CArtefact::ActivateArtefact`/`Action` + the `GEG_PLAYER_ACTIVATEARTEFACT` event, and the artefact
belt-restore application in `CActor` (same tick read for `flows/outfit-helmet.md`). The
`CCustomDevice` battery/on-off bodies, the per-subclass `UpdateAf` *bodies*, `SArtefactActivation` and
`SArtefactDetectorsSupport` internals, and leaf-artefact overrides are confirmed homes but
*(agent-traced — not personally re-read)*.

## Type tree (verified from headers)

- **`CCustomDetector`** ← `CCustomDevice` (a HUD device base — show/hide animation + battery). Children:
  - **`CSimpleDetector`** — single-artefact beep + LED flash.
  - **`CAdvancedDetector`** — beep + a needle/azimuth indicator pointing at the nearest artefact.
  - **`CEliteDetector`** — a 2D map overlay showing all artefacts in range.
    - **`CScientificDetector`** ← `CEliteDetector` — same, **plus detects anomaly zones**.
- **`CArtefact`** ← `CHudItemObject` (+ `CPHUpdateObject` physics). Leaf types — `CGraviArtefact`
  (levitation), `CBastArtefact`/`CBlackGraviArtefact` (strike-back on hit, both add `Feel::Touch`),
  `CGalantine`/`CRustyHair`/`CThorn`/`CZuda`/`CDummy`Artefact — override `Load`/`UpdateCLChild`
  (and sometimes `Hit`/`net_Spawn`) for their physical gimmick; the *belt-effect* model below is shared
  on `CArtefact`. *(leaf bodies agent-traced/SKIMMED.)*

Slots (`inventory_space.h`): detectors go in `DETECTOR_SLOT`; worn artefacts in `ARTEFACT_SLOT` (belt).

## Detector → artefact → UI sequence (spine verified; UI bodies agent-traced)

1. **`CCustomDetector::shedule_Update`** — when `IsWorking()` (equipped, shown, the local view entity),
   `m_artefacts.feel_touch_update(H_Parent()->Position(), m_fAfDetectRadius)`. `m_artefacts` is a
   `CAfList : CDetectList<CArtefact>` — a Feel::Touch subscriber list. Loaded via
   `m_artefacts.load(section, "af")` (the `af_*` config entries = detectable types + sound/freq).
2. **`CAfList::feel_touch_contact(O)`** (verified) — the filter deciding which touched objects count:
   include iff `O->cNameSect()` is in the detector's `m_TypesMap` **and** the artefact's
   `GetAfRank() <= m_af_rank`. So a low-rank detector physically cannot see high-rank artefacts, and a
   detector only sees the artefact sections it was configured for.
3. **`CCustomDetector::UpdateWork`** → **`UpdateAf()`** (virtual; the per-tier UI driver). The three
   overrides all exist; their bodies *(agent-traced)*:
   - **Simple:** nearest artefact → distance → beep period (closer = faster) via `HUD_SOUND_ITEM` +
     `ui().Flash()` (LED bone + light).
   - **Advanced:** nearest + azimuth (`angle_difference_signed` vs camera dir) → rotates a UI needle
     bone (inertia-smoothed).
   - **Elite:** clears + `RegisterItemToDraw(pAf->Position(), …)` for every in-range artefact → a 2D
     map on the detector screen. **Scientific** additionally runs a parallel `m_zones` list to draw
     anomaly zones.
4. The detector is a `CCustomDevice` HUD item: show/hide rides the same OnStateSwitch/animation pattern
   as weapons (`flows/weapon-state-machine.md`); `TurnDeviceInternal(true/false)` flips `m_bWorking`
   and the PDA-display shader factor; `UpdateCL` auto-turns-off below `m_fLowestBatteryCharge` if the
   device uses condition (battery). *(device bodies agent-traced.)*

## Artefact as a belt item (verified members; restore application verified)

- **Registration for detectors** is implicit: an artefact is a Feel::Touch-able world object, so any
  detector in range picks it up via `feel_touch_contact` — there is no explicit "register with
  detector" call. `net_Spawn` optionally builds `m_detectorObj` (`SArtefactDetectorsSupport`) for the
  invisible/patrolling-artefact behavior (show/hide particles, path movement) *(agent-traced)*.
- **Passive restore (worn on belt):** `CArtefact` holds `m_fHealthRestoreSpeed`,
  `m_fRadiationRestoreSpeed`, `m_fSatietyRestoreSpeed`, `m_fPowerRestoreSpeed`,
  `m_fBleedingRestoreSpeed`. These are applied **in the same actor condition tick** that handles
  outfit/helmet restore (`CActor`, `UpdateInventoryOwner` area, `flows/outfit-helmet.md`): for each
  belt artefact, `conditions().ChangeHealth(artefact->m_fHealthRestoreSpeed * artefact->GetCondition()
  * f_update_time)` (and the radiation/power/satiety/bleeding siblings) — **scaled by the artefact's
  own condition**. So artefact regen/drain stacks additively with outfit + helmet.
- **Immunities:** `m_ArtefactHitImmunities` (`CHitImmunity`, `GetImmunity(hit_type)`) — the artefact's
  hit-type resistance contribution.
- **Rank:** `m_af_rank` / `GetAfRank()` — gates detector visibility (above) and tiers artefacts.

## Artefact activation (spawn a zone) (entry verified; state machine agent-traced)

Artefacts with `m_bCanSpawnZone` (`CanBeActivated()`) can be *activated* to spawn an anomaly zone:
- **`CArtefact::Action(cmd, flags)`** intercepts the fire command (`kWPN_FIRE`): start → switch to the
  activating state; stop → back to idle. (verified entry; mirrors a HUD-item input handler.)
- **`CArtefact::ActivateArtefact`** — creates `m_activationObj` (`SArtefactActivation`) and starts its
  sequence (eStarting → eFlying → eBeforeSpawn → eSpawnZone, each a configured set of lights/particles/
  sounds/physics). On completion the artefact emits **`GEG_PLAYER_ACTIVATEARTEFACT`**
  (`u_EventGen(..., H_Parent()->ID())`) — the server-side handler spawns the zone. *(state-machine
  body + server handler agent-traced.)* This is the inverse of the zone→artefact-spawn path that the
  anomaly-zone flow (planned F27) will cover.

## Isolation ledger (the seams)

- **Inventory:** detector in `DETECTOR_SLOT`, artefact in `ARTEFACT_SLOT` (belt); equip/show is the
  HUD-item/inventory machinery (`flows/inventory.md`, `flows/weapon-state-machine.md`).
- **Actor condition:** artefact restore speeds (× artefact condition) summed into the same per-frame
  `conditions().Change*` tick as outfits (`flows/actor.md`, `flows/outfit-helmet.md`); immunities via
  `m_ArtefactHitImmunities`.
- **Feel::Touch:** the detector↔artefact coupling — `CDetectList<CArtefact>::feel_touch_update` +
  `feel_touch_contact` (rank/section gate). `CScientificDetector` runs a second list over zones.
- **HUD/UI:** `g_player_hud->attach_item`; per-tier UI classes (`CUIArtefactDetectorSimple`/`…Adv`/
  `…Elite`) consume `ui().Flash`/`SetValue`/`RegisterItemToDraw`; PDA-display shader factor.
- **Physics/particles/lights:** artefacts are `CPHUpdateObject`s; carried-particle velocity sync,
  trail lights, show/hide particles for invisible artefacts.
- **Network:** `GEG_PLAYER_ACTIVATEARTEFACT` (activation), standard `net_Spawn`/`net_Export`.
- **Lua callbacks/surface** *(agent-traced)*: `CArtefact` exposes `ActivateArtefact`, `CanBeActivated`,
  `AdditionalInventoryWeight`, `GetAfRank`, `FollowByPath`, `SwitchVisibility`, and read/write of all
  five restore speeds + immunities to script. No detector-specific Lua callback found.
- **Config read:** detector section — `af_radius` (→`m_fAfDetectRadius`), `af_vis_radius`,
  `af_class_*`/`af_freq_*`/`af_sound_*` (the `af`-prefixed list loaded by `m_artefacts.load`), battery
  via `CCustomDevice`. Artefact section — `*_restore_speed`, `hit_absorbation_sect`, `af_rank`,
  `additional_inventory_weight`, `particles`/`lights_*`/`trail_light_*`, `can_be_controlled`,
  `det_show/hide_particles`/`_snd`, `particles_bone`, activation-zone params.

## Threading

Main thread: `shedule_Update` (deferred scheduler channel), `UpdateAf`/UI, condition tick, and physics
update all run on the main thread. Feel::Touch contact tests run during the object's update.

## Failure & edge cases

- **Rank gate is silent:** a detector shows nothing for an artefact whose `af_rank > m_af_rank` — not a
  bug, the tier system. Likewise an artefact section missing from the detector's `af_*` list is
  invisible to it.
- **`IsWorking()` requires the local view entity:** detectors only feel-update when held by the current
  player view; an NPC's detector does nothing.
- **Battery cutoff:** a condition-using device auto-turns-off below `m_fLowestBatteryCharge`; a dead
  battery stops detection even while equipped.
- **Restore scales with artefact condition:** a damaged artefact gives proportionally less regen / less
  drain (same `× GetCondition()` rule as armor).
- **Activation needs a parent + `can_spawn_zone`:** `ActivateArtefact` no-ops unless the artefact is
  carried and `m_bCanSpawnZone` is set.

## Deeper

- Detector spine: `src/xrGame/CustomDetector.cpp` (`shedule_Update`, `UpdateWork`,
  `CAfList::feel_touch_contact`), `src/xrGame/CustomDetector.h` (`CAfList`/`CDetectList`, `m_af_rank`,
  `m_fAfDetectRadius`)
- Device base (on/off/battery/anim): `src/xrGame/CustomDevice.cpp` (`TurnDeviceInternal`,
  `OnStateSwitch`, `UpdateCL`)
- Per-tier UI: `src/xrGame/SimpleDetector.cpp`, `AdvancedDetector.cpp`, `EliteDetector.cpp`
  (`UpdateAf`), `src/xrGame/ui/ArtefactDetectorUI.h`
- Artefact: `src/xrGame/Artefact.cpp` (`Load`, `net_Spawn`, `UpdateWorkload`, `ActivateArtefact`,
  `Action`, `OnStateSwitch`), `src/xrGame/Artefact.h` (restore members, `m_ArtefactHitImmunities`,
  `m_af_rank`, `SArtefactActivation`, `SArtefactDetectorsSupport`)
- Belt-restore application: `src/xrGame/Actor.cpp` (`UpdateInventoryOwner` area) — `flows/outfit-helmet.md`
- Slots: `src/xrServerEntities/inventory_space.h` (`DETECTOR_SLOT`, `ARTEFACT_SLOT`)
- Leaf gimmicks: `src/xrGame/{Gravi,Bast,BlackGravi,Thorn,Zuda,Galantine,RustyHair}Artifact.*`

## Maintenance

- Re-grep `CAfList::feel_touch_contact` if the detection gate changes — the rank+section filter is the
  whole detector↔artefact contract.
- Keep the "artefact restore is applied in the actor tick, not in the artefact" fact accurate (shared
  with `flows/outfit-helmet.md`) — re-grep the belt loop in `CActor` if condition restore moves.
- Promote the *(agent-traced)* items (`CCustomDevice` battery/anim bodies, per-tier `UpdateAf` math,
  `SArtefactActivation`/`SArtefactDetectorsSupport` internals, leaf gimmicks) to verified when a
  detector-tuning or artefact-activation task needs them. The zone-spawn server handler pairs with the
  planned anomaly-zone flow (F27).
