---
domain: modlayer/gameplay/vehicles-heli
tags: [modlayer, gameplay, unpacked, lua, ai, physics, map]
sources: [unpacked/scripts/bind_heli.script, unpacked/scripts/heli_alife.script, unpacked/scripts/bind_car.script, unpacked/scripts/se_car.script, unpacked/scripts/ph_car.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, modlayer/gameplay/binders.md, flows/helicopter-ai.md, flows/physics-object-schemes.md, flows/vehicles-holder.md, flows/binder-architecture.md, flows/se-others.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Vehicles — helicopters & cars (Lua) — L1 map

The vehicle Lua tier: the helicopter stack (binder + combat manager + story scheme + ALife simulation
layer) and the car binding. Engine classes are F29 (`CCar` = mountable `CHolderCustom`; `CHelicopter` =
AI aircraft, not mountable). Helicopter mechanics live in `flows/helicopter-ai.md` (F95); the car side
(F96) is small enough to live here.

## Helicopters (F95 ✅ → `flows/helicopter-ai.md`)

| Piece | File | Role |
|-------|------|------|
| `heli_binder` | `bind_heli.script` | lifecycle, health/flame/death, hit flagging, `stype_heli` logic |
| `heli_combat` | `heli_combat.script` | always-on combat manager (flyby/round/search/retreat) |
| `heli_move` | `heli_move.script` | the only `stype_heli` scheme (patrol / coord nodes / retreat) |
| `heli_fly`/`heli_look`/`heli_fire` | same-named files | per-id flight/look/enemy-pick helpers |
| `heli_alife` | `heli_alife.script` | faction-heli ALife simulation (squad-style target FSM) |
| `heli_alife_warfare` | `heli_alife_warfare.script` | WARFARE-mode copy |
| `heli_snd` | `heli_snd.script` | voice banks (used from `xr_effects`) |
| `se_heli` | `se_heli.script` | server entity → `flows/se-others.md` (F84) |

Key behaviors: helis only attack the actor after being hit (`heli_enemy_flag`) or with
`actor_always_enemy`; combat pauses at night/surge; layer priority combat → alife/scheme.

## Cars (F96 ✅ — documented here; `bind_car`/`se_car` READ-IN-FULL 2026-06-11)

- **`bind_car.script`** (`car_binder`, attach hook `init`) — a near-clone of F89's
  `generic_physics_binder`: `reinit` builds `db.storage[id]`; `update` runs
  `xr_logic.initialize_obj(…, stype_item)` + issues scheme `update`; `net_spawn` registers
  hit/death/use callbacks + `db.add_obj`; `net_destroy` runs scheme `net_destroy` +
  `on_offline_condlist` + cleanup; `hit_callback` also issues to `st.ph_on_hit` (the F93 generic);
  save/load = `xr_logic.save_obj/load_obj`. **The car's logic scheme is `ph_car` (F93)** — usable mode
  (player-enterable, condlist-gated `set_nonscript_usable`) vs scripted mode (path driving + mounted
  weapon).
- **`se_car.script`** (`se_car`:`cse_alife_car`) — minimal F84-style passthrough; `on_register/
  on_unregister` SendScriptCallback + `keep_saved_data_anyway() = true`.
- **Player driving** is engine-side: `CCar` is a `CHolderCustom`, mounted via
  `CActor::attach_Vehicle` → `flows/vehicles-holder.md` (F29). No Lua in the drive loop.
- No `gamedata/` override of either file at writing.

## Subsystem deep-dives (delegates to flows)

- Helicopter stack → `flows/helicopter-ai.md` (F95) ✅
- Car scheme (`ph_car`) → `flows/physics-object-schemes.md` (F93) ✅
- Engine vehicle classes / mounting → `flows/vehicles-holder.md` (F29)
- Binder pattern → `flows/binder-architecture.md` (F88); server entities → `flows/se-others.md` (F84)

## Maintenance

- Re-check `gamedata/scripts/{heli_*,bind_heli,bind_car}` for overrides on pickup (none at writing).
- F96 is documented in this map (merged — the car Lua tier is a thin delta over F88/F93/F29); if the
  car side grows (mods adding car AI), promote it to its own flow and update `ROADMAP-gameplay.md`.
