---
domain: flows/state-manager-core
tags: [modlayer, gameplay, unpacked, lua, state-mgr, animation, goap, flow]
sources: [unpacked/scripts/state_mgr.script, unpacked/scripts/state_lib.script, unpacked/scripts/state_mgr_goap.script, unpacked/scripts/xr_motivator.script]
related: [flows/scheme-engine-core.md, flows/animation-callbacks.md, flows/ai-planner.md, flows/db-registry.md, modlayer/gameplay/state-manager.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: state manager core — state_mgr / state_lib (F70)

The per-NPC animation/posture/movement state machine that schemes drive. A scheme calls
`state_mgr.set_state(npc, "guard")`; the bound `state_manager` stores that target and, every frame,
runs a GOAP planner that walks sub-controllers (weapon, movement, mental, bodystate, animation,
animstate, direction) until the NPC matches the named state in `state_lib`. This is what makes an NPC
actually sit/walk/aim; the scheme layer (F66) only names the desired state.

**Provenance:** **verified** in `state_mgr.script`: the global `set_state`/`get_state` wrappers,
`state_manager:set_state`, `state_manager:update` (the callback-timeout + `planner:update()` loop), and
the `state_lib.states[name]` entry structure (read in `state_lib.script`). `bind_manager` head verified;
its full planner wiring and `state_mgr_goap.goap_graph` (the evaluator/operator registration) are
*(agent-traced — not personally re-read)*, as are the sub-controllers (F71). The motivator seam is
verified (F66). No `gamedata/` override of the `state_mgr*`/`state_lib` files (agent-checked).

## Concepts (verified)

- **named state** — an entry in `state_lib.states` (verified structure): a table of
  `{weapon, movement, mental, bodystate, animstate, animation, direction, special_danger_move, ...}`,
  each field a desired sub-controller target (or `nil` = don't care). E.g. `idle` is all-nil;
  `jump_down_0` sets weapon/movement/mental/bodystate/animstate/animation/direction.
- **target_state** — the state the manager is driving toward (`self.target_state`), default `"idle"`.
- **state_mgr** — the per-NPC `state_manager` instance, stored at `db.storage[id].state_mgr`, bound by
  `bind_manager` and ticked by the motivator.

## Sequence (verified spine)

### Request a state — `state_mgr.script:set_state(npc, state_name, callback, timeout, target, extra)`
Verified: looks up `db.storage[id]`; if `st.state_mgr` and a name is given, forwards to
`st.state_mgr:set_state(...)`. The thin global wrapper schemes call.

### Apply the request — `state_mgr.script:state_manager:set_state(state_name, callback, timeout, target, extra)`
Verified: errors if `state_lib.states[state_name]` is nil (illegal state); sets look params
(`look_position`/`look_object`/`look_dir`) from `target`; **no-ops if already the target_state**; on a
fire→non-fire weapon transition, idles the weapon via `npc:set_item`; sets `target_state`; toggles
`npc:special_danger_move` from the state's flag; stores `animation_position`/`direction` + `fast_set`
from `extra`; records the optional `callback`/`timeout`. So `set_state` just *records intent* — no
animation is forced here.

### Drive toward it — `state_mgr.script:state_manager:update()`
Verified: (1) if a callback is pending and the current animation has reached the target state's
`animation`, fire `callback.func` after `callback.timeout`; (2) `self.planner:update()` — run the GOAP
planner one cycle, which selects/advances the sub-controller operators (weapon/move/mental/bodystate/
anim/animstate/direction) needed to converge on the target state; the planner's goal is the synthetic
`end` property, reached when all the state's fields are satisfied. Called once per frame from the
motivator.

### Bind — `state_mgr.script:bind_manager(object)` *(head verified; rest agent-traced)*
Creates the `state_manager` for the object and wires it into the NPC's `motivation_action_manager`
(GOAP), registering the evaluators + operators for each sub-controller and the manager-control
properties (`end`/`locked`/…). The detailed property/operator set is `state_mgr_goap.goap_graph`
(agent-traced).

## Function digests (spine)

- **`state_mgr.script:set_state` / `get_state` (globals)** — *Does:* forward to the NPC's bound
  `state_manager`. *Reads:* `db.storage[id].state_mgr`. *(verified.)*
- **`state_mgr.script:state_manager:set_state`** — *Does:* record target state + look/callback/extra;
  validate against `state_lib`; handle weapon-idle on fire-exit. *Writes:* `target_state`,
  `look_*`, `animation_*`, `callback`. *(verified.)*
- **`state_mgr.script:state_manager:update`** — *Does:* callback-timeout fire + `planner:update()` GOAP
  convergence. *Called by:* `xr_motivator:update` (`st.state_mgr:update()`). *(verified.)*
- **`state_lib.script:states`** — *Does:* the named-state table; each entry the sub-controller target
  set. *(structure verified.)*
- **`state_mgr.script:bind_manager` / `state_mgr_goap.script:goap_graph`** — *Does:* build the planner +
  register evaluators/operators. *(head verified; graph agent-traced.)*

## Sub-controllers (F71 — named only)

The planner drives these (each compares the target state's field to the NPC's current and acts):
`state_mgr_weapon` (strapped/unstrapped/fire/none/drop), `state_mgr_movement` (walk/run/stand),
`state_mgr_mental` (free/danger/panic), `state_mgr_bodystate` (standing/crouch), `state_mgr_animation`
+ `state_mgr_animstate` (the two animation channels), `state_mgr_direction` (sight/look convergence).
Full bodies + the IN/OUT/IDLE animation marker machine are roadmap **F71**
(`flows/state-manager-core.md` is the core; F71 deepens the controllers).

## Isolation ledger (the seams)

- **Owned state:** the per-NPC `state_manager` at `db.storage[id].state_mgr` (`target_state`, `look_*`,
  `animation_*`, `callback`, `planner`, sub-controllers). The global `state_lib.states` table.
- **Driver (in):** `xr_motivator:update` calls `st.state_mgr:update()` every frame (F66). Schemes call
  `state_mgr.set_state` from their action `execute` (F66 / the scheme catalog).
- **Engine seam (out):** sub-controllers call engine methods (`npc:set_movement_type`,
  `set_mental_state`, `set_body_state`, `set_item`, sight params, animation callbacks —
  `flows/animation-callbacks.md`) and the GOAP `motivation_action_manager` (`flows/ai-planner.md`).
- **State definitions:** extended by `state_lib_animpoint`/`state_mgr_scenario`/`state_mgr_pri_a15`
  `add_state_lib()` merges *(agent-traced)*.

## Threading / failure / edge cases

- **Illegal state aborts the call:** `set_state` with a name not in `state_lib.states` logs an error and
  returns — a typo'd state name silently leaves the NPC in its prior state.
- **set_state is idempotent:** re-requesting the current `target_state` is a no-op; to force a redo you
  must change state.
- **Convergence is multi-frame:** `update` advances the planner incrementally; a state isn't reached the
  frame it's requested. Callbacks fire only once the *animation* channel matches and the timeout elapses.
- **Weapon fire-exit special-case:** leaving a `fire`/`sniper_fire` state idles the weapon explicitly —
  relevant when debugging NPCs that keep aiming.

## Deeper

- Manager: `unpacked/scripts/state_mgr.script` (`set_state`, `state_manager:set_state`/`:update`,
  `bind_manager`)
- States: `unpacked/scripts/state_lib.script` (`states`)
- Planner graph: `unpacked/scripts/state_mgr_goap.script` (`goap_graph`)
- Driver: `flows/scheme-engine-core.md` (`xr_motivator:update`); animation: `flows/animation-callbacks.md`;
  GOAP: `flows/ai-planner.md`; sub-controllers deepened in roadmap F71

## Maintenance

- The verified core (set_state record + update planner-loop + state_lib entry shape) is stable; re-read
  if state fields or the update loop change.
- Promote `bind_manager`/`goap_graph` + the sub-controllers to verified when F71 is built.
- Check `gamedata/scripts/` for `state_mgr*`/`state_lib` overrides before trusting (none at writing).
