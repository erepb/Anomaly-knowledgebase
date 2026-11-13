---
domain: features/script-attachments
tags: [engine, lua, animation, weapons, ui, physics, renderer, dx11, cpp]
sources: [src/xrGame/script_attachment_manager.cpp, src/xrGame/script_attachment_manager.h, src/xrGame/script_attachment_script.cpp, src/xrGame/iscript_attachment.h, src/xrGame/UI3dStatic.cpp, src/xrGame/UI3dStatic.h, src/xrGame/player_hud.cpp, src/xrGame/player_hud.h, src/xrGame/HudItem.cpp, src/xrGame/level_script.cpp, src/xrGame/physics_shell_scripted.cpp, gamedata/shaders/r3/ui_icons.s, gamedata/shaders/r3/ui_icons_pp.s, gamedata/meshes/anomaly_weapons/hands/bone_parts.ltx, gamedata/scripts/lua_help_ex.script]
related: [flows/object-lifecycle.md, flows/animation-callbacks.md, flows/weapon-hierarchy-addons.md, flows/hud-ui-load.md, flows/physics-objects-ragdoll.md, flows/renderer-scene-graph.md, features/shader-bus.md]
updated: 2026-11-13
---

# Script attachments, 3D UI icons, first-person IK

The "Scripted Weapon Attachments" series (PRs #714/#719/#724/#740/#741/#817, merged 2026.11.09). It
lets Lua hang arbitrary models on game objects, HUD items or the camera, draw live 3D models as UI
icons, pose/IK the first-person arms, and drive HUD animation timing. External tutorial linked from
`lua_help_ex.script` ("Script Attachment Guide"); Lua signatures live in `lua_help_ex.script`.

## `script_attachment` (`script_attachment_manager.*`)

One attached model. It is an `IRenderable` + `ICollidable` registered in the spatial DB. Key state:

- **Type** (`script_attachment_type`): `eSA_HUD` (parented to a first-person HUD model; arm bones
  below 21 pick `m_model_2`), `eSA_World` (on a world object), `eSA_CamAttached` (camera-space, own
  projection FOV via `set_custom_fov` / global `level.set_cam_attached_fov`).
- **Hierarchy** — owned by a `CGameObject` (`m_script_attachments`, created by
  `obj:add_attachment(name, model)`, or `level.add_attachment` for camera/HUD) or by another
  attachment (`m_children`, `add_attachment` on an attachment). `get_toplevel_parent` walks up.
- **Transform** — position/rotation/origin/scale (`m_attachment_offset[4]`); **scale is a vector**
  (`get_scale` returns a vector, `get_scale_val` a single number for older scripts). Parent bone via
  `set_parent_bone`. `bone_callback` overrides an attachment bone from a parent bone or a Lua functor
  returning `Fmatrix` (`script_attachment_bone_cb`).
- **Animation** — `play_motion` with an end callback (`m_motion_end_cb`).
- **Physics** — optional shell from the model's shapes (`set_use_ph_shapes`, `ph_freeze`,
  `set_ph_contact_callback`); collision form `CCF_Attachment_Skeleton`. Changing scale re-enables the
  shell to avoid clipping. Shell/element scale is also a vector now (`physics_shell_scripted.cpp`).
- **Per-attachment UI** — `set_ui(func)` renders a `CUIWindow` in 3D at a bone/offset; drawn in the
  forward stage *before* the sorted HUD via `IRenderable::renderable_RenderUI`
  (`r__dsgraph_render.cpp`). Kept visible during item/script animations.
- **Light** — `attach_light` (`AttachmentScriptLight`) on a bone.
- **Shaders** — `get_shaders` / `set_shader` / `reset_shader` per model surface.
- **Model instances / multirender** — `model_instance(obj)` renders another game object's visual
  through the attachment; `set_instance_multirender` allows one visual in several places (forced on
  for objects used as 3D icons).
- **Shader bus** — per-attachment lane values (`shader_bus.set_attachment`, see `features/shader-bus.md`).

Update cadence: `CGameObject` update calls each attachment's `Update()`; HUD attachments get their
XFORM in `player_hud::update` after the arm IK pass.

## 3D UI icons (`CUI3dStatic`)

A `CUIStatic` that draws a model (`SetVisual`) or a game object (`SetObject`, including its
attachments) with rotation/scale. Two paths:

- **Direct** — renders each icon in the UI pass.
- **DX11 atlas** (`r4_atlas 1`) — `CUI3dStatic::PrepareAtlas` (called by `HUDManager` and
  `MainMenu` before the UI traversal) renders all queued icons into one MSAA atlas, then statics blit
  their rect. Cvars `r4_atlas_resolution`/`_width`/`_height`/`_msaa`. Shaders `ui_icons` (r1–r3) and
  `ui_icons_pp` (r3).

## First-person arms: hand poses, IK, timing

All in `player_hud`:

- **Hand poses** — `game.set_hand_pose(hand, section, anim, accrue, falloff)` / `clear_hand_pose`
  override one hand's animation (hand 0 right, 1 left, 2 both). `bone_parts.ltx` /
  `wpn_hand_00.ltx` under `meshes/anomaly_weapons/hands/` define the per-hand bone parts this needs.
- **Arm IK** — `player_hud::finalize_hands_ik` runs once per frame after HUD layers: it calls
  `_G.COnHudIKTarget` (Lua sets targets) then `solve_hand_ik` for each arm (`ArmIK`: 3-bone chain +
  pole). `level.set_arm_ik_enabled(hand, bool)`. `hud_ik_compensation` anchors IK shoulders to the
  camera + idle HUD offset (`update_ik_shoulder_reference`).
- **Timing / motion marks** — `set_part_cycle_time` (normalized or absolute seconds),
  `get/set_hand_motion_marks`, `motion_is_stop_at_end`; HUD animation timing helpers were moved out
  of the per-weapon classes into `HudItem` (`motion_length` gained `ignore_stop_at_end`). Motion-mark
  interval times are forwarded to Lua callbacks.
- **Script anim item model** — `create_script_anim_item_model(section)`; hook
  `actor_on_create_hud_motion_item` (`_G.player_hud__OnCreateHudMotionItem`).
- `shell_particles` can be overridden per HUD section (`m_shell_particles_override`).

## Deeper

- Attachment class: `script_attachment_manager.h:script_attachment`; Lua surface:
  `script_attachment_script.cpp` (`.def` list)
- Game object side: `script_game_object_script3.cpp` (`add_attachment`, `get_attachment`)
- Level side: `level_script.cpp` (`add_attachment`, `set_arm_ik_enabled`, `set_hand_pose`)
- Icons: `UI3dStatic.cpp:CUI3dStatic::PrepareAtlas`, `CUI3dStatic::Draw`
- IK: `player_hud.cpp:player_hud::solve_hand_ik`, `player_hud::finalize_hands_ik`

## Maintenance

- Re-list `.def(` names in `script_attachment_script.cpp` when the API changes.
- Confirm `_G.COnHudIKTarget` is still the IK hook name in `player_hud.cpp`.
- Note: an earlier "additive animations" change was reverted (`b86d733f4`); no additive-anim code is in
  the tree — don't document it.
