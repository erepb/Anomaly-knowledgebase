---
domain: flows/scheme-combat
tags: [modlayer, gameplay, unpacked, lua, schemes, combat, flow]
sources: [unpacked/scripts/xr_combat.script, unpacked/scripts/xr_combat_ignore.script, unpacked/scripts/xr_combat_camper.script, unpacked/scripts/xr_combat_zombied.script, unpacked/scripts/xr_combat_monolith.script, unpacked/scripts/xr_danger.script, unpacked/scripts/xr_hit.script]
related: [flows/scheme-engine-core.md, flows/scheme-walker-patrol.md, flows/agent-squad-combat.md, flows/ai-perception.md, flows/damage-hit.md, flows/condlist-library.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: combat schemes — combat type, ignore, danger, hit (F76)

How scripted NPCs fight: a small Lua layer that **selects a combat style**, **filters who counts as an
enemy** (combat_ignore), **reacts to danger and hits**, and optionally **overrides the engine combat
planner** with special styles (zombied/camper/monolith). The heavy lifting is the engine's GOAP combat
planner (`stalker_ids.action_combat_planner`, the C++ side is F35 `flows/agent-squad-combat.md`); these
schemes gate and parameterize it. Built on F66 (`flows/scheme-engine-core.md`).

**Provenance:** **verified** in `xr_combat.script`: `set_combat_type` (the `combat_type` condlist →
`script_combat_type`), `evaluator_check_combat:evaluate` (the gate), and `add_to_binder` (it grabs
`manager:action(stalker_ids.action_combat_planner)`). `xr_combat_ignore` (the `set_enemy_callback`
filter), `xr_danger`, `xr_hit`, and the three special styles are **confirmed present** and documented
*(agent-traced — not personally re-read)*; combat is intricate, so several details are flagged. No
`gamedata/` overrides of any of the seven files (verified by listing).

## Sequence (spine)

### Pick the combat style — `xr_combat.script:set_combat_type(npc, actor, t)` (verified)
Called from `xr_motivator:update` when `st.combat` is set (F66). Verified: evaluates the `combat_type`
condlist (`xr_logic.pick_section_from_condlist(actor, npc, t.combat_type.condlist)`); the string `"nil"`
becomes Lua `nil` (= use the engine's default combat); stores the result in `storage.script_combat_type`
and `t.script_combat_type`. So an NPC's combat style is a condlist-selected value (zombied/camper/
monolith/engine-default), re-evaluated as combat begins.

### Gate the engine planner — `xr_combat.script:evaluator_check_combat:evaluate` + `add_to_binder` (verified)
`evaluator_check_combat:evaluate` returns `st.script_combat_type ~= nil` — i.e. "a script style is
active." `add_to_binder` fetches the engine combat action (`manager:action(stalker_ids.action_combat_planner)`)
and gates it on this evaluator, and conditionally binds the special-style actions. So: **script style set
→ engine combat planner suppressed → the special-style action runs instead**; no script style → the
engine's own combat planner handles the fight.

### Filter enemies — `xr_combat_ignore.script` *(agent-traced)*
Registers an engine `npc:set_enemy_callback` (in `reset_generic_scheme`); the callback's `is_enemy(obj,
enemy)` decides whether the engine may treat someone as an enemy. It applies distance/visibility/weather
rules, safe-zone + hostage exemptions, faction peace (`xr_bribe.at_peace`), and the condlist overrides
`combat_ignore_cond` / `no_combat_job` / `enemy_ignore_cond` (from `st.overrides`, F68). Returning false
makes the NPC ignore that enemy. Also fires `npc_on_fighting_actor` and can disable combat_ignore for a
squad when the actor hits them. This is the main "why won't this NPC fight / why is it ignoring me" knob.

### React to danger — `xr_danger.script` *(agent-traced)*
An evaluator (`evaluator_danger`) reads `npc:best_danger()` (grenade / corpse / being-attacked), validates
it (`is_danger`: type, distance radius from `ai_tweaks/xr_danger.ltx`, wounded/safe-zone/relation
exemptions), applies an **inertia** window so danger persists briefly, and gates the danger action. The
action runs a staged response (take cover / strafe / assault toward danger / search) via `state_mgr`.
Fires `npc_on_eval_danger` for override.

### React to hits — `xr_hit.script` *(agent-traced)*
A hit callback (subscribed via `subscribe_action_for_events`) records `st.hit.{who, bone_index,
deadly_hit}` and calls `xr_logic.try_switch_to_another_section` to evaluate the `on_hit` condlist — so a
logic section can switch when the NPC is shot. `deadly_hit` is set when damage ≥ health.

### Special styles (deltas — agent-traced)
- **`xr_combat_zombied`** — slow, no tactical cover: walk straight to the enemy's last position, fire in
  raid/threat states, long hit-reaction pauses.
- **`xr_combat_camper`** — fire from a fixed cover position (hide_fire / hide), look around when blind,
  forget after ~30 s.
- **`xr_combat_monolith`** — two distance tiers (close = hide_fire from cover, far = assault + pathfind to
  the enemy), look-around when blind.

## Function digests (spine)

- **`xr_combat.script:set_combat_type`** — *Does:* condlist → `script_combat_type` (`"nil"`→nil). *Writes:*
  `storage.script_combat_type`. *Called by:* `xr_motivator:update`. *(verified.)*
- **`xr_combat.script:evaluator_check_combat:evaluate`** — *Does:* `script_combat_type ~= nil` gate.
  *(verified.)*
- **`xr_combat.script:add_to_binder`** — *Does:* gate `action_combat_planner` + bind special styles.
  *(verified.)*
- **`xr_combat_ignore.script:is_enemy` / `reset_generic_scheme`** — *Does:* `set_enemy_callback` filter +
  the override/distance rules. *(agent-traced.)*
- **`xr_danger.script:evaluator_danger` / `action_danger`** — *Does:* detect + respond to danger.
  *(agent-traced.)*
- **`xr_hit.script:action_process_hit:hit_callback`** — *Does:* record hit + `try_switch` on `on_hit`.
  *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `storage.script_combat_type`, `st.overrides.{combat_ignore,no_combat_job,enemy_ignore}`
  (F68), `st.hit`, the danger inertia cache; `xr_combat_ignore.fighting_with_actor_npcs`.
- **Engine seams:** the GOAP **combat planner** (`stalker_ids.action_combat_planner`, gated here);
  `npc:set_enemy_callback` (engine asks Lua whether X is an enemy); `best_danger`/`best_enemy`
  (perception, `flows/ai-perception.md`); the hit callback (`flows/damage-hit.md`).
- **Driven by:** F66 — `xr_motivator:update` calls `set_combat_type` when `st.combat`/`overrides.on_combat`
  is set; the schemes are generic schemes enabled by `modules`.
- **Condlist seam:** `combat_type`/`combat_ignore_cond`/`on_hit`/`on_combat` are condlists (F67/F68).
- **Faction seam:** `xr_bribe.at_peace`, `game_relations` (who is hostile).

## Threading / failure / edge cases

- **`combat_ignore` is the usual culprit:** an NPC that won't fight (or ignores the actor) is almost
  always `is_enemy` returning false — check `combat_ignore_cond`/`no_combat_job`, distance rules, and
  safe zones before suspecting the engine.
- **`"nil"` vs nil combat type:** the literal string `"nil"` means "use engine combat," not "no combat" —
  a common authoring confusion.
- **Special styles bypass engine tactics:** zombied/camper/monolith deliberately drop the engine's cover
  AI; expect simpler, more predictable movement when one is set.
- **Combat is intricate:** the `is_enemy` distance/weather/surge rules and the danger staging are
  agent-traced — verify the specific branch before relying on exact thresholds.

## Deeper

- Combat core: `unpacked/scripts/xr_combat.script` (`set_combat_type`, `evaluator_check_combat`,
  `add_to_binder`)
- Enemy filter: `unpacked/scripts/xr_combat_ignore.script` (`is_enemy`, `ignore_enemy_by_overrides`,
  `reset_generic_scheme`); danger: `unpacked/scripts/xr_danger.script`; hits: `unpacked/scripts/xr_hit.script`
- Styles: `xr_combat_zombied.script`, `xr_combat_camper.script`, `xr_combat_monolith.script`
- Engine combat (C++): `flows/agent-squad-combat.md`; perception: `flows/ai-perception.md`; framework:
  `flows/scheme-engine-core.md`

## Maintenance

- The verified core is the gate (`set_combat_type` + `evaluator_check_combat` + the planner grab); re-read
  if combat-style selection changes.
- Promote `xr_combat_ignore`/`xr_danger`/`xr_hit` + the styles to verified when a task needs their exact
  rules (each is its own read).
- Check `gamedata/scripts/` for overrides before trusting (none for these seven at writing).
