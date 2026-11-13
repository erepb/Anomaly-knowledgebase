---
domain: flows/camera-pp-effectors
tags: [xrgame, camera, effector, postprocess, shake, bobbing, zone, flow]
sources: [src/xrGame/ActorEffector.cpp, src/xrGame/ActorEffector.h, src/xrGame/CameraEffector.h, src/xrGame/EffectorFall.cpp, src/xrGame/EffectorBobbing.cpp, src/xrGame/EffectorZoomInertion.cpp, src/xrGame/EffectorShot.cpp, src/xrGame/pp_effector_custom.cpp, src/xrGame/pp_effector_distance.cpp, src/xrGame/PostprocessAnimator.cpp, src/xrGame/script_effector.cpp, src/xrGame/level_script.cpp]
related: [flows/actor.md, flows/render-frame.md, flows/anomaly-zones.md, flows/weapon-state-machine.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: camera & post-process effectors

The two transient-effect stacks layered onto the actor's view each frame: **camera effectors** (shake
the camera position/rotation/FOV — landing bounce, head bob, weapon recoil, scope sway, scripted
cutscene moves) and **post-process effectors** (overlay screen color/blur/noise/duality — hit flash,
anomaly tint, scripted `.ppe` effects). They feed the actor camera (`flows/actor.md`) and the
renderer's post chain (`flows/render-frame.md`); zones and hits push effectors onto them.

**Provenance:** spine personally code-verified — `CEffectorFall::ProcessCam(SCamEffectorInfo&)` (the
camera-effector `ProcessCam` signature/lifetime pattern) and the `level.add_cam_effector`(×4 overloads)/
`add_pp_effector`/`remove_pp_effector` Lua bindings. The two manager stacks live in
`xrEngine/CameraManager` (`m_EffectorsCam` list + `m_EffectorsPP` vector, `SPPInfo` composite) and the
per-effector bodies (`CEffectorBobbing`/`CEffectorZoomInertion`/`CCameraShotEffector`,
`CPostprocessAnimator` `.ppe`, `CPPEffectorDistance`, `CZoneEffector`) are confirmed homes but
*(agent-traced — not personally re-read; the engine manager is outside xrGame)*.

## Two stacks (agent-traced manager; per-frame model)

`CActorCameraManager` (← `xrEngine` `CCameraManager`) owns both stacks and is updated each frame:
- **Camera stack** `m_EffectorsCam` (an ordered list; adds are deferred a frame to avoid iterator
  invalidation). Each frame every effector's `ProcessCam(SCamEffectorInfo&)` runs **in order**, each
  mutating the running camera info (position `p`, direction `d`, normal `n`, right `r`, `fFov`,
  `fAspect`) in place — so effectors **compose by sequential mutation**. Expired ones (`fLifeTime<=0`,
  `Valid()==false`) are removed.
