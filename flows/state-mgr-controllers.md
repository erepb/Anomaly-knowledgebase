---
domain: flows/state-mgr-controllers
tags: [modlayer, gameplay, unpacked, lua, state-mgr, animation, flow]
sources: [unpacked/scripts/state_mgr_animation.script, unpacked/scripts/state_mgr_animstate.script, unpacked/scripts/state_mgr_movement.script, unpacked/scripts/state_mgr_mental.script, unpacked/scripts/state_mgr_bodystate.script, unpacked/scripts/state_mgr_direction.script, unpacked/scripts/state_mgr_weapon.script, unpacked/scripts/state_mgr_animation_list.script]
related: [flows/state-manager-core.md, flows/scheme-engine-core.md, flows/animation-callbacks.md, modlayer/gameplay/state-manager.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: state_mgr sub-controllers (F71)

The per-channel controllers the state_mgr GOAP planner (F70) drives to converge an NPC on a named
`state_lib` state. Each channel (weapon, movement, mental, bodystate, direction, animstate, animation)
contributes evaluators (am I already at target?) and operators (act toward target) to the planner. The
**animation** channel is the only one with real internal complexity — an IN→IDLE→OUT marker machine fed
by the engine's animation-end callback. This deepens F70 (`flows/state-manager-core.md`).

**Provenance:** the **animation marker machine is verified** in `state_mgr_animation.script`
(`animation:animation_callback` read in full; `select_anim` + the `MARKER_IN/OUT/IDLE = 1/2/3` constants
confirmed). The other six channels' eva/act symbols are *(agent-traced — not personally re-read)*; they
follow one uniform pattern (compare `state_lib.states[target].<field>` to the NPC's current, then call an
engine `set_*`). No `gamedata/` overrides of the `state_mgr_*` files (agent-checked).

## The uniform channel pattern *(agent-traced; verified for animation)*

For a simple channel X, the planner has:
- **evaluator** `eva_state_mgr_X` — true when the NPC's current X already equals `state_lib.states[target].X`.
- **operator(s)** `act_state_mgr_X_<value>` — call the engine setter to move toward the target value.

The per-channel setters (agent-traced):
- **movement** (`state_mgr_movement.script`) → `npc:set_movement_type(move.walk|run|stand)`; turn/search
  variants also drive direction.
- **mental** (`state_mgr_mental.script`) → `npc:set_mental_state(anim.free|danger|panic)`.
- **bodystate** (`state_mgr_bodystate.script`) → `npc:set_body_state(move.standing|crouch)`; combined
  ops (crouch_danger, standing_free) set body + mental together.
- **weapon** (`state_mgr_weapon.script`) → `npc:set_item(object.strap|idle|…, weapon)` for
  strapped/unstrapped/none/fire/drop; fires `npc_on_weapon_strapped`/`_unstrapped`/`_drop` callbacks;
  helpers `get_weapon`/`strappable_weapon`.
- **direction** (`state_mgr_direction.script`) → `npc:set_sight(...)`; evaluator compares
  `npc:sight_params()` to the requested look (position/object/dir) within tolerance and fires the
  turn-end callback on match.
- **animstate** (`state_mgr_animstate.script`) — drives the *second* animation channel (sit/stand poses);
  it is the **same `animation` class** instantiated against `state_mgr_animstate_list` (vs the full
  `state_mgr_animation_list`), reusing the marker machine below.

## The animation marker machine (verified)

The `animation` class (in `state_mgr_animation.script`) plays a named animation as three phases, tracked
by `states.anim_marker` (`MARKER_IN=1`, `MARKER_OUT=2`, `MARKER_IDLE=3`) and a `seq_id` cursor:

- **`animation:select_anim`** (verified by structure) picks the next clip and sets the marker: when the
  target differs and a current state must be torn down → play the current state's `out[]` clips
  (MARKER_OUT); when entering a new state → play its `into[]` clips (MARKER_IN); when current==target →
  play `idle`/`rnd` clips (MARKER_IDLE). Clips are chosen per **weapon slot** (`anim_for_slot`).
- **`animation:animation_callback(skip_multianim_check)`** (verified, read in full) — the engine calls it
  when `npc:animation_count()` hits 0:
  - **MARKER_IN:** if more `into` clips remain (`#into_table > seq_id`), advance `seq_id` and continue;
    otherwise reset `seq_id=1`, **set `current_state = target_state`** (the state is now "reached"), and
    re-drive. This is what makes `eva_state_mgr_animation` return true (current==target).
  - **MARKER_IDLE:** schedule the next random/idle via `next_rnd` (from the state's `prop.sumidle`/
    `maxidle`), then re-drive.
  - **MARKER_OUT:** advance through `out` clips; when done, **set `current_state = nil`** (animation
    stopped), so the channel reports "none."
- **`animation:set_state(new_state, fast_set)`** sets the target; `fast_set` skips the IN/OUT sequencing
  for an immediate jump (used for combat/panic snappiness). `update_anim`→`select_anim`→`add_anim`
  pushes the chosen clip to the engine (`npc:add_animation`, with position/direction from the manager).

So "state reached" for the animation channel is signalled by the callback setting `current_state =
target_state`; the planner's `eva_state_mgr_animation` then passes and the overall state converges (F70).

## Function digests (spine)

- **`state_mgr_animation.script:animation:animation_callback`** — *Does:* advance the IN/IDLE/OUT marker
  on animation-end; set `current_state` to target (reached) or nil (stopped). *Reads:* `npc:animation_count()`,
  `anim_path.animations[state]`. *(verified.)*
- **`state_mgr_animation.script:animation:select_anim`** — *Does:* choose the next clip + set the marker
  (out/into/idle) per weapon slot. *(verified by structure.)*
- **`state_mgr_animation.script:animation:set_state` / `update_anim` / `add_anim`** — *Does:* set target,
  select, and push the clip to the engine. *(verified head.)*
- **per-channel `eva_state_mgr_<X>` / `act_state_mgr_<X>_*`** — *Does:* compare current vs
  `state_lib.states[target].X` and call the engine setter. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** each channel's controller object on the `state_manager` (notably
  `st.animation`/`st.animstate` with `states.{current_state, target_state, anim_marker, seq_id, next_rnd}`).
