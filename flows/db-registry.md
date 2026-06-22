---
domain: flows/db-registry
tags: [modlayer, gameplay, unpacked, lua, registry, state, flow]
sources: [unpacked/scripts/db.script, unpacked/scripts/_g.script]
related: [flows/object-lifecycle.md, flows/cscript-game-object.md, flows/scheme-engine-core.md, flows/gameplay-boot.md, flows/utils-libraries.md, modlayer/gameplay/utils-db.md, modlayer/gameplay/_overview.md, modlayer/scripting.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: db — the global Lua state registry (F69)

`db.script` is the central mutable registry every gameplay script reads and writes. It is the Lua-side
mirror of the live world: a per-object state store keyed by object id, the actor reference, and a set
of name/id-keyed lookup tables for object categories (zones, anomalies, helis, smart_terrains, …).
Binders populate it on spawn; schemes, UI, and AI read it every frame. This is the spine of F69.

**Provenance:** `unpacked/scripts/db.script` read in full and verified (no `gamedata/` override exists).
The cross-script *writers/readers* named below are confirmed by the function bodies in `db.script`
itself (e.g. `add_obj` requires `storage[id]` to pre-exist) and by the boot flow; the exact binder
call-sites are *(agent-traced — not personally re-read)* and cited as such.

## What db holds (verified — the actual tables)

`db.script` declares these module globals at load (all verified present):
- **`storage`** — the per-object state store, keyed by `obj:id()`. Each live object has an entry that
  holds `.object` (the game object, set by `add_obj`) and the scheme/logic state table written by its
  binder's `reinit` and by `xr_logic` (`.st`, `active_section`, `active_scheme`, per-scheme subtables,
  `pstor` persistent storage). This is the table `xr_logic`/`xr_motivator` operate on (see
  `flows/scheme-engine-core.md`).
- **`actor`** — the actor game object when online, `nil` when offline. The ubiquitous `if db.actor`
  guard. Paired with **`actor_proxy`** (an `actor_proxy.actor_proxy()` instance) which bridges
  offline/online so scripts can resolve the actor id while offline.
- **`OnlineStalkers`** — array of online `script_stalker` object ids (dedup-checked).
- Name/id-keyed category lookups: **`zone_by_name`**, **`anomaly_by_name`**, **`anim_obj_by_name`**,
  **`signal_light`**, **`smart_terrain_by_id`**, **`heli`**, **`heli_enemies`**, **`bridge_by_name`**,
  **`camp_storage`**, **`campfire_by_name`**/**`campfire_table_by_smart_names`**, **`story_by_id`**/
  **`story_object`**, **`info_restr`**, **`no_weap_zones`**, **`level_doors`**, **`script_ids`**.
- World-coordination tables: **`used_level_vertex_ids`**, **`spawned_vertex_by_id`**,
  **`offline_objects`**, **`actor_inside_zones`** (which restrictors the actor is currently inside),
  **`dynamic_ltx`** (runtime-built INI per gulag/smart name), **`goodwill`** (`{sympathy, relations}`),
  **`vehicle`** (current vehicle or nil).

## Sequence (verified) — register / unregister

The object lifecycle (F5 `flows/object-lifecycle.md`) drives db through these entry points:

- **`db.script:add_obj(obj)`** — `storage[obj:id()].object = obj`; if `obj:clsid() == clsid.script_stalker`
  also `add_stalker(obj)`. **Note (verified):** it indexes `storage[id]` *expecting it to already
  exist* — the binder's `reinit` creates `storage[id]` first, then `add_obj` fills `.object`.
- **`db.script:del_obj(obj)`** — `storage[obj:id()] = nil` (+ `del_stalker` for stalkers). This is the
  single point where an object's whole Lua state is dropped.
- **`db.script:add_actor(obj)` / `del_actor()`** — set/clear `actor`, drive `actor_proxy:net_spawn/
  net_destroy`, and call `add_obj/del_obj`. So the actor is also a normal `storage` entry.
