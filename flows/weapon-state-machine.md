---
domain: flows/weapon-state-machine
tags: [engine, cpp, weapons, hud, animation, flow, state-machine, callbacks]
sources: [src/xrGame/HudItem.h, src/xrGame/HudItem.cpp, src/xrGame/Weapon.h, src/xrGame/Weapon.cpp, src/xrGame/WeaponMagazined.h, src/xrGame/WeaponMagazined.cpp, src/xrGame/WeaponAK74.cpp, src/xrGame/WeaponFire.cpp, gamedata/scripts/callbacks_gameobject.script]
related: [flows/weapon-fire-callbacks.md, flows/object-lifecycle.md, flows/inventory.md, flows/actor.md, flows/stalker-execution.md, flows/animation-callbacks.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: weapon state machine (show / idle / fire / reload / hide)

How a weapon moves between states — hidden → showing → idle → fire / reload / zoom / switch-mode →
hiding — and how **HUD-animation timing drives the transitions**. This is the *when* of weapon
behavior; the *what happens when a shot leaves the barrel* (bullet → trace → hit → callbacks) is the
separate, already-verified `flows/weapon-fire-callbacks.md`. Read that for ballistics; read this for
the state pump.

**Provenance:** researched with a read-only agent, then spot-verified against code. The two state
enums, the `OnAnimationEnd → SwitchState` routing, and the `CHudItem::UpdateCL` animation-end driver
are personally code-verified. The `switch2_*` / `PlayAnim*` / `ReloadMagazine` bodies are
*(agent-traced — not personally re-read)*; the transition table below is the deliverable and is
anchored on the verified routing.

## The two state enums (verified)

A weapon's state is one `u32` in `CHUDState` (base) extended by `CWeapon`:

- **`CHUDState::EHudStates`** (`HudItem.h`): `eIdle = 0, eShowing, eHiding, eHidden, eBore,
  eLastBaseState = eBore`. The base HUD-item states every hud item shares.
- **`CWeapon::EWeaponStates`** (`Weapon.h`): `eFire = eLastBaseState + 1, eFire2, eReload, eMisfire,
  eSwitch, eSwitchMode, eAimStart, eAimEnd`. Weapon-specific states layered on top.

Tri-state reload has its own substate enum `CWeapon::EWeaponSubStates`
(`eSubstateReloadBegin/InProcess/End/InProcessEmptyEnd`, gated by `m_bTriStateReload`) — verified
present in `Weapon.h`.

State accessors (`HudItem.h`): `GetState()`, `GetNextState()`, `SetState()`, `SetNextState()`.

## The transition engine (verified)

Two functions make it a state machine, plus one timer:

1. **`CHudItem::SwitchState(u32 S)`** (`HudItem.cpp`) — *requests* a transition: sets the next state
   and, if the object is `Local()` and not being destroyed, sends a `GE_WPN_STATE_CHANGE` network
   event (so transitions are event-ordered, not inline). The event lands in `CHudItem::OnEvent` →
   **`OnStateSwitch(S, oldState)`**.
2. **`OnStateSwitch`** — base `CHudItem::OnStateSwitch` updates the stored state + fires the Lua
   functor `_G.CHudItem__OnStateSwitch`; `CWeaponMagazined::OnStateSwitch` (`WeaponMagazined.cpp`)
   dispatches into the per-state entry handler `switch2_Idle/Fire/Reload/Showing/Hiding/Hidden/...`,
   which starts the matching HUD animation and (for most) calls `SetPending(TRUE)` to lock input
   until the animation finishes.
3. **The animation-end timer** — when `switch2_*` starts an animation via `PlayHUDMotion`, it sets
   `m_dwMotionEndTm` and `m_bStopAtEndAnimIsRunning = true`. **`CHudItem::UpdateCL`** (verified)
   polls each frame: when `m_dwMotionCurrTm > m_dwMotionEndTm` it clears the flag and calls
   **`OnAnimationEnd(m_startedMotionState)`** — and along the way fires `OnMotionMark` for any motion
   marks reached (the reload mark mechanism).

The loop closes in **`CWeaponMagazined::OnAnimationEnd(u32 state)`** (verified routing in
`WeaponMagazined.cpp`): `eReload → SwitchState(eIdle)` (after `ReloadMagazine`), `eShowing →
SwitchState(eIdle)`, `eHiding → SwitchState(eHidden)`, `eFire → SwitchState(eIdle)`,
`eAimStart/eAimEnd → SwitchState(eIdle)`. **Animation completion is what advances the machine** —
that `UpdateCL` timer → `OnAnimationEnd` → `SwitchState` cycle is the heart of the flow.

## State transition table

`anim_*` names are read from the weapon `.ltx` (via `pSettings`); the `empty`/`aim` variants are
chosen at play time by ammo count / zoom. *(transitions anchored on verified `OnAnimationEnd`
routing + verified enums; the `switch2_*`/`PlayAnim*` choice of clip is agent-traced.)*

| From | Trigger | To | Animation | What advances it |
|------|---------|-----|-----------|------------------|
| eHidden | inventory `OnActiveItem` (item drawn) | eShowing | `anim_show[_empty]` | `OnAnimationEnd(eShowing)` → eIdle |
| eShowing | animation ends | eIdle | — | timer |
| eIdle | `kWPN_FIRE` start (`Action`/`FireStart`) | eFire | `anim_shot[s][_aim][_l]` | trigger held; `state_Fire` loops shots |
| eFire | trigger released / ammo 0 | eIdle | (finish shot) | `OnAnimationEnd(eFire)` / `StopShooting` |
| eIdle | `kWPN_RELOAD` / empty mag | eReload | `anim_reload[_empty][_misfire]` | motion-mark or `OnAnimationEnd(eReload)` |
| eReload | reload mark / animation ends | eIdle | — | `ReloadMagazine` then eIdle |
| eIdle | `kWPN_ZOOM` start | eAimStart | `anim_idle_aim_start` | `OnAnimationEnd` → eIdle (now zoomed) |
| eIdle | `kWPN_ZOOM` end | eAimEnd | `anim_idle_aim_end` | `OnAnimationEnd` → eIdle |
| eIdle | `kWPN_FIREMODE_*` | eSwitchMode | `anim_firemode_switch` | `OnAnimationEnd` → eIdle, `UpdateFireMode` |
| eIdle | inventory `OnHiddenItem` (holstered) | eHiding | `anim_hide[_empty]` | `OnAnimationEnd(eHiding)` → eHidden |
| eHiding | animation ends | eHidden | — (detach HUD) | timer |

## Reload detail (verified homes)

`kWPN_RELOAD` (or an empty magazine) → `CWeaponMagazined::Reload` → `TryReload` (checks ammo
availability across ammo types, picks the next type, `SwitchState(eReload)` or back to `eIdle` if
none) → `switch2_Reload` plays the clip and sets `m_needReload`. The magazine is actually refilled by
**`ReloadMagazine`**, fired either from the **motion mark** (`OnMotionMark`, e.g. an `lmg_reload`
mark mid-animation) or at `OnAnimationEnd(eReload)` — then the machine returns to `eIdle`. A held
fire key during reload sets `m_pendingShot`, replayed on entry to `eIdle` (`switch2_Idle`). The
misfire path uses `anim_reload_misfire` + `bClearJamOnly`. *(bodies agent-traced; the
mark-or-end → `ReloadMagazine` → eIdle structure is the load-bearing part.)*

**NPC reload edges** (repo addition). `CWeapon::OnStateSwitch` fires `_G.CAI_Stalker__OnWeaponReloadStart`
/ `_OnWeaponReloadStop` (bridged to the `npc_on_weapon_reload_start` / `_stop` bus callbacks) whenever
`eReload` is entered or left **and the weapon's parent is a `CAI_Stalker`**. Two deliberate choices:
the hook sits in `CWeapon::OnStateSwitch` rather than `CWeaponMagazined`'s override, because the
tri-state shotgun path skips that override for `eReload`, and the actor is excluded because actor
reloads already surface as `actor_on_weapon_reload`. The stop edge is **best-effort**: a weapon detached
mid-reload (owner died, weapon dropped) has no stalker parent and fires nothing — `wpn:get_state() ==
7` (`eReload`) stays the authoritative read.

## Fire detail (state side only)

`kWPN_FIRE` start → `CWeapon::FireStart`/`switch2_Fire` sets the shot counters; the actual cadence
runs in **`CWeaponMagazined::state_Fire(dt)`** called from `UpdateCL` while in `eFire`: a per-shot
timer (`fShotTimeCounter`, derived from RPM config) gates each `OnShot` + `FireTrace`. Trigger
release / empty mag → `StopShooting`/`OnMagazineEmpty` → `eIdle`. The **bullet** side of `OnShot`/
`FireTrace`/`FireBullet` and the `eOnWeaponFired`/`eOnWeaponJammed`/`eOnWeaponMagazineEmpty`
callbacks are documented in `flows/weapon-fire-callbacks.md`.

## Isolation ledger (the seams)

- **Callbacks fired:** Lua functors `_G.CHudItem__PlayHUDMotion` (can return `$cancel` to abort an
  animation), `_G.CHudItem__OnStateSwitch`, `_G.CHudItem__OnMotionMark`,
  `_G.CAI_Stalker__OnWeaponReloadStart`/`Stop` (NPC-only, above); per-object weapon callbacks
  `eOnWeaponZoomIn`/`Out`, `eWeaponNoAmmoAvailable`, and (in `flows/weapon-fire-callbacks.md`)
  `eOnWeaponFired`/`eOnWeaponJammed`/`eOnWeaponMagazineEmpty`; and `eActorHudAnimationEnd` fired by
  `CHudItem::OnAnimationEnd` when the parent is the actor. *(functor names agent-traced; the
  `OnAnimationEnd`/`UpdateCL` driver is verified.)*
- **Lua surface (repo additions):** `obj:hud_inertion_enabled()` / `set_hud_inertion_enabled(bool)`
  toggle the `CHudItem` `fl_inertion_enable` flag (HUD sway/inertia) per item at runtime;
  `wpn:get_CurrentFirePoint()` / `_2()` / `_Silencer()` expose the live fire points; the motion-mark
  timing cvar is `scale_hud_motion_marks_by_speed` (`flows/animation-callbacks.md`).
- **Config read (weapon `.ltx`):** the `anim_*` clip names (show/hide/reload/shot/idle, with
  `_empty`/`_aim`/`_l` variants), fire rate (`rpm`/`rpm_mode_2` → shot timers), `fire_modes`,
  `magazine_size`, `ammo_type` list, misfire condition/probability params, addon names. *(key names
  agent-traced; mechanism = `pSettings`.)*
- **Scheduler seam:** `UpdateCL` (real-time channel of the backbone) is the pump — it runs
  `state_Fire(dt)` and the `m_dwMotionEndTm` poll that fires `OnAnimationEnd`. Transitions ride
  `GE_WPN_STATE_CHANGE` events.
- **HUD / animation seam (the core):** `PlayHUDMotion` starts a clip + arms the end timer;
  `UpdateCL` fires `OnMotionMark` (mid-clip events like reload-insert) and `OnAnimationEnd`
  (clip done → next state). First-person uses `g_player_hud`; `OnAnimationEnd` notifies the actor via
  `eActorHudAnimationEnd`.
- **Inventory / draw seam:** `OnActiveItem` (drawn → `eShowing`) and `OnHiddenItem` (holstered →
  `eHiding`/`eHidden`) come from the inventory active-slot machine (`flows/inventory.md`);
  `IsPending()` blocks input while an animation runs.
- **Other domains:** actor (sprint stop, HUD attach, the `eActorHudAnimationEnd` callback),
  `CShootingObject` base (ballistics — `flows/weapon-fire-callbacks.md`), sound/particles/effects
  for shot feedback.

## Threading

All state-machine work runs on the main thread via `UpdateCL` (serial scheduler). Only the bullet
manager (separate flow) can offload.

## Failure & edge cases

- **Transitions are event-driven** (`SwitchState` → `GE_WPN_STATE_CHANGE` → `OnStateSwitch`), not
  immediate — the new state's `switch2_*` runs when the event is processed, so don't assume the state
  changed the instant `SwitchState` returned.
- **Animations are what advance most states.** A missing/short `.ltx` clip means `OnAnimationEnd`
  fires early/never as expected — a frequent source of "weapon stuck in reload/showing" bugs.
- **`_G.CHudItem__PlayHUDMotion` returning `$cancel`** suppresses an animation entirely, which can
  strand the state machine if a mod does it on a state that relies on the end timer.
- **A held fire key during reload** is not lost — `m_pendingShot` replays it on entry to `eIdle`.

## Deeper

- Base HUD state + driver: `src/xrGame/HudItem.h` (`CHUDState`, `EHudStates`),
  `src/xrGame/HudItem.cpp` (`SwitchState`, `OnStateSwitch`, `UpdateCL`, `OnAnimationEnd`,
  `OnMotionMark`, `PlayHUDMotion`)
- Weapon states + handlers: `src/xrGame/Weapon.h` (`EWeaponStates`, `EWeaponSubStates`),
  `src/xrGame/WeaponMagazined.cpp` (`OnStateSwitch`, `switch2_*`, `OnAnimationEnd`, `state_Fire`,
  `Reload`/`TryReload`/`ReloadMagazine`, `PlayAnim*`)
- Fire start/stop (state side): `src/xrGame/Weapon.cpp` (`Action`, `FireStart`, `OnActiveItem`,
  `OnHiddenItem`), `src/xrGame/WeaponFire.cpp` (`FireEnd`, `StopShooting`)
- Ballistics (separate): `flows/weapon-fire-callbacks.md`

## Maintenance

- Re-grep the two enums in `HudItem.h`/`Weapon.h` and the `OnAnimationEnd` switch in
  `WeaponMagazined.cpp` if states are added/reordered — the transition table is anchored on them.
- Promote the *(agent-traced)* `switch2_*`/`PlayAnim*`/`ReloadMagazine`/`TryReload` bodies and the
  `.ltx` key list to verified by reading them when a task needs the per-clip detail.
- Keep the "animation completion advances the machine" framing — it's the non-obvious coupling that
  makes weapon timing data-driven from `.ltx`.
