---
domain: flows/animation-callbacks
tags: [xrgame, animation, skeleton, motion-mark, footstep, callback, threading, flow]
sources: [src/Layers/xrRender/SkeletonAnimated.cpp, src/xrEngine/SkeletonMotions.cpp, src/xrEngine/SkeletonMotions.h, src/xrGame/HudItem.cpp, src/xrGame/player_hud.cpp, src/xrGame/player_hud.h, src/xrGame/step_manager.cpp]
related: [flows/weapon-state-machine.md, flows/engine-boot-frame.md, flows/lua-callback-lifecycle.md, flows/options-runtime.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: animation / motion callbacks

How skeletal animation playback drives gameplay events. Three callback channels ride the animation
timeline: **motion marks** (named time intervals embedded in the animation that fire mid-playback,
e.g. the reload insert point), **animation end** (drives state machines), and **footsteps** (leg-step
timings that fire material-aware callbacks). The HUD-item state machine in
`flows/weapon-state-machine.md` is the main consumer of end/mark; this flow is the clock that drives it.

**Provenance:** spine personally code-verified — `CHudItem::UpdateCL` (the mark/end poll),
`CHudItem::OnMotionMark`, `CHudItem::OnAnimationEnd`, `CStepManager::update`/`on_animation_start`,
`motion_marks::pick_mark`, and the `CKinematicsAnimated::LL_PlayCycle`/`LL_UpdateTracks` homes. The
blend-pool internals (`CBlend::update`/`IBlendSetup`) and `anim_script_callback` are confirmed homes
but their bodies are *(agent-traced — not personally re-read)*.

## Playback + per-frame advance (verified homes; blend internals agent-traced)

- **`CKinematicsAnimated::LL_PlayCycle`** (`SkeletonAnimated.cpp`) — starts an animation on a bone
  partition: allocate a `CBlend` from the pool, set it up (motion def, accrue/falloff, speed, and an
  optional `PlayCallback`), add it to every bone in the partition, append to `blend_cycles[part]`.
- **`CKinematicsAnimated::LL_UpdateTracks`** — per-frame, advances every blend's time (`CBlend::update`)
  and destroys finished cycles; the optional `PlayCallback` fires when a blend with
  `stop_at_end_callback` reaches its end. *(blend update body agent-traced.)* Bone calc runs on the
  worker thread (`CalculateBonesThread`, `flows/engine-boot-frame.md`).

## HUD-item mark/end driver (verified)

**`CHudItem::UpdateCL`** is the gameplay-facing clock for HUD weapons/items. When a stop-at-end
animation is running (`m_bStopAtEndAnimIsRunning` + `m_current_motion_def`):
- compute `motion_prev_time` (last frame) and `motion_curr_time` (now), both seconds since
  `m_dwMotionStartTm`.
- for each `motion_marks` `M` in `m_current_motion_def->marks`: `M.pick_mark(prev)` and
  `M.pick_mark(curr)`; if the mark was **not** in the previous interval but **is** now
  (`Iprev==NULL && Icurr!=NULL`) → **`OnMotionMark(state, M)`** (the mark was crossed this frame).
- set `m_dwMotionCurrTm = Device.dwTimeGlobal`; if it passed `m_dwMotionEndTm`, clear the motion state
  and fire **`OnAnimationEnd(state)`**.

**`motion_marks::pick_mark(t)`** (`SkeletonMotions.cpp`) — returns the interval `[first,second]`
containing `t` (intervals are ordered; breaks early once `first > t`), else `NULL`.

**Mark timing vs animation speed** (repo addition, cvar `scale_hud_motion_marks_by_speed`, default
**off**). Mark times are authored in the animation's own time base, but the wall-clock elapsed time
above ignores playback speed, so a sped-up or slowed animation crosses its marks at the wrong moment.
With the cvar on, both `motion_prev_time` and `motion_curr_time` are multiplied by the motion's
quantized `CMotionDef` speed **and** by `attachable_hud_item::final_anim_speed` — the actual speed
`anim_play` ended up using after the LTX `anim_speed` and any script override from
`actor_on_hud_animation_play`. Off by default because reload/fire logic hangs off these marks and every
existing config is tuned against the unscaled behaviour.

**`CHudItem::OnMotionMark`** — fires the Lua functor `_G.CHudItem__OnMotionMark(state, mark_name,
obj, parent)`. (Weapon subclasses override to drive e.g. `ReloadMagazine` on the reload mark.)

**`CHudItem::OnAnimationEnd`** — if the parent is the actor, fire the actor callback
`eActorHudAnimationEnd(obj, hud_sect, motion_name, state, slot)`; base handles `eBore→SwitchState(eIdle)`.
Weapon subclasses override to route states (`flows/weapon-state-machine.md`).

## Footsteps (verified)

- **`CStepManager::on_animation_start(motion_id, blend)`** — when a locomotion animation begins, look
  up `motion_id` in `m_steps_map` (loaded from a `step_params` `.ltx` section) to get per-leg
  `SStepParam` timings; record `m_blend` and `m_time_anim_started`.
- **`CStepManager::update(b_hud_view)`** — per frame, for each leg compute
  `offset_time = m_time_anim_started + (cycle timing × step[i].time)`; when reached and not yet handled
  this cycle, pick the ground material pair and fire the footstep:
  - **actor** (object `ID()==0`): Lua functor `_G.CActor__FootstepCallback(material_name, power,
    hud_view)` — if it returns true, play the step sound; **other objects**:
    `object->FootStepCallback(power, play, on_ground, hud_view)` (C++) + step sound.
  - spawn collision particles for the material; mark the leg handled. Only within ~50 m and on ground.

## Isolation ledger (the seams)

- **Lua callbacks fired:** `_G.CHudItem__OnMotionMark` (mark), actor `eActorHudAnimationEnd` (end),
  `_G.CActor__FootstepCallback` (actor footstep), plus `_G.CHudItem__OnStateSwitch` from the state
  machine (`flows/weapon-state-machine.md`). All resolved via `ai().script_engine().functor`
  (`flows/lua-callback-lifecycle.md`); a missing functor is a silent no-op (mark) — they are optional.
- **Config read:** animation defs + embedded motion marks from `.omf`; footstep leg timings from a
  `step_params` `.ltx` section; HUD animation names/durations from item `.ltx` (`pSettings`).
- **Per-frame integration:** `UpdateCL` (HUD items) and `CStepManager::update` run on the **main
  thread**; `LL_UpdateTracks`/bone calc run on the **worker thread** (`CalculateBonesThread`,
  `flows/engine-boot-frame.md`). The gameplay callbacks all fire from the main-thread side, not the
  worker — only the optional `CBlend` `PlayCallback` (script animations) fires from the track update.
- **Other flows:** end/mark drive the weapon state machine (`flows/weapon-state-machine.md`); the IK
  leg controller is notified at animation start; footsteps cross into the material/sound/particle
  systems.

## Threading

Bone/track advance is on the worker thread; the gameplay-facing mark/end/footstep callbacks fire from
main-thread `UpdateCL`/`CStepManager::update`. Blend lists use a reader-writer lock; track update is
guarded against double-running per frame.

## Failure & edge cases

- **"Stuck" animation:** if a HUD animation clip is missing/zero-length, `m_dwMotionEndTm` is never
  passed and `OnAnimationEnd` never fires — the item hangs in that state (a common modding symptom).
- **Mark fires once per crossing:** the `prev==NULL && curr!=NULL` edge test means a mark fires on the
  frame it's entered; a frame stall that skips the interval entirely can miss it.
- **Footstep timing is animation-relative:** offsets are stored relative to the cycle and converted at
  play start, so changing animation speed shifts steps consistently; missing `step_params` disables
  steps for that motion.
- **Optional Lua:** mark/footstep callbacks check functor existence first; absence is non-fatal.

## Deeper

- Skeleton playback: `src/Layers/xrRender/SkeletonAnimated.cpp` (`LL_PlayCycle`, `LL_UpdateTracks`),
  `src/xrEngine/SkeletonMotions.cpp`/`.h` (`motion_marks::pick_mark`, `CMotionDef::marks`, `CBlend`)
- HUD driver: `src/xrGame/HudItem.cpp` (`UpdateCL`, `OnMotionMark`, `OnAnimationEnd`, `OnStateSwitch`,
  `PlayHUDMotion`)
- Footsteps: `src/xrGame/step_manager.cpp` (`on_animation_start`, `update`), `step_manager_defs.h`
  (`SStepParam`)
- Worker bone calc: `src/xrEngine/EngineThreading.cpp` (`CalculateBonesThread`) — see
  `flows/engine-boot-frame.md`
- Consumers: `flows/weapon-state-machine.md`

## Maintenance

- Re-grep `CHudItem::UpdateCL` if the mark-crossing test or end detection changes — it is the single
  driver for both HUD callbacks.
- Keep the main-thread (gameplay callbacks) vs worker-thread (track/bone advance) split accurate
  against `flows/engine-boot-frame.md`.
- Promote the *(agent-traced)* `CBlend::update`/`IBlendSetup`/`anim_script_callback` internals to
  verified if a task needs exact blend math or the script-animation callback path.