- **Driver (in):** the state_mgr GOAP planner (`goap_graph`, F70) registers these evas/acts and steps them
  in `state_manager:update`.
- **Engine seam (out):** `npc:set_movement_type`/`set_mental_state`/`set_body_state`/`set_item`/`set_sight`/
  `add_animation`, and the **animation-end callback** the engine fires into `animation_callback`
  (`flows/animation-callbacks.md`).
- **Data:** clip definitions in `state_mgr_animation_list` / `state_mgr_animstate_list` (name → into/idle/
  rnd/out by weapon slot + `prop`); the target fields in `state_lib.states` (F70).

## Threading / failure / edge cases

- **Convergence is callback-paced:** the animation channel only advances on `animation_count()==0`, so a
  long clip blocks the marker transition — states with heavy `into` sequences take time to "reach."
- **`fast_set` bypass:** skips IN/OUT; use it (via `set_state` extra) when an instant posture change is
  needed (combat) — otherwise transitions play their full into/out clips.
- **Weapon-slot keying:** clips are selected by `weapon_slot()`; a state can look different per weapon, and
  a missing slot falls back to slot 0. Wrong/with-no-weapon NPCs pick the slot-0 clip.
- **animstate vs animation:** two independent channels share the class; a pose (animstate) and a full
  animation can both be active — don't conflate their `current_state`s.

## Deeper

- Animation machine: `unpacked/scripts/state_mgr_animation.script` (`animation:animation_callback`,
  `select_anim`, `set_state`); clip tables: `state_mgr_animation_list.script` / `state_mgr_animstate_list.script`
- Channels: `state_mgr_movement.script`, `state_mgr_mental.script`, `state_mgr_bodystate.script`,
  `state_mgr_weapon.script`, `state_mgr_direction.script`, `state_mgr_animstate.script`
- Core + planner: `flows/state-manager-core.md` (F70); engine callback: `flows/animation-callbacks.md`

## Maintenance

- The verified animation marker machine is the stable, complex core; re-read `animation_callback` if
  animation sequencing changes.
- The six simple channels are agent-traced under one pattern — read + promote the specific channel a task
  touches (each is small).
- Check `gamedata/scripts/` for `state_mgr_*` overrides before trusting (none at writing).
