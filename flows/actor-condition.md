---
domain: flows/actor-condition
tags: [xrgame, actor, condition, survival, satiety, radiation, bleeding, booster, stamina, flow]
sources: [src/xrGame/ActorCondition.cpp, src/xrGame/ActorCondition.h, src/xrGame/EntityCondition.cpp, src/xrGame/EntityCondition.h, src/xrGame/eatable_item.cpp]
related: [flows/actor.md, flows/damage-hit.md, flows/outfit-helmet.md, flows/detector-artefact.md, flows/actor-survival-status.md, modlayer/gameplay/actor-survival.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: actor condition & survival

The per-tick survival simulation: how the player's health, stamina (power), satiety, radiation,
bleeding, psy-health, alcohol and timed **boosters** evolve each frame. This **deepens**
`flows/actor.md` (which covers actor spawn/control/death broadly) and `flows/damage-hit.md` (hit →
condition); here is the condition *tick* those feed. It is also the consumer of the outfit/helmet and
artefact restore speeds (`flows/outfit-helmet.md`, `flows/detector-artefact.md`) and of eatable-item
boosters.

**Lua-layer caveat (F103):** in the shipped Anomaly corpus a Lua tier sits on top of this tick —
sleep/thirst meters the engine doesn't have, and crucially `arszi_psy.script`, which **zeroes
actor-bound telepatic hits before the engine applies them** and runs its own psy meter; the engine
psy bar is then display-only (set via `SetPsyBar`). The HUD indicator strip and booster icons are
also script-side. See `flows/actor-survival-status.md`.

**Provenance:** spine personally code-verified — `CEntityCondition::UpdateCondition` (the
delta-accumulator apply/clamp/reset + the three critical-health callbacks + last-chance health),
`ChangeHealth`/`ChangePower`/`ChangeRadiation`/`ChangePsyHealth` (the accumulators),
`CEntityCondition::UpdateRadiation`/`BleedingSpeed`, `CActorCondition::UpdateCondition` (top: satiety/
boosters/alcohol/weight-stamina/power-leak) and `UpdateSatiety`. The booster apply path
(`ApplyBooster`/`BoostParameters`/`DisableBoostParameters`/`UpdateBoosters`), the `EBoostParams` enum,
`ConditionWalk/Stand/Jump`, `UpdateHealth`/`UpdatePower` bodies, the `eatable_item` entry, and the
tutorial-threshold Lua callbacks are confirmed homes but *(agent-traced — not personally re-read)*.

## The delta-accumulator pattern (verified) — the core idea

Every condition change goes through an accumulator, applied once per tick. `ChangeHealth(v)` does
`m_fDeltaHealth += (CanBeHarmed() || v>0) ? v : 0` (a god-mode actor ignores damage but still heals);
`ChangePower`/`ChangeRadiation`/`ChangePsyHealth` just `+= v`. Nothing writes the live value directly —
they all stage into `m_fDelta*`. Then `CEntityCondition::UpdateCondition` (once per tick) computes the
`Update*` contributions, **applies all deltas, resets them to 0, and clamps**. This is why
outfit/artefact restore, zone hits, eatable items, and bleeding all compose cleanly: they each call a
`Change*` and the tick reconciles them.

## Base condition tick — `CEntityCondition::UpdateCondition` (verified)

1. bail if `GetHealth() <= 0`.
2. **critical-health probes** at three points — before, after wounds, after radiation — firing
   `m_object->OnCriticalHitHealthLoss()` / `OnCriticalWoundHealthLoss()` / `OnCriticalRadiationHealthLoss()`
   when `m_fDeltaHealth + GetHealth() <= 0`. These are the death-cause callback seams.
3. `UpdateHealth()` (bleeding + natural regen + wound incarnation), `UpdatePower()`, `UpdateRadiation()`,
   `UpdatePsyHealth()`, `UpdateEntityMorale()` — each adds to its `m_fDelta*`. *(UpdateHealth/Power
   bodies agent-traced.)*
4. **last-chance health:** if `curr_health > m_fKillHitTreshold` but `curr_health + m_fDeltaHealth < 0`
   (a single lethal blow from healthy), set `m_fLastChanceHealth` and an invulnerability window instead
   of dying — the "survive one big hit" mechanic. Otherwise apply `m_fDeltaHealth`.
5. apply `m_fDeltaPower/PsyHealth/EntityMorale/Radiation`; **reset all deltas to 0**; clamp health to
   `[MIN_HEALTH, max_health()]`, power `[0, m_fPowerMax]`, radiation `[0, m_fRadiationMax]`, etc.

**Radiation** (`UpdateRadiation`, verified): while `m_fRadiation > 0`, decay
`m_fDeltaRadiation -= V_Radiation·dt` and damage `m_fDeltaHealth -= V_RadiationHealth·m_fRadiation·dt`
(damage scales with the accumulated dose; decays over time). Radiation comes in via `ChangeRadiation`
from zones/items/material contact.

**Bleeding** (`BleedingSpeed`, verified): `Σ wound.TotalSize() · m_fBleedSpeedK`, clamped `[0, 10]`;
`UpdateHealth` turns that into health loss and shrinks wounds over time (incarnation). Wounds come from
the hit pipeline (`flows/damage-hit.md`).

## Actor extension — `CActorCondition::UpdateCondition` (verified)

Before the base tick, the actor adds:
- **`UpdateSatiety()`** + **`UpdateBoosters()`** + alcohol accumulation (run even in some god modes).
- **stamina by movement & weight:** `ConditionWalk(weight/base, accel, sprint)` while moving else
  `ConditionStand` — drains power scaled by carried-weight ratio and movement mode *(bodies
  agent-traced)*.
- **max-power leak:** `SetMaxPower(GetMaxPower() - power_leak_speed·dt·k_max_power)` where `k_max_power`
  grows with carried weight — the longer/heavier you act, the lower your stamina *ceiling* drifts
  (recovered by rest/satiety).
- guards: returns under god-mode/dead/non-local-view.

### Satiety (`UpdateSatiety`, verified) — hunger drives regen

- apply queued `m_fSatietyChange` (from food), clamp `[0,1]`, reset.
- drain: `m_fSatiety -= v_satiety·dt`.
- `satiety_health_koef = (satiety - critical) / (satiety>=critical ? 1-critical : critical)` — **signed**:
  above the critical threshold it's positive → `m_fDeltaHealth += v_satiety_health·koef·dt` (regen);
  below it's negative → health *drain* (starvation). Power always regenerates proportional to satiety:
  `m_fDeltaPower += v_satiety_power·satiety·dt`. (Sleep uses slower `*_sleep` variants.)

So **satiety is the engine of passive health/stamina regen** — full belly heals, empty belly starves.
Outfit/helmet/artefact restore speeds (`flows/outfit-helmet.md`) are *added on top* via the same
`ChangeHealth/Power/...` accumulators in the actor's inventory-owner tick.

## Boosters (agent-traced; entry verified-called)

Consumables apply **timed** boosts. `ApplyBooster(SBooster, section)` registers a booster in
`m_booster_influences` (keeping the stronger if one already exists) and calls `BoostParameters`, which
dispatches on `EBoostParams` (restore rates: HpRestore/PowerRestore/RadiationRestore/BleedingRestore;
MaxWeight; and per-hit-type immunity/protection boosts) to bump the matching member.
`UpdateBoosters()` (called from the actor tick) decrements each booster's timer and on expiry calls
`DisableBoostParameters` (applies the negated value) and updates the HUD booster indicators. Eatable
items enter via `CEatableItem::UseBy`, which builds an `SBooster` per `boost_*` config line and calls
`conditions().ApplyBooster`. *(bodies agent-traced.)*

## Change API (verified entry points)

`ChangeHealth`/`ChangePower`/`ChangeRadiation`/`ChangePsyHealth` (accumulators, verified) +
`CActorCondition::ChangeSatiety` (→ `m_fSatietyChange`) / `ChangeAlcohol`, and `ChangeBleeding` (heals
wounds via incarnation). These are exactly the entry points outfit/artefact restore, zones, food, and
scripts use. Eatable/medkit influence (`SMedicineInfluenceValues`) calls the set of them at once.

## Isolation ledger (the seams)

- **Hit/damage:** `flows/damage-hit.md` lands `SHit` → `ConditionHit` → wounds + `Change*` deltas; the
  three `OnCritical*HealthLoss` callbacks are the death-cause seam into `flows/actor.md` death.
- **Equipment restore:** outfit/helmet/artefact restore speeds add via `Change*` in the actor
  inventory tick (`flows/outfit-helmet.md`, `flows/detector-artefact.md`); outfit/helmet/backpack
  `power_loss` scales stamina drain (`HitPowerEffect`).
- **Zones:** radiation/psy/fire influences from `feel_touch` objects + injurious materials queue
  `SHit`s of the matching type (`flows/anomaly-zones.md`).
- **Consumables:** `eatable_item.cpp` → `ApplyBooster` + medicine influence values → `Change*`.
- **HUD:** booster indicators, condition bars read these members; tutorial-threshold Lua callbacks
  fire on crossing low-power / bleeding / satiety / radiation / psy thresholds *(agent-traced:
  `_G.on_actor_*`)*.
- **Lua surface:** `conditions():GetHealth/SetHealth`, `Change*`, `V_Satiety*`, `ApplyBooster`,
  `BoosterForEach`, `AddWound`, etc. (the `actor:conditions()` API mods use).
- **Config read** (actor/condition section): `satiety_v(_sleep)`, `satiety_power_v`,
  `satiety_health_v`, `satiety_critical`, `radiation_v`, `radiation_health_v`, `bleeding_v`,
  `wound_incarnation_v`, `health_restore_v`, `psy_health_v`, `*_power`/`*_weight_power`/`overweight_*_k`/
  `accel_k`/`sprint_k`, `max_power_leak_speed(_sleep)`, `max_walk_weight`, `*_zone_max_power` caps,
  `[tutorial_conditions_thresholds]`; booster sections (`boost_time`, `boost_*_restore`,
  `boost_max_weight`, `boost_*_immunity/protection`); eatable influence (`eat_*`, `apply_time_sec`).
- **Sleep/peripheral:** `Actor_Sleep.cpp` is effectively empty — sleeping is detected via
  `HasInfo("actor_is_sleeping")` and selects the `*_sleep` rate variants. `ActorNightVision.cpp` /
  `ActorBackpack.cpp` are separate (NV is an effector; backpack contributes only `power_loss`).

## Threading

Main thread (scheduled actor update). Single-threaded with gameplay/Lua.

## Failure & edge cases

- **Starvation:** satiety below `satiety_critical` flips `satiety_health_koef` negative → continuous
  health drain. The single most common "why am I slowly dying" cause.
- **Max-power leak:** stamina *ceiling* drops with sustained heavy activity; a player can't sprint not
  because power hit 0 but because max-power leaked — recovered by satiety/rest.
- **Last-chance health:** a healthy actor survives one lethal blow (invuln window) instead of dying —
  intended, not a bug; repeated within the window still kills.
- **God-mode asymmetry:** `ChangeHealth` ignores negative deltas under god-mode but still applies
  positive — healing works, damage doesn't.
- **Deltas reset every tick:** anything that wants a sustained effect must call `Change*` *each tick*
  (that's how restore speeds work); a one-shot `ChangeHealth` is a single bump.

## Deeper

- Base sim: `src/xrGame/EntityCondition.cpp` (`UpdateCondition`, `ChangeHealth`/`ChangePower`/
  `ChangeRadiation`/`ChangePsyHealth`/`ChangeBleeding`, `UpdateHealth`, `UpdatePower`, `UpdateRadiation`,
  `BleedingSpeed`, `SMedicineInfluenceValues`, `SBooster::Load`), `EntityCondition.h` (`EBoostParams`,
  `SBooster`, `m_fDelta*`, `change_v()`)
- Actor sim: `src/xrGame/ActorCondition.cpp` (`UpdateCondition`, `UpdateSatiety`, `UpdateBoosters`,
  `ApplyBooster`, `BoostParameters`/`DisableBoostParameters`, `ConditionWalk`/`Stand`/`Jump`,
  `ChangeSatiety`, `LoadCondition`), `ActorCondition.h`
- Consumable entry: `src/xrGame/eatable_item.cpp` (`UseBy`)
- Parents/consumers: `flows/actor.md`, `flows/damage-hit.md`, `flows/outfit-helmet.md`,
  `flows/detector-artefact.md`, `flows/anomaly-zones.md`

## Maintenance

- Re-grep `CEntityCondition::UpdateCondition` if the apply/clamp/reset order changes — it is the single
  reconciler; keep the delta-accumulator explanation accurate (it underpins all restore/damage).
- Keep `UpdateSatiety`'s signed `satiety_health_koef` description exact — it is the
  regen-vs-starvation hinge and a frequent tuning target.
- Promote the *(agent-traced)* booster path (`ApplyBooster`/`BoostParameters`), `ConditionWalk/Stand`,
  and `UpdateHealth/Power` bodies to verified when a survival-balance or booster-authoring task needs
  the exact math.
