---
domain: modlayer/gameplay/binders
tags: [modlayer, gameplay, unpacked, lua, lifecycle, callbacks, map]
sources: [unpacked/scripts/bind_stalker.script, unpacked/scripts/class_registrator.script, unpacked/scripts/xr_motivator.script, gamedata/scripts/bind_monster.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, modlayer/gameplay/physics-schemes-ph.md, flows/binder-architecture.md, flows/object-lifecycle.md, flows/scheme-engine-core.md, flows/cscript-game-object.md, flows/db-registry.md, flows/gameplay-boot.md, flows/se-stalker.md, flows/physics-object-schemes.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Binders (`bind_*`) — L1 map

The **client-side attach tier**: how Lua hooks onto an online engine object. Every `bind_*.script` defines
an `object_binder` subclass (the Lua face of the C++ `CScriptBinder`, F5) plus a module attach entry
(`init`/`bind`) that the engine C++ object calls to `obj:bind_object(binder)`. The binder then owns the
object's `db.storage[id]` blackboard (F69), registers its engine callbacks (F1/F42), and runs its logic /
scheme tick each frame. This is the online counterpart to the `se_*` offline server tier (F83/F84). This
map routes the family; the pattern lives in `flows/binder-architecture.md` (F88).

**Read first:** `flows/binder-architecture.md` (F88) — the binder lifecycle contract
(`reinit`/`net_spawn`/`net_destroy`/`update`/`save`/`load`) + the attach mechanism. Always grep
`gamedata/scripts/` for an override; **`bind_monster.script` IS a live override**.

## The core binders (verified in F88)

| Binder | File | Attaches to | Notes | Flow |
|--------|------|-------------|-------|------|
| `actor_binder` | `bind_stalker.script` | the player (`CActor`) | delegates logic to `bind_stalker_ext` | F88 ✅ |
| `motivator_binder` | `xr_motivator.script` | stalkers/NPCs | the scheme tick | F66 ✅ |
| `generic_object_binder` | `bind_monster.script` (**gamedata override**) | monsters | scheme tick + squad-follow | F88 ✅ |

`object_binder` itself is the C++ `CScriptBinder` (`engine/server-entities.md`, F5). `class_registrator`
(F65) maps each clsid → client (C++) + server (`se_*`) class; the binder is attached by the client object's
own bind hook, not by class_registrator.

## The object-binder catalog (verified by listing `unpacked/scripts/bind_*`; mechanics ⬜ per flow)

- **World objects** (F89 ✅ → `flows/world-object-binders.md`): `bind_physic_object` (`generic_physics_binder`,
  the canonical scheme-running object binder), `bind_campfire`, `bind_camp`, `bind_container`, `bind_door_*`,
  `bind_dynamic_light`, `bind_dynamo_hand`, `bind_faction`, `bind_awr`.
- **Anomaly fields** (F90 ✅ → `flows/anomaly-field-binders.md`): `bind_anomaly_zone` (the **artefact-spawn
  economy** + layers), `bind_anomaly_field` (per-field binder + the **dynamic-anomalies** relocation system +
  activation veto), `plugins/dynamic_anomalies.ltx`.
- **Restrictors — passive `sr_*`** (F91 ✅ → `flows/restrictors-passive-sr.md`): `sr_idle/timer/light/
  particle/postprocess/silence/…`.
- **Restrictors — active `sr_*`** (F92 ✅ → `flows/restrictors-active-sr.md`): `sr_camp/cutscene/teleport/
  monster/crow_spawner`.
- **Physics-object schemes `ph_*`** (F93 ✅ → `flows/physics-object-schemes.md`): `ph_door/button/car/
  minigun/…` (`stype_item` schemes run by F89's `generic_physics_binder`), `phantom_manager` (binder +
  psy-phantom spawn service).

*(These F89–F93 are deltas on the F88 pattern: an `object_binder` subclass + a scheme on the object — all
documented.)*

## How it fits (verified in F88)

1. Engine creates a client object → calls its Lua bind hook (`actor_init`/`init`/`bind`) →
   `obj:bind_object(<binder>(obj))`.
2. Engine drives the binder lifecycle: `reinit` (blackboard + callbacks) → `net_spawn` (online) →
   `update` per frame (logic / `xr_logic` scheme tick, F66) → `save`/`load` (F15) → `net_destroy` (offline).
3. The binder's `db.storage[id]` blackboard (F69) is where schemes (F66) and everything else read/write.

## Subsystem deep-dives (delegates to flows)

- The binder pattern + actor/monster binders + class_registrator → `flows/binder-architecture.md` (F88) ✅
- The stalker scheme binder → `flows/scheme-engine-core.md` (F66) ✅
- World-object binders (physics/campfire/camp/faction/container/light/door) → `flows/world-object-binders.md` (F89) ✅
- Anomaly-field binders (artefact spawn + dynamic anomalies) → `flows/anomaly-field-binders.md` (F90) ✅
- Restrictor `sr_*` schemes → `flows/restrictors-passive-sr.md` (F91) ✅ + `flows/restrictors-active-sr.md` (F92) ✅
- Physics-object `ph_*` schemes + phantom_manager → `flows/physics-object-schemes.md` (F93) ✅
- Engine base + C++ lifecycle → `flows/object-lifecycle.md` (F5), `engine/server-entities.md`

## Maintenance

- This map is the binder routing hub; `ROADMAP-gameplay.md` (Phase G4, F88–F93, all ✅) is the
  authoritative plan/status.
- **`bind_monster.script` is a `gamedata/` override** — always honor it; re-check other `bind_*` for overrides.
- Re-list `unpacked/scripts/bind_*` if the corpus changes.
