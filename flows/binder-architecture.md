---
domain: flows/binder-architecture
tags: [modlayer, gameplay, unpacked, lua, lifecycle, callbacks, flow]
sources: [unpacked/scripts/bind_stalker.script, unpacked/scripts/class_registrator.script, unpacked/scripts/xr_motivator.script, gamedata/scripts/bind_monster.script, unpacked/scripts/bind_monster.script]
related: [flows/object-lifecycle.md, flows/cscript-game-object.md, flows/scheme-engine-core.md, flows/db-registry.md, flows/gameplay-boot.md, flows/save-load.md, flows/lua-callback-lifecycle.md, flows/se-stalker.md, flows/actor.md, engine/server-entities.md, modlayer/gameplay/binders.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: binder architecture (F88)

How **Lua attaches to an online engine object**. Every game object that has gameplay logic gets a Lua
*binder* — a subclass of the engine `object_binder` (the Lua face of the C++ `CScriptBinder`, F5) — whose
lifecycle methods (`reinit`/`net_spawn`/`net_destroy`/`update`/`save`/`load`) the engine calls each time the
object spawns, ticks, persists, and despawns **online**. This is the client-side counterpart to the `se_*`
server-entity tier (F83/F84): `se_*` is the offline ALife object, the binder is the online behavior harness.
Binders own the per-object `db.storage[id]` blackboard (F69), register the object's engine callbacks, and
run the scheme engine.

**Provenance:** **verified** — `bind_stalker.script` (actor_binder) spine + the attach entries read this
session; `class_registrator.script` read in full; the monster binder read from
**`gamedata/scripts/bind_monster.script` (the live override**, 528 vs 477 lines, 52 changed lines — the
`generic_object_binder` runs there). Attach mechanism verified: `bind_stalker.actor_init` →
`npc:bind_object(actor_binder(npc))`; `bind_stalker.init` → `xr_motivator.AddToMotivator` →
`npc:bind_object(motivator_binder(npc))`; `bind_monster.bind` → `obj:bind_object(generic_object_binder(obj))`.
The `object_binder` base is C++ (`engine/server-entities.md`, F5).

## Concepts (verified)

- **object_binder** — the engine Lua base class (C++ `CScriptBinder` exposed via luabind). Defines the
  virtuals the engine calls on each online object; subclasses override them and **call up** via
  `object_binder.METHOD(self, …)`. *(verified: every binder's methods open with `object_binder.<m>(self)`.)*
- **binder** — an `object_binder` subclass attached to a specific online object via `obj:bind_object(binder)`.
  The three core ones: **actor_binder** (player), **motivator_binder** (stalkers/NPCs, F66),
  **generic_object_binder** (monsters). The many `bind_*` files (F89–F93) are object-specific binders.
- **attach entry** — a module function (`init`/`actor_init`/`bind`) the **engine C++ client object calls**
  during construction; it news a binder and `bind_object`s it. *(verified: `actor_init`/`init`/`bind` bodies.)*
- **class_registrator** — maps each `clsid`/`script_clsid` to its **client (C++) class + server (Lua se_)
  class** via `object_factory:register`. This is the registration the object_factory uses to instantiate the
  right server object; the binder is attached by the client object's own bind hook, not here. *(verified.)*

## Sequence (the binder lifecycle — the engine calls these, all verified)

### Attach — `init` / `actor_init` / `bind`
When a client object is created the engine calls its Lua bind hook: `bind_stalker.actor_init(npc)` →
`npc:bind_object(actor_binder(npc))` (the actor); `bind_stalker.init(obj)` →
`xr_motivator.AddToMotivator(obj)` → `obj:bind_object(motivator_binder(obj))` (a stalker, F66);
`bind_monster.bind(obj)` → `obj:bind_object(generic_object_binder(obj))` (a monster). From here the engine
owns the binder and drives its lifecycle.

### `reinit()` — create the blackboard + register callbacks
`object_binder.reinit(self)` first, then `db.storage[id] = empty_table(...)`, `self.st = db.storage[id]`
(the per-object blackboard, F69), then **register the object's engine callbacks** via
`self.object:set_callback(callback.X, fn, self)`. The actor registers a large vanilla+coc set (item
take/drop, hud_animation_end, hit, key_press/release/hold, weapon_*, foot_step, …) all routed to
`bind_stalker_ext`; the monster registers `patrol_path_in_point`/`hit`/`death`/`sound`. Optionally
`RegisterScriptCallback("save_state", self)` when `USE_MARSHAL`. **`reinit` runs on every (re)load**, so it
is where per-object state + callbacks are (re)established.

### `net_spawn(se_abstract)` — object comes online
Returns false (aborting) if `object_binder.net_spawn` returns false. The actor's does the world-entry setup
(`db.add_actor`, weather reset, input enable, death-drop settings, `bCheckStart=true` for first-update) and
delegates to `bind_stalker_ext.actor_on_net_spawn`. Returns true on success.

### `update(delta)` — per-frame online tick
`object_binder.update(self, delta)` first. The **actor binder** runs a first-update boot (`bCheckStart` →
`bind_stalker_ext.actor_on_first_update`, give `global_dialogs`/`level_changer_icons` infos, `pda.fill_primary_objects`,
`activate_slot`, set `db.actor.afterFirstUpdate`) then delegates per-frame work to `bind_stalker_ext`. The
**monster binder** is the scheme tick: fires `monster_on_update`, handles the corpse-loot tip, then
`xr_logic.try_switch_to_another_section` + `xr_logic.issue_event(obj, st[active_scheme], "update", delta)`
(F66/F68) and the squad-follow movement (`mob_capture` + `move(walk/run_with_leader)`); this is the monster
equivalent of `motivator_binder:update` (F66). Dead monsters/online enemy paths short-circuit
(`xr_logic.mob_release`).

### `net_destroy()` — object goes offline
Finalize managers (actor: surge/psi/weather), stop sounds, **clear all the registered callbacks** (set each
to nil), `db.del_actor`/`del_*`, then `object_binder.net_destroy(self)`. The cleanup mirrors `reinit`.

### `save(packet)` / `load(reader)` + `save_state` / `load_state`
Net-packet persistence (`save`/`load`, F15) + the optional marshal `save_state`/`load_state(m_data)` table
persistence (`USE_MARSHAL`). Both call up to `object_binder`. The binder is where a Lua object's runtime
state is serialized alongside the engine packet (pairs the `se_*` `STATE_*`, F83/F84).

## Function digests (spine)

- **`bind_stalker.actor_init` / `init` / `bind_monster.bind`** — *Does:* `obj:bind_object(<binder>(obj))`;
  the engine-called attach entry. *Calls:* `xr_motivator.AddToMotivator` for the stalker path.
- **`object_binder` (engine base)** — *Does:* the C++ `CScriptBinder` lifecycle; subclasses chain up.
- **`actor_binder:reinit/net_spawn/net_destroy/update`** — *Does:* the player binder; owns `db.actor_binder`,
  registers the actor callback set, delegates logic to `bind_stalker_ext`.
- **`generic_object_binder:reinit/update`** — *Does:* the monster binder; `db.storage` + death/hit/sound
  callbacks; `update` = scheme tick (`xr_logic.issue_event`) + squad-follow movement. **gamedata override.**
- **`motivator_binder`** — *Does:* the stalker/NPC binder; the scheme tick (documented in F66).
- **`class_registrator.register(object_factory)`** — *Does:* `cs_register`/`c_register`/`s_register` every
  clsid → client+server class (+ the `_ALIFE_UNREGISTER` hack). *Called by:* gameplay boot (F65).

## Isolation ledger (the seams)

- **Engine base (F5/F42):** `object_binder` = C++ `CScriptBinder`; the engine calls
  `reinit`/`net_spawn`/`net_destroy`/`update`/`save`/`load` per online object (`flows/object-lifecycle.md`).
- **Attach seam:** the C++ client object calls the Lua `init`/`actor_init`/`bind` hook →
  `obj:bind_object(...)`. (Which hook a class uses is hardwired in the C++ client, F5.)
- **Blackboard seam (F69):** `reinit` creates `db.storage[id]` (`self.st`); `net_destroy` tears it down.
- **Callback seam (F1/F42):** binders register per-object engine callbacks via `object:set_callback`
  (`flows/cscript-game-object.md`); actor → `bind_stalker_ext`, monster → its own `*_callback` methods.
- **Scheme seam (F66/F68):** stalker `motivator_binder:update` + monster `generic_object_binder:update` both
  drive `xr_logic` (`flows/scheme-engine-core.md`).
- **Registration seam (F65):** `class_registrator` ties clsid → client+server class for the object_factory
  (`flows/gameplay-boot.md`); the `se_*` server side is F83/F84.
- **Persistence seam (F15):** `save`/`load` (+ marshal `save_state`) pair the `se_*` `STATE_*`.

## Threading / failure / edge cases

- **`reinit` runs on every load, `net_spawn` only on actual spawn:** per-object callbacks live in `reinit`
  so they survive load; one-time world setup lives in `net_spawn`. Registering a callback in the wrong one is
  a classic bug.
- **`net_spawn` can veto:** returning false from `object_binder.net_spawn` aborts the spawn — a binder that
  forgets to honor that early-return breaks objects.
- **Monster binder is a gamedata override:** the live `generic_object_binder` is in
  `gamedata/scripts/bind_monster.script` (the modded-exe `monster_move` hook + squad-follow are there) — cite
  the `gamedata/` copy. The `unpacked/` version is stale for the monster path.
- **class_registrator binds CLIENT = C++ class, SERVER = Lua se_ class:** the binder (actor_binder etc.) is
  NOT what's registered here; it's attached by the client object's bind hook. Confusing these is common.
- **callback cleanup must mirror registration:** `net_destroy` nils every callback `reinit` set — a missed one
  leaks a callback into a destroyed object.

## Deeper

- Actor binder: `unpacked/scripts/bind_stalker.script` (`actor_binder`, `init`/`actor_init`); logic in
  `bind_stalker_ext`
- Stalker binder: `unpacked/scripts/xr_motivator.script` (`motivator_binder`, `AddToMotivator`) → F66
- Monster binder: `gamedata/scripts/bind_monster.script` (`generic_object_binder`, `bind`) — **live override**
- Registration: `unpacked/scripts/class_registrator.script` (`register`, `cs_register`/`c_register`/`s_register`)
- Engine base + lifecycle: `flows/object-lifecycle.md` (F5), `flows/cscript-game-object.md` (F42),
  `engine/server-entities.md`
- Object binders (world/anomaly/restrictor/physics): F89–F93; map `modlayer/gameplay/binders.md` (M7)

## Maintenance

- Verified core (the attach hooks, the `object_binder` lifecycle contract, `class_registrator`, actor +
  monster binder spines) is stable. `bind_stalker_ext` (the actor's actual logic) is delegated-to — a
  separate read if a task needs actor manager internals.
- **The monster binder is a `gamedata/` override** — always cite/verify `gamedata/scripts/bind_monster.script`.
- The `bind_*` object binders (F89–F93) are deltas on this pattern; promote them as those flows land.
