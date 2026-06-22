---
domain: flows/scheme-engine-core
tags: [modlayer, gameplay, unpacked, lua, xr_logic, schemes, ai, flow]
sources: [unpacked/scripts/xr_logic.script, unpacked/scripts/xr_motivator.script, unpacked/scripts/xr_gulag.script, unpacked/scripts/modules.script, unpacked/scripts/_g.script]
related: [flows/ai-squad-logic.md, flows/script-npc-actions.md, flows/db-registry.md, flows/gameplay-boot.md, flows/condlist-library.md, flows/logic-ltx-switching.md, flows/smart-terrain-core.md, flows/utils-libraries.md, modlayer/gameplay/scheme-engine.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: scheme engine core — xr_logic / xr_motivator (F66)

The framework that drives all scripted NPC behavior ("logic"). Every NPC carries a **motivator binder**
that ticks each frame; the binder consults **xr_logic**, which holds the active *scheme* (walker, camper,
sleeper, …) and switches between logic *sections* when condlist triggers fire. Schemes register
themselves at load (`LoadScheme`, see `flows/gameplay-boot.md`) and store their per-NPC state in
`db.storage[id]` (`flows/db-registry.md`). This is the heart of the gameplay corpus; the scheme catalog
(roadmap F74–F82), smart_terrain/gulag jobs (F72/F73), condlists (F67), and logic-LTX (F68) all attach
here.

**Provenance:** spine read and verified in `xr_logic.script` (`activate_by_section`, `switch_to_section`,
`try_switch_to_another_section`, `issue_event`, `assign_storage_and_bind`, the head of
`pick_section_from_condlist`) and `xr_motivator.script` (`motivator_binder:update`). `configure_schemes`,
`initialize_obj`, `parse_condlist`, `cfg_get_overrides`, and `modules.enable/reset_generic_schemes`
exist (names verified) but their bodies are *(agent-traced — not personally re-read)*. The
condlist-evaluation tail and `xr_conditions`/`xr_effects` are a deliberate seam → `flows/condlist-library.md`.

## Concepts (verified)

- **scheme** — a named behavior (e.g. `walker`). Registered via `_g.LoadScheme(file, scheme, ...stypes)`
  into globals `schemes[scheme]=file` and `schemes_by_stype[stype][scheme]=true`.
- **section** — a config block in the NPC's logic LTX (e.g. `walker@guard`); its scheme is the prefix
  before `@`. `utils_data.get_scheme_by_section` extracts it.
- **active scheme/section** — stored on `db.storage[id].active_scheme`/`active_section`.
- **st (state)** — `db.storage[id]` and its per-scheme subtable `db.storage[id][scheme]` (holds the
  scheme's `actions`, `signals`, `logic` switch-list, etc.).

## Sequence (verified spine)

### Per-frame tick — `xr_motivator.script:motivator_binder:update(delta)`
Verified order: `object_binder.update` → `xr_sound.update(id)` → **dead path** (if `not alive`:
`death_manager.decide_items_to_keep` once, then `try_switch_to_another_section` on the active scheme, and
return) → `xr_meet.process_npc_usability` → `sr_light.check_light` → clear stale combat-ignore →
**squad commander** (`self.squad:update()` if this NPC commands its squad) → `state_mgr:update()` (if
present; `flows/state-manager-core.md`, F70) → `SendScriptCallback("npc_on_update", object, db.storage[id])`
→ `stalker_generic.update_invulnerability` → **500 ms throttle** (`self.__tmr = tg+500`; below runs at most
twice/sec) → `ph_door.try_to_open_door/close_door` → **logic/combat block**: if an active scheme exists,
handle combat overrides (`overrides.on_combat` condlist + `xr_combat.set_combat_type`) else
`try_switch_to_another_section(object, st[active_scheme], db.actor)`.

### Activate a section — `xr_logic.script:activate_by_section(npc, ini, section, gulag_name, loading)`
Verified: stamps `activation_time`/`activation_game_time` (clamped on load); `section=="nil"` → clear
overrides + active scheme/section and return; `section==nil` → resolve from the NPC's current gulag job;
`scheme = utils_data.get_scheme_by_section(section)`; guard that the scheme is allowed for this stype
(`schemes_by_stype[stype][scheme]`); `cfg_get_overrides`; `reset_generic_schemes_on_scheme_switch`;
`filename = schemes[scheme]` then `_G[filename].set_scheme(npc, ini, scheme, section, gulag_name)`; set
`active_section`/`active_scheme`; for stalkers, reset patrol/path then
`issue_event(..., "activate_scheme", loading, npc)` (non-stalkers get `"reset_scheme"`).

### Switch sections — `xr_logic.script:switch_to_section(npc, ini, section)`
Verified: no-op on empty/nil or same-as-active; refresh `overrides`; if a scheme is active,
`issue_event(..., "deactivate", ...)` on the old; clear active; then `activate_by_section(..., false)`.

### Decide a switch — `xr_logic.script:try_switch_to_another_section(npc, st, actor)`
Verified: iterates `st.logic` (the parsed switch-condition list) and, per entry `c.name`, tests the
trigger and on match calls `switch_to_section(npc, st.ini, pick_section_from_condlist(actor, npc, c.condlist))`.
Verified trigger taxonomy: `on_actor_dist_le[_nvis]`, `on_actor_dist_ge[_nvis]`, `on_signal`, `on_info`,
`on_timer`, `on_game_timer`, `on_actor_in_zone`/`on_actor_not_in_zone`, `on_npc_in_zone`/`not_in_zone`,
`on_actor_inside`, … (visibility-gated variants use `see_actor`; timers compare against `activation_time`).

### Dispatch to the active action — `xr_logic.script:issue_event(npc, st, event_fn, ...)`
Verified: for each `(action_ptr, is_active)` in `st.actions`, if active and it has `event_fn`, call
`action_ptr[event_fn](action_ptr, ...)`. This is how `activate_scheme`/`deactivate`/`execute`/`save`/etc.
reach the scheme's action object. `check_action` is the same pattern but returns the first result.

### Bind a scheme's action — `xr_logic.script:assign_storage_and_bind(npc, ini, scheme, section, temp)`
Verified: lazily creates `db.storage[id][scheme] = {npc=npc}` then calls
`_G[schemes[scheme]].add_to_binder(npc, ini, scheme, section, storage, temp)` — the scheme's own
`add_to_binder` creates its action/evaluator objects and registers them on the NPC's
`motivation_action_manager`. (Per-scheme `set_scheme`/`add_to_binder` bodies = the scheme catalog, F74+.)

### Setup & switch-list parse *(agent-traced)*
`configure_schemes` parses the `[logic]` section + `custom_data`, disables old then enables generic
schemes (`modules.enable_generic_schemes`), and stores `ini`/`section_logic`/`stype`/`gulag_name` on
`db.storage[id]`. `initialize_obj` is the non-gulag entry that runs configure → determine →
`activate_by_section`. The switch-list (`st.logic`) is built by `cfg_get_switch_conditions` +
`parse_condlist` (`flows/logic-ltx-switching.md`, F68).

## Function digests (spine)

- **`xr_motivator.script:motivator_binder:update`** — *Does:* the verified per-frame tick above.
  *Reads/Writes:* `self.st`(=`db.storage[id]`), `__tmr`. *Calls:* `try_switch_to_another_section`,
  `state_mgr:update`, `xr_combat.set_combat_type`, `ph_door.*`. *Fires:* `npc_on_update`. *(verified.)*
- **`xr_logic.script:activate_by_section`** — *Does:* make a section's scheme active (or clear). *Writes:*
  `active_section/scheme`, `overrides`, activation times. *Calls:* `set_scheme`, `issue_event`. *(verified.)*
- **`xr_logic.script:switch_to_section`** — *Does:* deactivate old + activate new on change. *(verified.)*
- **`xr_logic.script:try_switch_to_another_section`** — *Does:* evaluate `st.logic` triggers → switch.
  *Calls:* `pick_section_from_condlist`, `switch_to_section`. *(verified.)*
- **`xr_logic.script:issue_event`** — *Does:* dispatch an event fn to active actions in `st.actions`.
  *(verified.)*
- **`xr_logic.script:assign_storage_and_bind`** — *Does:* create per-scheme storage + call the scheme's
  `add_to_binder`. *(verified.)*
- **`xr_logic.script:pick_section_from_condlist`** — *Does:* evaluate a condlist (infoportion checks +
  `xr_conditions`/`xr_effects`) and return the chosen section. *(head verified; eval tail →
  `flows/condlist-library.md`.)*
- **`xr_logic.script:configure_schemes`/`initialize_obj`** — *Does:* parse logic + set up the active
  section. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `db.storage[id]` (logic state) + `db.storage[id][scheme]` (per-scheme: `actions`,
  `signals`, `logic`, `overrides`). The engine globals `schemes`/`schemes_by_stype` (from boot).
- **Driver (in):** the engine ticks the binder; `motivator_binder:update` is the per-frame entry.
- **Scheme seam (out):** `set_scheme`/`add_to_binder`/the action object per scheme — the catalog (F74+);
  `flows/script-npc-actions.md` is the C++ action/evaluator side.
- **Condlist seam:** `pick_section_from_condlist` → `xr_conditions[]`/`xr_effects[]`
  (`flows/condlist-library.md`, F67).
- **Job seam:** sections/LTX come from smart_terrain gulag jobs (`flows/smart-terrain-core.md` F72,
  `flows/ai-squad-logic.md` F12) or directly via `initialize_obj`.
- **State-mgr seam:** `st.state_mgr:update()` drives animation/movement (`flows/state-manager-core.md`).
- **Engine action seam:** schemes register actions on the NPC's `motivation_action_manager` (GOAP;
  `flows/ai-planner.md`).

## Threading / failure / edge cases

- **stype guard:** activating a scheme not allowed for the NPC's stype errors out and aborts — schemes
  must be registered with the right stype in `LoadScheme`.
- **`section "nil"` vs `nil`:** the string `"nil"` clears logic; Lua `nil` triggers a gulag-job lookup.
  These are distinct paths — don't conflate.
- **500 ms throttle:** door/logic-switch work runs ≤2×/sec; per-frame-exact behavior must go above the
  `__tmr` gate (state_mgr/squad/npc_on_update run every frame, the switch block is throttled).
- **Dead NPCs still switch:** the dead path still calls `try_switch_to_another_section` (e.g. for
  on_death logic), then returns early.

## Deeper

- Engine: `unpacked/scripts/xr_logic.script` (`activate_by_section`, `switch_to_section`,
  `try_switch_to_another_section`, `issue_event`, `assign_storage_and_bind`, `pick_section_from_condlist`,
  `configure_schemes`, `initialize_obj`, `parse_condlist`, `cfg_get_overrides`)
- Binder: `unpacked/scripts/xr_motivator.script` (`motivator_binder:update`/`net_spawn`/`reinit`)
- Generic schemes: `unpacked/scripts/modules.script` (`enable_generic_schemes`, `reset_generic_schemes`)
- Registration + state: `flows/gameplay-boot.md` (`LoadScheme`), `flows/db-registry.md` (`storage`)
- Seams: `flows/condlist-library.md` (F67), `flows/logic-ltx-switching.md` (F68),
  `flows/smart-terrain-core.md` (F72), `flows/script-npc-actions.md` (C++ actions)

## Maintenance

- The verified spine (activate/switch/try_switch/issue_event/assign_storage_and_bind + motivator update)
  is the stable core; re-read those bodies if logic behavior changes.
- Promote `configure_schemes`/`initialize_obj`/`parse_condlist`/generic-scheme parts to verified when
  read (natural pickups: F67 condlists, F68 logic-LTX).
- Check `gamedata/scripts/` for overrides of `xr_logic`/`xr_motivator`/`modules` before trusting (none
  found for the core at time of writing; `xr_weapon_jam` and other schemes do override).
