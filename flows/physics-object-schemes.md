---
domain: flows/physics-object-schemes
tags: [modlayer, gameplay, unpacked, lua, schemes, physics, flow]
sources: [unpacked/scripts/ph_door.script, unpacked/scripts/ph_button.script, unpacked/scripts/ph_code.script, unpacked/scripts/ph_idle.script, unpacked/scripts/ph_hit.script, unpacked/scripts/ph_on_hit.script, unpacked/scripts/ph_death.script, unpacked/scripts/ph_oscillate.script, unpacked/scripts/ph_sound.script, unpacked/scripts/ph_appforce.script, unpacked/scripts/ph_car.script, unpacked/scripts/ph_minigun.script, unpacked/scripts/phantom_manager.script, unpacked/configs/creatures/m_phantom.ltx]
related: [flows/scheme-engine-core.md, flows/logic-ltx-switching.md, flows/world-object-binders.md, flows/binder-architecture.md, flows/physics-objects-ragdoll.md, flows/vehicles-holder.md, flows/special-actors.md, flows/restrictors-passive-sr.md, flows/sim-board.md, flows/scheme-combat.md, modlayer/gameplay/physics-schemes-ph.md, modlayer/gameplay/binders.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: physics-object schemes (`ph_*`) (F93)

The **`ph_*` schemes** — logic schemes for physics props: doors, buttons, code pads, one-shot impulses,
oscillators, sound sources, scripted cars and mounted guns. Same F66 template
(`set_scheme`/`add_to_binder`/`action`) as `sr_*`/`xr_*`, but registered as **`stype_item`** and run by
the `generic_physics_binder` (F89) — the binder's `update` runs `xr_logic.initialize_obj(…, stype_item)`
and `issue_event` delivers the engine `use`/`hit`/`death` callbacks to the subscribed scheme action. Plus
`phantom_manager`, which is **not a scheme** (a binder + spawn service for psy phantoms).

**Provenance:** **verified, no agent** — all 10 small scripts + `ph_door` READ-IN-FULL this session;
`ph_car`/`ph_minigun` spine read (`__init`/`reset_scheme`/`fastcall`/`update`/`destroy_car`/`set_scheme`);
their movement/fire interiors (`go_to_walkpoint`, fire-point selection, `fast_update` bodies) **not read**.
**No `gamedata/` override of any** (verified by listing).

## Registration (verified in `modules.script`)

`LoadScheme(filename, scheme, stype)` rows, all `stype_item`: `ph_car`, `ph_door`, `ph_idle`, `ph_hit`,
`ph_on_hit`, `ph_button`, `ph_code`, `ph_minigun`, `ph_oscillate`, `ph_sound`, and **`ph_death` registering
scheme name `ph_on_death`** (file≠scheme name). **`ph_appforce` is NOT registered** — `ph_force` (a
delayed `set_const_force` push toward a patrol point) is dead code in the base corpus; a mod must
`LoadScheme` it itself. Base-config usage (grep `= ph_X@` in `unpacked/configs/`): door 235, idle 58,
button 21, code 5, sound 2; car/minigun/oscillate/hit/on_death **0** (loaded, story-mod surface).

## The fastcall channel (verified)

`obj:set_fastcall(fn, self)` → `CScriptGameObject::set_fastcall` (`script_game_object_use.cpp`) — an
engine **per-frame high-rate callback** (vs the throttled scheme `update`); return true to deregister.
`ph_door` uses it to detect open/close completion; `ph_car`/`ph_minigun` run their whole drive/fire logic
in it (self-throttled via `fc_upd_num`/`fc_upd_avg`), returning true when the scheme goes inactive or the
vehicle dies.

## The schemes

### `ph_door` *(verified, full)* — joint-physics door
`action_door` drives the physics joint named **"door"** (`get_physics_shell():get_joint_by_bone_name`):
open = `set_max_force_and_velocity(2100,-3)` + `release_fixed`, close = `(200,3)` + `fix` on arrival;
open/closed detected by joint axis angle vs limits (`slider` flips to axis 0); completion polled on the
fastcall channel → `on_door_is_open/closed` + `set/unset_door_ignore_dynamics`. `use_callback` → `on_use`
condlist switch; `hit_on_bone` → per-bone section switch; `locked`/`no_force`/tips. `add_to_binder` also
`register_door_for_npc()` + records `db.level_doors[id]` — **module fns `try_to_open_door`/
`try_to_close_door` are called from `xr_motivator:update` (F66): NPCs auto-open scripted doors within
√3 m unless `not_for_npc`** (default from a hardcoded `not_for_npcs` visual blacklist — cupboards,
fridges, safes); `locked`/`not_for_npc` ↔ engine `lock/unlock_door_for_npc` AI gating.

### `ph_button` *(verified, full)* — pressable switch
Plays `anim` via `play_cycle`; `use_callback` → if active scheme is `ph_button`, `on_press` condlist →
`switch_to_section`. The hit path exists but is commented out. `tooltip` sets the use prompt.

### `ph_code` *(verified, full)* — code pad
`codepad:use_callback` → `ui_numpad.start(self)` (UI dialog); `OnNumberReceive`: exact `code` match →
run `on_code` condlist, or (no `code` set) look the entered string up in `on_check_code<N>` →
per-code condlist. Effects fire via `pick_section_from_condlist` (which also runs `%…%` effects, F67).

### `ph_idle` *(verified, full)* — interaction holder
"Do nothing" parking section: sets `tips` + `nonscript_usable`; `on_use` → condlist switch;
`hit_on_bone` → per-bone switch. The default state for interactive props between events.

### `ph_hit` *(verified, full)* — one-shot impulse
`action_hit:reset_scheme` applies a single `hit.strike` **to its own object** (`power`/`impulse`/`bone`,
direction = `dir_path` patrol point 0 minus own position) — knock something over on section entry.

### `ph_on_hit` *(verified, full)* — generic hit listener
The only `ph_*` **generic** scheme: `setup_generic_scheme` (called by `modules.enable_generic_schemes`
for every stype_item object) gates on an `on_hit` field in the logic section, binds, and `hit_callback`
→ `try_switch_to_another_section`. **Latent dead path:** `enable_generic_schemes`' stype_item tail calls
`ph_on_hit.set_scheme(…)`, which **does not exist** in the file (would nil-crash if reached) — 0 base
configs trigger it; `on_hit` is also not a `cfg_get_switch_conditions` trigger (verified by grep).

### `ph_on_death` (`ph_death.script`) *(verified, full)* — death listener
`death_callback` → `try_switch_to_another_section`. Pair with `on_death`-style switch logic in the
section; the box-loot death path is separate (binder-level, F89).

### `ph_oscillate` *(verified, full)* — joint oscillator
Periodic `set_const_force` on the joint named by `joint` (gulag-prefixed): force ramps at
`force/period`, direction flips each period rotated by `correct_angle` (`vector_rotate_y`), half-period
pause between swings. Aborts on a missing joint. Swinging lamps/signs.

### `ph_sound` *(verified, full)* — positional sound source
`snd_source` — **the body is hijacked to the Zone FM radio**: plays
`ui_pda_radio_tab.get_playing_object(1)` at the object, gated by the `sound/radio/zone` option; the
original `xr_sound` theme code (`snd`/`looped`/`random` config still parsed) is commented out. A hit
destroys the source (`no_hit=true` default protects it) and `destructed` persists via
`save_var`/`load_var`; `min/max_idle` pause between plays; `volume` is a condlist.

### `ph_car` *(spine verified; movement/fire interiors not read)* — scripted car
`action_car` wraps the engine `CCar` (`obj:get_car()`, F29). Two modes by config: **usable mode**
(`usable` condlist present) = a player-enterable car — fastcall just evaluates the condlist into
`set_nonscript_usable` + use/locked tips; **scripted mode** = AI driving along `path_walk`
(nearest-point resume, `walk_arrival_callback`, speed persisted via `save_var`) with an optional mounted
weapon (`HasWeapon` → `CCar.eWpnActivate`; fire at `points` (`path_fire`)/`actor`/story-id target with
`on_target_vis`/`on_target_nvis` condlists). Headlights are driven through the **script-entity action
queue** (`xr_logic.mob_capture` + `action(object("left_light", activate), …)`, F37). `invulnerable`
restores health each update; `GetfHealth()<=0` on fastcall → `destroy_car` (stop, lights off,
`SendScriptCallback("vehicle_on_death_callback")`, `mob_release`, `on_death_info` info portion) →
`update` switches the section to `nil`.

### `ph_minigun` *(spine verified; fire interiors not read)* — mounted gun
`action_mgun` — a stationary turret that is **also an engine `CCar`** (`get_car()`). Fire-target modes:
`points` (`path_fire`), `actor`, a story-id object, or **`job`** (Alundaio): the turret shoots the
current best enemy of the NPC holding a named smart-terrain job — `smart`+`job` config resolved via
`SIMBOARD.smarts_by_names` + `smart.npc_by_job_section` (F86/F73), target = `db.storage[npc].enemy_id`
filtered by `xr_combat_ignore.ignore_enemy_by_overrides` (F76) + `IsWounded`, vetoed by the NPC's
`rx_dont_shoot`. `on_target_vis`/`on_target_nvis` condlists, `fire_angle`/`fire_range`/`track_target`;
death like the car. The NPC *manning* the turret is a separate job scheme — this object IS the gun.

### `phantom_manager` *(verified, full)* — NOT a scheme
Two systems, both bound via `m_phantom.ltx` `script_binding` (not `class_registrator`): **legacy** —
`PhantomManager` counter + `level.spawn_phantom(pos)` spawning the engine `CPhantom` apparition (F36),
`Phantom(object_binder)` counts them; consumed by `sr_psy_antenna` (F91, probability + `phantom_max`).
**DynamicPhantom** (Alundaio) — real spawned objects: a `monster_on_update` callback (5 s throttle per
monster) spawns the monster's `spawn_phantom` section via `level.spawn_item` when the psy-antenna is
active (or `can_spawn_phantom`), the monster is fighting the actor (`fighting_with_actor_npcs`, F76),
within 30 m, ≤2 per section (`phantom_registry`, decremented in `net_destroy`).

## Isolation ledger (the seams)

- **Scheme engine (F66/F68):** all register via `set_scheme`(`assign_storage_and_bind` +
  `cfg_get_switch_conditions`)/`add_to_binder`(`subscribe_action_for_events`); `update` →
  `try_switch_to_another_section`. `ph_on_hit` is generic (`setup_generic_scheme`).
- **Binder (F89/F88):** run on `generic_physics_binder` objects (`initialize_obj` stype_item;
  `issue_event` routes use/hit/death callbacks).
- **Physics (F28):** `get_physics_shell`/`get_joint_by_bone_name`/`set_max_force_and_velocity`/
  `fix`/`release_fixed`/`set_const_force`/`get_physics_object` door-dynamics toggles.
- **Vehicles (F29):** `obj:get_car()` → engine `CCar` (car AND minigun); `CCar.eWpnActivate` etc.
- **Script-entity queue (F37):** car headlights via `mob_capture` + `action(object(…))` commands.
- **Engine fastcall (F42):** `set_fastcall` per-frame channel (`script_game_object_use.cpp`).
- **NPC door seam (F66):** `xr_motivator:update` → `ph_door.try_to_open_door/try_to_close_door` over
  `db.level_doors`; engine `lock/unlock_door_for_npc`.
- **Sim board (F86) + combat-ignore (F76):** minigun `job` targeting; phantom combat gate.
- **Phantoms (F36/F91):** `level.spawn_phantom` (CPhantom) / `level.spawn_item` + `sr_psy_antenna`.
- **UI:** `ph_code` → `ui_numpad`; `ph_sound` → `ui_pda_radio_tab` + the `sound/radio/zone` option.

## Threading / failure / edge cases

- **`ph_appforce` is unregistered** — referencing `ph_force@x` in logic fails section resolution; it is
  not a usable scheme without a mod-side `LoadScheme`.
- **`ph_door` needs a physics shell with a "door" joint:** `reset_scheme` returns early without one and
  `update` then logs "door failed to initialize" every tick — the classic broken-door symptom.
- **Doors and NPCs:** NPCs only operate doors with `not_for_npc=false` (cupboard-type visuals are
  blacklisted by default) — an NPC stuck at a door usually means `locked`/`not_for_npc`/engine door lock.
- **`ph_on_hit` legacy path is broken:** anything that reaches `ph_on_hit.set_scheme` nil-crashes;
  the working path is the generic auto-attach via an `on_hit` field in the item's logic section.
- **`ph_sound` is radio-only now:** the `snd` theme config is parsed but unused (body rewired to Zone FM);
  a destroyed source stays silent across saves (`destr` pstor var).
- **Car/minigun die on the fastcall:** health ≤0 → `destroy_car` → next `update` switches to section
  `nil` — logic after death must hang off `on_death_info`/`vehicle_on_death_callback`, not the scheme.
- **Minigun `job` mode needs both `smart` and `job`** (else the smart ref is dropped) and the job key is
  prefixed `logic@` internally.
- **No `gamedata/` override** of any `ph_*` file at writing — re-check per session.

## Deeper

- Doors/props: `unpacked/scripts/ph_door.script` (`action_door`, `try_to_open_door`),
  `ph_button.script`, `ph_code.script` (`codepad`), `ph_idle.script` (`action_idle`)
- Impulses: `ph_hit.script` (`action_hit`), `ph_oscillate.script` (`action_oscillator`),
  `ph_appforce.script` (`ph_force`, unregistered)
- Listeners: `ph_on_hit.script`, `ph_death.script` (`ph_on_death`); registration:
  `modules.script` (`LoadScheme`/`enable_generic_schemes`)
- Vehicles/turrets: `ph_car.script` (`action_car`), `ph_minigun.script` (`action_mgun`,
  `find_valid_target`)
- Phantoms: `phantom_manager.script` (`DynamicPhantom`/`monster_on_update`/`PhantomManager`),
  `unpacked/configs/creatures/m_phantom.ltx` (`script_binding`)
- Engine: `flows/physics-objects-ragdoll.md` (F28), `flows/vehicles-holder.md` (F29),
  `flows/special-actors.md` (F36, CPhantom); map: `modlayer/gameplay/physics-schemes-ph.md` (M11)

## Maintenance

- Promote `ph_car`/`ph_minigun` movement/fire interiors (path driving, fire-point selection,
  `fast_update`) from not-read when a task needs them; everything else is read-in-full and stable.
- Re-check `gamedata/scripts/ph_*` for overrides each session (none at writing).
- If a mod registers `ph_appforce` or fixes the `ph_on_hit.set_scheme` call, update the dead-path notes.