- **Post-process stack** `m_EffectorsPP` (vector). Each frame every effector's `Process(SPPInfo&)`
  fills an `SPPInfo`, composited into `pp_affected` via `SPPInfo::add` (**additive** — colors/blur/
  duality summed, noise max'd — unless an effector sets `bOverlap` to replace lower ones). The result
  is written to the render target (`flows/render-frame.md`).

## Camera effectors (signature verified; bodies agent-traced)

`CEffectorCam::ProcessCam(SCamEffectorInfo&)` is the per-frame hook (verified via `CEffectorFall`):
decrement `fLifeTime`, mutate the camera info, return `Valid()`. Type-tagged by `ECamEffectorType`
(`CameraEffector.h`: `eCEFall`/`eCEShot`/`eCEBobbing`/`eCEZoom`/`eCERecoil`/`eCEHit`/`eCEControllerPsyHit`/
`eCEDOF`/…) — adding a new effector of a type first removes the existing one of that type. Verified
example `CEffectorFall::ProcessCam` advances a sine phase and offsets `info.p.y` (the landing bounce).
Others *(agent-traced)*: **Bobbing** (permanent; walk/run/limp amplitude from `[bobbing_effector]`,
`SetState` each frame from the actor), **ZoomInertion** (permanent; scope sway lerping toward a random
point scaled by `[zoom_inertion_effector]` + weapon dispersion), **CCameraShotEffector** (`eCEShot`;
weapon recoil angles, `flows/weapon-state-machine.md`), **CAnimatorCamEffector** (plays a `.anm`
camera animation — scripted moves).

## Post-process effectors (composite + Lua verified; bodies agent-traced)

`CEffectorPP::Process(SPPInfo&)` fills the post params (blur, gray, duality h/v, noise, color base/add/
gray, color-map). Variants *(agent-traced)*: **`CPostprocessAnimator`** plays a `.ppe` envelope file
(11 params; `Lerp`/`Const`/`Controlled` variants scale by a factor func), **`CPPEffectorCustom`** lerps
a fixed `SPPInfo` by a factor (hit flash / direct overlay), **`CPPEffectorDistance`** drives its factor
by distance to a source (zone glow fading with range). `bOverlap` lets one effector mask the rest;
`bFreeOnRemove` auto-deletes on expiry; `Stop(speed)` fades out.

## Script & integration seams (Lua verified)

- **Lua surface** (`level_script.cpp`, verified): `level.add_cam_effector(fn, id, cyclic, cb_func [,
  cam_fov [, b_hud [, power]]])` (4 overloads — loads a `.anm`, returns its length, optional completion
  callback, optional HUD-affect/power), `level.add_pp_effector(fn, id, cyclic)` (loads a `.ppe`),
  `level.remove_pp_effector(id)`, plus `set_pp_effector_factor`/`_factor2`. Custom ids are allocated
  from a high base (`effCustomEffectorStartID`). This is the main mod entry for screen FX/cutscene
  camera. `CScriptEffector` is the Lua-controlled PP effector wrapper.
- **Added 2026.09–2026.11:** a 5th `add_cam_effector` overload with a trailing `speed` (anim playback
  speed, returns the length); PP queries `level.get_pp_effector_factor(id)`, `get_pp_effector_length(id)`,
  `check_pp_effector(id)` (`PostprocessAnimator`); `device():set_cam_attached_fov` /
  `get_cam_attached_fov` (`script_render_device_script.cpp`) — projection FOV for camera-attached
  draws (default 83, global — restore when done; used by `eSA_CamAttached` script attachments, see
  `features/script-attachments.md`); actor `get/set_actor_damage_stagger_time_factor` scales the
  slowdown (`time_to_lock` accel) a monster hit applies to the actor (`base_monster_feel.cpp`).
- **Zones:** `CZoneEffector` (`zone_effector.cpp`, the `flows/anomaly-zones.md` postprocess seam)
  creates a distance-driven `CPostprocessAnimatorLerp` as the actor nears a zone, factor modulated by
  outfit protection. *(agent-traced — its existence verified in F27.)*
- **Damage/weapon:** weapon fire pushes `CCameraShotEffector` (recoil); landing pushes `CEffectorFall`;
  monster psy abilities push `CControllerPsyHitCamEffector`. Bobbing/zoom-inertion are permanent
  effectors driven by movement/zoom state.

## Isolation ledger (the seams)

- **Actor camera:** both stacks live on `CActorCameraManager` (`Actor()->Cameras()`,
  `flows/actor.md`); camera effectors mutate the view, PP effectors the screen.
- **Renderer:** the composited `SPPInfo` is applied to the render target's post params each frame
  (`flows/render-frame.md`).
- **Zones:** `CZoneEffector` → `add_pp_effector` (`flows/anomaly-zones.md`).
- **Weapon/damage:** recoil/fall/psy camera effectors.
- **Lua:** `level.add_cam_effector`/`add_pp_effector`/`remove_pp_effector`/`set_pp_effector_factor*` +
  the optional `.anm` completion callback functor.
- **Config:** `.anm` (camera animation envelopes), `.ppe` (post-process envelopes), `.ltx` sections
  `[bobbing_effector]`/`[zoom_inertion_effector]`, zone `pp_eff_name`/`radius_min`/`radius_max`.

## Threading

Main thread (camera/render update). The composite runs once per frame in the camera manager update.

## Failure & edge cases

- **One effector per type:** adding a camera effector of an existing `ECamEffectorType` removes the old
  one — two recoils don't stack; the second replaces.
- **Camera composes by mutation, PP by addition:** camera effectors mutate the shared info in list
  order (later ones see earlier offsets); PP effectors sum into `SPPInfo` (or `bOverlap` to mask). A
  mis-ordered or non-`bOverlap` PP effect blends rather than replaces.
- **Lifetime vs cyclic:** a non-cyclic effector expires at `fLifeTime<=0` and auto-removes; a cyclic/
  permanent one (bobbing, zoom) must be removed explicitly or it persists.
- **Deferred camera add:** a camera effector added this frame applies next frame (deferred queue) —
  don't expect same-frame effect.
- **`.ppe`/`.anm` path errors:** a missing effect file fails the add silently-ish; the Lua id won't do
  anything.

## Deeper

- Camera effectors: `src/xrGame/EffectorFall.cpp` (`ProcessCam` — verified), `EffectorBobbing.cpp`,
  `EffectorZoomInertion.cpp`, `EffectorShot.cpp`, `CameraEffector.h` (`ECamEffectorType`),
  `ActorEffector.cpp`/`.h` (`CActorCameraManager`, `CAnimatorCamEffector`, `AddEffector`)
- PP effectors: `src/xrGame/PostprocessAnimator.cpp` (`.ppe`), `pp_effector_custom.cpp`,
  `pp_effector_distance.cpp`, `script_effector.cpp` (`CScriptEffector`)
- Manager (engine): `xrEngine/CameraManager.cpp`/`.h` (`m_EffectorsCam`/`m_EffectorsPP`, `SPPInfo`,
  `UpdateCamEffectors`/`UpdatePPEffectors`/`ApplyDevice`) — agent-traced
- Lua: `src/xrGame/level_script.cpp` (`add_cam_effector`, `add_pp_effector`, `remove_pp_effector`)
- Zone seam: `src/xrGame/zone_effector.cpp` (`CZoneEffector`) — `flows/anomaly-zones.md`

## Maintenance

- Keep the "camera = sequential mutation, PP = additive composite" distinction accurate — it explains
  how effects stack.
- Re-grep `level.add_cam_effector`/`add_pp_effector` if the Lua FX surface changes — it is the main mod
  entry point.
- Promote the *(agent-traced)* engine manager (`CCameraManager` stacks/`SPPInfo`) and the per-effector
  bodies to verified when a camera-FX or postprocess-tuning task needs the exact math.
