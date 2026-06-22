---
domain: flows/world-object-binders
tags: [modlayer, gameplay, unpacked, lua, lifecycle, physics, flow]
sources: [unpacked/scripts/bind_physic_object.script, unpacked/scripts/bind_campfire.script, unpacked/scripts/bind_camp.script, unpacked/scripts/bind_faction.script, unpacked/scripts/bind_container.script, unpacked/scripts/bind_dynamic_light.script, unpacked/scripts/bind_dynamo_hand.script, unpacked/scripts/bind_door_labx8.script, unpacked/scripts/bind_awr.script]
related: [flows/binder-architecture.md, flows/scheme-engine-core.md, flows/db-registry.md, flows/physics-objects-ragdoll.md, flows/scheme-gather-corpse.md, flows/save-load.md, flows/smart-terrain-core.md, modlayer/gameplay/binders.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: world-object binders (F89)

The `bind_*` binders for **non-NPC world objects** — physics props/boxes, campfires, NPC camps, factions,
containers, dynamic lights, hand-crank dynamos, scripted doors. Each is an `object_binder` subclass on the
F88 pattern (`flows/binder-architecture.md`), attached via a `bind`/`init` hook the engine calls. They split
into **two sub-patterns**: *scheme-running* binders that drive `xr_logic` schemes on the object (the `ph_*`
catalog, F93, runs on these), and *manager/registry* binders that wrap an engine sub-object or a Lua manager
and tick it. This flow documents the pattern + each binder's distinctive behavior; the lifecycle contract is
F88.

**Provenance:** **verified** — `bind_physic_object.script` (`generic_physics_binder`), `bind_camp.script`,
`bind_faction.script` read in full; `bind_campfire.script` binder section read; the rest
(`bind_container`/`bind_dynamic_light`/`bind_dynamo_hand`/`bind_door_labx8`/`bind_awr`) **structurally
confirmed** (class decl = `object_binder` subclass + `bind`/`init` attach entry verified by grep; bodies
*(agent-traced)*). **No `gamedata/` override** of any (verified by listing). Cross-refs `xr_box.get_box_manager`,
`sr_camp.CCampManager` cited from the code read.

## The two sub-patterns (verified)

- **Scheme-running** (`generic_physics_binder`, `container_binder`): `update` runs
  `xr_logic.initialize_obj(obj, st, loaded, actor, modules.stype_item)` once then
  `xr_logic.issue_event(... "update")` (F66); hit/death/use engine callbacks forward into the scheme via
  `issue_event` + fire a global `SendScriptCallback`. `save`/`load` persist scheme state via
  `xr_logic.save_obj`/`load_obj`. **This is the object equivalent of `motivator_binder`** (F66) — the `ph_*`
  schemes (F93) plug in here.
- **Manager/registry** (`campfire_binder`, `camp_binder`, `faction_binder`, `generic_light_binder`):
  `net_spawn` wraps an engine sub-object (`obj:get_campfire()`, `alife_object(id)`) or news a Lua manager
  (`sr_camp.CCampManager`) into a module registry; `update` ticks it; `net_destroy` clears the registry slot.

## The binders (verified where noted)

### `generic_physics_binder` — `bind_physic_object.script` (verified, the canonical one)
Conditionally attached by `init(obj)` (only if `IsInvbox`, the object has a `logic` section, or it's a known
box visual). `reinit` creates `db.storage[id]`; `net_spawn` → `db.add_obj`; `update` →
`xr_logic.initialize_obj`(stype_item) then `issue_event("update")` + (re)set hit/death/use callbacks +
`xr_sound.update`; `net_destroy` → scheme `net_destroy` event + `on_offline_condlist` + `db.del_obj`. The
**callbacks** forward to the scheme (`ph_on_hit`/active scheme) and fire global `physic_object_on_*`
callbacks. **`death_callback` is the box-loot path:** a `drop_box` spawn-ini or a box visual →
`xr_box.get_box_manager():spawn_items` + the `boxes_smashed` statistic (F80 loot). `save`/`load` via
`xr_logic.save_obj`/`load_obj` (F15/F66).

### `campfire_binder` — `bind_campfire.script` (verified)
Wraps the engine `obj:get_campfire()`. `net_spawn` registers into `campfires_all[id]` and
`db.campfire_table_by_smart_names[smart]` (so a camp/smart can light its fires), and **restores on/off state**
from `alife_storage_manager` `campfire_states` (load keeps the fire as saved; fresh spawn turns it off). The
player use-toggle is commented out here (handled by `axr_turn_on_campfire`, F113). Registry + state binder, no
scheme.

### `camp_binder` — `bind_camp.script` (verified)
The ambient-camp director: `net_spawn` news `sr_camp.CCampManager(obj)` into `camps[id]`; `update` ticks it
(NPC storytelling / guitar / harmonica logic around a camp); `net_destroy` clears the slot. Manager-pattern.

### `faction_binder` — `bind_faction.script` (verified)
Tiny: `net_spawn` grabs the server faction object (`alife_object(id)`); `update` ticks `self.faction:update()`
(faction/community state). No blackboard, no scheme.

### `container_binder` / `generic_light_binder` / `dynamo_hand_binder` / `door_binder_labx8` / `bind_awr` *(structurally confirmed; bodies agent-traced)*
All are `object_binder` subclasses with a `bind`/`init` attach (verified). `container_binder` is a
scheme-running lootable container (like `generic_physics_binder`); `generic_light_binder` (`bind_dynamic_light`)
drives a dynamic light source; `dynamo_hand_binder` is a hand-crank charger; `door_binder_labx8` is a scripted
lab door; `bind_awr` (501 lines) is a larger special binder. Read the file directly using the F88 contract for
exact behavior.

## Function digests (spine)

- **`init`/`bind`** — *Does:* `obj:bind_object(<binder>(obj))`; `bind_physic_object.init` is *conditional*
  (invbox / logic section / box visual).
- **`generic_physics_binder:update`** — *Does:* `xr_logic.initialize_obj`(stype_item) + `issue_event("update")`
  + (re)register hit/death/use callbacks. The scheme tick for objects.
- **`generic_physics_binder:death_callback`** — *Does:* box loot (`xr_box…spawn_items` + `boxes_smashed`).
- **`campfire_binder:net_spawn`** — *Does:* register `campfires_all` + per-smart table; restore on/off state.
- **`camp_binder:update`** — *Does:* tick the `sr_camp.CCampManager` for `camps[id]`.
- **`faction_binder:update`** — *Does:* tick the server faction object.

## Isolation ledger (the seams)

- **Binder base (F88):** all subclass `object_binder`; methods chain `object_binder.X(self)`; attached by the
  C++ client object's `bind`/`init` hook.
- **Scheme seam (F66/F68/F93):** scheme-running binders call `xr_logic.initialize_obj`/`issue_event`/
  `save_obj`/`load_obj`; `ph_*` schemes (F93) run on `generic_physics_binder`.
- **Blackboard seam (F69):** `db.storage[id]` (scheme binders); `db.add_obj`/`del_obj` register the live object.
- **Registry seams:** `campfires_all` + `db.campfire_table_by_smart_names` (campfires), `camps` (camp managers),
  `sr_camp.CCampManager` is the camp logic.
- **Loot seam (F80):** `generic_physics_binder:death_callback` → `xr_box.get_box_manager():spawn_items`.
- **Persistence seam (F15):** `save`/`load` via `xr_logic.save_obj`/`load_obj` + `set_save_marker`; campfire
  on/off via `alife_storage_manager` `campfire_states`.
- **Callback bus (F1):** `physic_object_on_use_callback`/`_on_hit_callback`.

## Threading / failure / edge cases

- **`generic_physics_binder` re-registers callbacks every `update`:** hit/death/use are set each frame (not
  just in `reinit`) — robust to engine callback resets but means you can't permanently nil them from outside.
- **Conditional attach:** a physics object with no `logic` section and a non-box visual gets **no binder** —
  only boxes, invboxes, and logic-driven objects are bound. A "dead" prop with no behavior simply isn't bound.
- **Campfire state restore:** on load a saved campfire stays as-saved (the `campfire_states[id]` entry is
  consumed); a fresh spawn defaults to off. Lighting is driven externally (camps / `axr_turn_on_campfire`).
- **Box loot is probabilistic for plain boxes:** a `drop_box` ini always spawns; a bare box visual spawns loot
  ~50% of the time (verified in `death_callback`).
- **`camp`/`faction` binders have no blackboard:** they don't create `db.storage` — they're pure manager/registry
  binders, so scheme tooling won't find state for them.

## Deeper

- Canonical object binder: `unpacked/scripts/bind_physic_object.script` (`generic_physics_binder`)
- Campfire/camp/faction: `unpacked/scripts/bind_campfire.script` (`campfires_all`,
  `db.campfire_table_by_smart_names`), `bind_camp.script` (`camps`, `sr_camp.CCampManager`), `bind_faction.script`
- Others: `bind_container.script`, `bind_dynamic_light.script`, `bind_dynamo_hand.script`,
  `bind_door_labx8.script`, `bind_awr.script`
- Pattern + lifecycle: `flows/binder-architecture.md` (F88); schemes on objects: `flows/scheme-engine-core.md`
  (F66), `ph_*` (F93); physics: `flows/physics-objects-ragdoll.md` (F28); box loot: `flows/scheme-gather-corpse.md` (F80)
- Map: `modlayer/gameplay/binders.md` (M7)

## Maintenance

- Verified core (`generic_physics_binder` scheme + loot, campfire/camp/faction registries, the two
  sub-patterns) is stable. Promote `container`/`dynamic_light`/`dynamo_hand`/`door`/`awr` bodies from
  agent-traced when a task needs exact behavior.
- `generic_physics_binder` is the binder the `ph_*` schemes (F93) attach to — cross-link when F93 lands.
- No `gamedata/` override at writing — re-check per session.
