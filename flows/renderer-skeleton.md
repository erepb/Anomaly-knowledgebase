---
domain: flows/renderer-skeleton
tags: [engine, renderer, animation, r4, dx11, flow]
sources: [src/Layers/xrRender/SkeletonCustom.cpp, src/Layers/xrRender/SkeletonCustom.h, src/Layers/xrRender/SkeletonAnimated.cpp, src/Layers/xrRender/SkeletonAnimated.h, src/Layers/xrRender/SkeletonRigid.cpp, src/Layers/xrRender/SkeletonX.cpp, src/Layers/xrRender/SkeletonX.h, src/Layers/xrRender/Animation.cpp]
related: [flows/renderer-visual-types.md, flows/animation-callbacks.md, flows/physics-objects-ragdoll.md, flows/renderer-backend.md, engine/renderer.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: render-side skeleton animation / kinematics

How bones are stored, how their world matrices are computed each frame from playing animations, and how
that feeds GPU vertex skinning. This is the kinematics layer between the game animation timeline
(`flows/animation-callbacks.md` — motion marks, anim-end callbacks) and the skinned visual
(`flows/renderer-visual-types.md`). The game talks to it through the `IKinematics`/`IKinematicsAnimated`
interface; physics drives it through bone callbacks.

**Provenance:** researched with a read-only agent, then spot-verified against code. The class layering
(`CKinematics : FHierrarhyVisual, IKinematics`; `CKinematicsAnimated : CKinematics, IKinematicsAnimated`;
`CSkeletonX`), the bone-calc recursion (`CKinematics::Bone_Calculate` body read: `CLBone` per bone +
recurse children with the bone's transform as parent, callback seam visible in `BoneChain_Calculate`),
and the playback/skinning entry points (`CalculateBones`, `LL_PlayCycle`, `UpdateTracks`/`LL_UpdateTracks`,
`CSkeletonX::_Render`) are personally code-verified. Key interpolation/dequantization and the bone-matrix
constant upload are *(agent-traced — not personally re-read)*.

## Trigger / entry

The game plays an animation (`IKinematicsAnimated::PlayCycle/PlayFX`) and, each frame, something calls
`CalculateBones` (directly, or via the render/update path) to recompute bone world matrices; the skinned
visual's `Render` then uploads those matrices and submits geometry.

## Sequence (verified structure)

### Class layering

A skeleton **is a visual**: `CKinematics : FHierrarhyVisual, IKinematics` — a hierarchy of child
meshes (`CSkeletonX_*` from `flows/renderer-visual-types.md`) plus the bone system. `CKinematicsAnimated
: CKinematics, IKinematicsAnimated` adds animation playback. Two bone records: **`CBoneData`** (shared,
read-only — name, parent/children, bind pose, model-to-bone offset, IK/shape) and **`CBoneInstance`**
(per-instance, mutable — `mTransform` local, `mRenderTransform` world-for-GPU, a callback). The game
uses the abstract `IKinematics`/`IKinematicsAnimated` interface (downcast via `dcast_PKinematics*`).

### Bone calculation (verified)

`CKinematics::CalculateBones(bForceExact)` is **frame-gated** (early-out if already updated this frame,
plus distance/SSA culling to update distant skeletons less often), calls `OnCalculateBones()`
(`CKinematicsAnimated` overrides it to pump animations via `UpdateTracks`), then walks the hierarchy:
**`CKinematics::Bone_Calculate(CBoneData*, parent)`** fetches the bone instance, composes its matrix via
**`CLBone`**, and recurses into children passing `&BONE_INST.mTransform` as the new parent. `CLBone`
builds `mTransform = parent × bind` and, **if a bone callback is set, invokes it** — that is where
physics/ragdoll overrides the animated matrix (`flows/physics-objects-ragdoll.md`). It also composes
`mRenderTransform = mTransform × model-to-bone` (the GPU-ready matrix). `BoneChain_Calculate` is the
single-chain (leaf-to-root) variant used for queries.

### Animation playback (verified entries)

`CKinematicsAnimated::LL_PlayCycle(part, motionID, …)` allocates a `CBlend` from a fixed pool and adds
it to the per-bone blend lists for the targeted **partition** (body part — so legs and torso can play
different cycles); `LL_PlayFX` is the one-shot effect variant. `UpdateTracks` (frame-gated) →
`LL_UpdateTracks(dt, …)` advances each `CBlend` through its accrue→falloff state machine, advances
playback time, **fires the play callback once at end** (the game's anim-end hook —
`flows/animation-callbacks.md`), and destroys finished blends. The animated `BuildBoneMatrix`
dequantizes + blends the active animation keys per bone into `mTransform`. *(key dequant/mix agent-traced.)*

### Skinning render (verified entry)

`CSkeletonX::_Render(geom, vCount, iOffset, pCount)` dispatches by render mode: single-bone (one
transform), HW skinning 1–4 bones/vertex (upload the bones' `mRenderTransform` into a shader constant
array, then `RCache.Render`), or soft/CPU skinning (`xrCPU_Pipe`, locks a dynamic VB and transforms
vertices on the CPU). The vertex carries bone indices + weights; the GPU blends positions/normals by
the uploaded matrices. *(bone-matrix upload + soft path agent-traced.)*

## Function digests (spine)

- **`SkeletonRigid.cpp:CKinematics::CalculateBones`** — *Does:* once-per-frame guarded bone update
  (LOD-throttled) → `OnCalculateBones` → `Bone_Calculate(root)`. *Writes:* all `CBoneInstance`
  transforms. *Called by:* render/update path before skinning. *Gotcha:* frame-gated + distance-culled.
- **`SkeletonRigid.cpp:CKinematics::Bone_Calculate`** — *Does:* recursive hierarchy walk; `CLBone` per
  bone, recurse children. *(verified.)*
- **`SkeletonRigid.cpp:CKinematics::CLBone`** — *Does:* `mTransform = parent×bind`; **invoke bone
  callback if set** (physics seam); compose `mRenderTransform`. *(callback seam verified via
  `BoneChain_Calculate`; body agent-traced.)*
- **`SkeletonAnimated.cpp:CKinematicsAnimated::LL_PlayCycle`** — *Does:* allocate + register a `CBlend`
  on a partition. *Reads:* motion set. *Writes:* per-bone blend lists, `blend_cycles`. *Called by:*
  game animation. *(verified entry.)*
- **`SkeletonAnimated.cpp:CKinematicsAnimated::UpdateTracks` / `LL_UpdateTracks`** — *Does:* advance
  blends (accrue/falloff), advance time, fire end callbacks, destroy finished. *Called by:*
  `OnCalculateBones`. *(verified entries.)*
- **`SkeletonX.cpp:CSkeletonX::_Render`** — *Does:* dispatch skinning mode; upload bone matrices; submit
  geometry. *Calls:* `RCache.Render`. *Called by:* the skinned visual's `Render`. *(verified entry.)*

## Isolation ledger (the seams)

- **Owned state:** shared `CBoneData` hierarchy + per-instance `CBoneInstance` transforms; the `CBlend`
  pool + per-bone blend lists; loaded motion sets.
- **Game animation seam:** `IKinematicsAnimated::PlayCycle/PlayFX` (start blends), the play callback
  fired at animation end, and motion marks — all in `flows/animation-callbacks.md`.
- **Physics/ragdoll seam:** `CBoneInstance` callbacks — when set (with overwrite), physics writes the
  bone matrix and animation is overridden; this is how a corpse goes ragdoll
  (`flows/physics-objects-ragdoll.md`).
- **Visual seam:** `CKinematics` *is* an `FHierrarhyVisual`; its `CSkeletonX_*` children render skinned
  geometry using the bone matrices (`flows/renderer-visual-types.md`).
- **Backend seam:** `CSkeletonX::_Render` uploads bone matrices as shader constants and submits via
  `RCache` (`flows/renderer-backend.md`); motion-vector builds also upload previous-frame matrices.
- **Config:** skeleton-update throttling (distance/SSA), `psSkeletonUpdate` bbox cadence.

## Threading

`CalculateBones`/`UpdateTracks` are frame-gated (once per frame, guarded by the global time) and run on
the update/render thread; soft skinning uses `xrCPU_Pipe`.

## Failure & edge cases

- **Frame-gated** — calling `CalculateBones` twice a frame recomputes only once; force with
  `CalculateBones_Invalidate` if a mid-frame change must show.
- **Bone callback overrides animation** — if a physics callback is active, the animated pose is ignored
  for that bone (ragdoll). A "stuck/ignoring animation" bone is often a leftover physics callback.
- **Distance culling** updates far skeletons less often — fast actions on distant NPCs may visibly lag.
- **Partition blends** mean two cycles can fight over shared bones across channels.

## Deeper

- Bone hierarchy + calc: `xrRender/SkeletonCustom.cpp`/`.h` (`CKinematics`), `SkeletonRigid.cpp`
  (`CalculateBones`, `Bone_Calculate`, `CLBone`, `BoneChain_Calculate`)
- Animation playback: `xrRender/SkeletonAnimated.cpp`/`.h` (`CKinematicsAnimated`, `LL_PlayCycle`,
  `UpdateTracks`/`LL_UpdateTracks`, animated `BuildBoneMatrix`), `Animation.cpp` (`CBlend`)
- Skinning render: `xrRender/SkeletonX.cpp`/`.h` (`CSkeletonX::_Render`)
- Game animation: `flows/animation-callbacks.md`; ragdoll: `flows/physics-objects-ragdoll.md`; visual:
  `flows/renderer-visual-types.md`

## Maintenance

- `CalculateBones`→`Bone_Calculate`→`CLBone` is the bone spine; `_Render` is the skinning spine —
  re-grep if the bone-instance model or skinning dispatch changes.
- Promote the *(agent-traced)* key dequant/mix (`AnimationKeyCalculate.h`, animated `BuildBoneMatrix`)
  and the bone-matrix constant upload (`SkeletonX.cpp` HW path) to verified when an animation/skinning
  task needs exact behavior.
- Keep the "bone callback overrides animation" fact — it is the animation↔physics handoff.
