---
domain: flows/scheme-animpoint
tags: [modlayer, gameplay, unpacked, lua, schemes, animpoint, smartcover, flow]
sources: [unpacked/scripts/xr_animpoint.script, unpacked/scripts/xr_animpoint_predicates.script, unpacked/scripts/xr_campfire_point.script]
related: [flows/scheme-walker-patrol.md, flows/smart-cover.md, flows/scheme-engine-core.md, flows/state-manager-core.md, flows/special-actors.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: animpoint scheme (F77)

NPCs that occupy a **smartcover** and play looped "animpoint" animations — sitting at a campfire,
leaning on a wall, eating, playing guitar. A **delta on the walker template** (F74) tied to the
smart-cover system (F33 `flows/smart-cover.md`). Its distinctive shape is a **two-action split**:
`action_reach_animpoint` walks the NPC to the cover position, then `action_animpoint` loops cover
animations, with a **predicate filter** choosing which actions are allowed and **camp director/listener**
roles coordinating NPCs around a campfire.

**Provenance:** the **architecture is verified** in `xr_animpoint.script` — the two actions
(`action_reach_animpoint:execute`, `action_animpoint:execute`) and the animpoint object's spine
(`calculate_position`, `start`, `update`, `set_scheme`) are confirmed present. The bodies of
`calculate_position` (smartcover lookup), `update` (camp role dispatch), and the predicate/association
tables are *(agent-traced — not personally re-read)*. No `gamedata/` overrides (verified by listing).

## Sequence (spine — architecture verified, bodies agent-traced)

### Config — `xr_animpoint.script:set_scheme`
Reads `cover_name` (the target smartcover, default `$script_id$_cover`), `use_camp` (default true),
`reach_distance`/`reach_movement` (the approach), an optional `direction` override, and an optional
`avail_animations` manual override. Stores the **animpoint object** on `st`.

### Reach phase — `action_reach_animpoint:execute`
Drives the NPC to the cover: `animpoint:calculate_position()` resolves the smartcover (by name from
`se_smart_cover.registered_smartcovers`) → position/vertex/look/`description`; then
`set_dest_level_vertex_id` + `state_mgr.set_state(reach_movement)` until within `reach_distance`. The
`evaluator_reach_animpoint` flips true on arrival, handing off to the in-cover action.

### In-cover phase — `action_animpoint:execute`
Per frame: `animpoint:start()` (once — registers with the campfire if `use_camp`), then
`animpoint:update()` (pick the next action), then
`state_mgr.set_state(action, {look_position}, {animation_position, animation_direction})` — the
position/direction-locked animation that seats the NPC in the cover (the `state_mgr` extra params, F70).

### Action selection — predicates + camp roles (agent-traced)
- `calculate_position` reads the smartcover's `description()` (e.g. `animpoint_sit_ass`) and looks it up
  in `xr_animpoint_predicates.associations` to get the candidate action set; `fill_approved_actions`
  filters by per-NPC `predicate(npc_id, is_in_camp)` (sleep/roast/guitar/PDA conditions).
- With a campfire (`use_camp`), `animpoint:update` asks `camp.get_camp_action(npc_id)` → `(action,
  is_director)` and uses an `assoc_tbl[camp_action][director|listener]` to pick a synchronized action
  (idle/harmonica/guitar/story), so the camp performs coordinated activity. Listeners delay role changes.

## Function digests (spine)

- **`xr_animpoint.script:set_scheme`** — *Does:* read cover/camp config; create animpoint object.
  *(verified present.)*
- **`xr_animpoint.script:action_reach_animpoint:execute`** — *Does:* navigate to the cover position.
  *(verified present.)*
- **`xr_animpoint.script:action_animpoint:execute`** — *Does:* loop cover animations via the animpoint
  object + `state_mgr`. *(verified present.)*
- **`xr_animpoint.script:animpoint:calculate_position`** — *Does:* resolve smartcover → position +
  `description` + approved actions. *(verified present; body agent-traced.)*
- **`xr_animpoint.script:animpoint:update`** — *Does:* pick the next action (camp role / predicate).
  *(verified present; body agent-traced.)*
- **`xr_animpoint_predicates.script:associations` / predicates** — *Does:* description → action set + the
  per-NPC filters. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** the animpoint object on `db.storage[id][scheme]` (position, vertex, look, description,
  approved_actions, current_action, camp). Same scheme framework (F66/F74).
- **Smart-cover seam (F33):** `se_smart_cover.registered_smartcovers[cover_name]` supplies position/angle/
  `description`; the description maps (via `smart_covers.descriptions`) to loophole animations.
- **Camp seam:** `sr_camp` (`get_current_camp`/`register_npc`/`get_camp_action`) coordinates campfire
  roles; `db.campfire_table_by_smart_names` for roast proximity. Campfire registration via
  `xr_campfire_point` / `bind_campfire`.
- **state_mgr seam:** uses the position/direction-locked `set_state` extra params (F70) to seat the NPC.

## Threading / failure / edge cases

- **Two-action handoff:** an NPC stuck "walking forever" to an animpoint usually failed the reach
  distance / accessibility — check the smartcover vertex is reachable.
- **Predicate gating (agent-traced):** guitar/harmonica only in camp; roast needs a nearby active
  campfire; a ~20% night chance can set `need_sleep` and switch the action set.
- **Director/listener (agent-traced):** one camp NPC is the director driving the activity; others sync as
  listeners with a delay — campfire scenes are coordinated, not independent.

## Deeper

- Scheme: `unpacked/scripts/xr_animpoint.script` (`set_scheme`, `action_reach_animpoint:execute`,
  `action_animpoint:execute`, `animpoint:calculate_position`/`start`/`update`)
- Predicates: `unpacked/scripts/xr_animpoint_predicates.script` (`associations`, predicate fns)
- Campfire: `unpacked/scripts/xr_campfire_point.script`; camp coordination: `flows/special-actors.md` (sr_camp)
- Template: `flows/scheme-walker-patrol.md`; smartcovers: `flows/smart-cover.md`; animation:
  `flows/state-manager-core.md`

## Maintenance

- Architecture (two-action split + animpoint object) is verified; promote `calculate_position`/`update`/
  the predicate tables to verified when a task needs exact action selection.
- Check `gamedata/scripts/` for overrides before trusting (none at writing).