- Category register/unregister (each a 1–3 line setter, verified): `add_zone/del_zone`,
  `add_anomaly/del_anomaly`, `add_anim_obj/del_anim_obj`, `add_heli/del_heli`,
  `add_smart_terrain/del_smart_terrain`, `add_stalker/del_stalker` (array, dedup + `table.remove`),
  `add_enemy/delete_enemy` (heli enemies), `add_bridge/del_bridge`.

## Function digests (spine)

- **`db.script:add_obj` / `del_obj`** — *Does:* create/drop the `storage[id]` entry's `.object` and the
  OnlineStalkers membership. *Reads:* `obj:id()`, `obj:clsid()`. *Writes:* `storage`, `OnlineStalkers`.
  *Called by:* binders on `net_spawn`/`net_destroy` *(agent-traced)*. *(verified.)*
- **`db.script:add_actor` / `del_actor`** — *Does:* maintain the global `actor` ref + `actor_proxy` +
  the actor's `storage` entry. *Writes:* `actor`, `actor_proxy`, `storage`. *(verified.)*
- **`db.script:add_stalker` / `del_stalker`** — *Does:* dedup-append / `table.remove` ids in
  `OnlineStalkers`. *Gotcha:* `del_stalker` guards a nil list and searches back-to-front. *(verified.)*
- **category setters** (`add_zone`, `add_anomaly`, `add_anim_obj`, `add_heli`, `add_smart_terrain`,
  `add_bridge`, `add_enemy`) — *Does:* index a category table by `name()`/`id`. *(verified.)*

## Isolation ledger (the seams)

- **Owned state:** all of the above tables — process-global, mutable, no encapsulation. This *is* the
  shared blackboard of the gameplay layer.
- **Writers:** the `bind_*` binders, on `net_spawn`/`reinit`/`net_destroy` (`flows/object-lifecycle.md`;
  the binder layer is roadmap F88). They create `storage[id]`, then `db.add_obj`.
- **Readers:** essentially everything — `xr_logic`/`xr_motivator` (`flows/scheme-engine-core.md`),
  schemes, UI, AI (`axr_beh`/`axr_fight_from_cover` use `used_level_vertex_ids`), restrictors
  (`actor_inside_zones`), helis (`heli`/`heli_enemies`).
- **Persistence seam:** `alife_storage_manager` marshals `storage[id].pstor`/`pstor_ctime` to the save
  (`flows/save-load.md`). `db` itself stores no save logic; it is reconstructed by binders on load.
- **Engine seam:** ids/clsids come from `CScriptGameObject` (`flows/cscript-game-object.md`); `clsid`
  table from the engine class registry (`flows/gameplay-boot.md`).

## Threading / failure / edge cases

- **Lazy creation invariant:** `storage[id]` must exist before `add_obj`; a binder that calls `add_obj`
  without first creating `storage[id]` in `reinit` will index-nil-crash. Respect the reinit→add_obj
  order.
- **Offline actor:** `db.actor` is `nil` offline; the near-universal `if db.actor` guard exists because
  of this. Use `db.actor_proxy` when an id is needed offline.
- **No GC of stale ids:** entries live until `del_obj`; a missed `net_destroy` leaks a `storage` entry.

## Deeper

- Registry: `unpacked/scripts/db.script` (tables + `add_*`/`del_*`)
- Actor proxy: `unpacked/scripts/actor_proxy.script` (`actor_proxy`)
- Consumers: `flows/scheme-engine-core.md` (`storage[id]` as the logic state), `flows/object-lifecycle.md`
  (who calls `add_obj`/`del_obj`), `flows/save-load.md` (`pstor` persistence)

## Maintenance

- `db.script` is the source of truth for which registries exist; if a table here is added/removed,
  re-list the globals at the top of the file and update the "What db holds" list.
- Check `gamedata/scripts/db.script` for an override before trusting this (currently none).
- The binder call-sites are agent-traced; promote to verified when F88 (binder architecture) is written.
