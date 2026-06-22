---
domain: modlayer/gameplay/physics-schemes-ph
tags: [modlayer, gameplay, unpacked, lua, schemes, physics, map]
sources: [unpacked/scripts/ph_door.script, unpacked/scripts/ph_car.script, unpacked/scripts/ph_minigun.script, unpacked/scripts/phantom_manager.script, unpacked/scripts/modules.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, modlayer/gameplay/binders.md, flows/physics-object-schemes.md, flows/scheme-engine-core.md, flows/world-object-binders.md, flows/physics-objects-ragdoll.md, flows/vehicles-holder.md, flows/special-actors.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Physics-object schemes (`ph_*`) — L1 map

Logic schemes for **physics props** — the `stype_item` tier of the F66 scheme engine, run by the
`generic_physics_binder` (F89) on `physic_object`-type entities. Doors, buttons, code pads, impulses,
oscillators, sound sources, scripted cars, mounted guns. The mechanics live in
`flows/physics-object-schemes.md` (F93) — this map just routes the family.

## The family (13 files, all verified in F93; no `gamedata/` override)

| Scheme | File | What |
|--------|------|------|
| `ph_door` | `ph_door.script` | joint-physics door (+ NPC auto-open via `xr_motivator`/`db.level_doors`) |
| `ph_button` | `ph_button.script` | pressable switch → `on_press` condlist |
| `ph_code` | `ph_code.script` | code pad → `ui_numpad` → `on_code`/`on_check_code<N>` |
| `ph_idle` | `ph_idle.script` | interaction holder (tips, `on_use`, `hit_on_bone`) |
| `ph_hit` | `ph_hit.script` | one-shot strike impulse to itself on section entry |
| `ph_on_hit` | `ph_on_hit.script` | generic hit listener (auto-attached on `on_hit` field) |
| `ph_on_death` | `ph_death.script` | death listener → switch (note file≠scheme name) |
| `ph_oscillate` | `ph_oscillate.script` | periodic joint force swing |
| `ph_sound` | `ph_sound.script` | positional sound source (body rewired to Zone FM radio) |
| `ph_car` | `ph_car.script` | scripted/usable car on engine `CCar` (F29) |
| `ph_minigun` | `ph_minigun.script` | mounted gun (also `CCar`); `job` mode targets via SIMBOARD |
| *(none)* | `ph_appforce.script` | `ph_force` push — **unregistered, dead in base corpus** |
| *(binder)* | `phantom_manager.script` | psy-phantom spawn service + binders (NOT a scheme; F36/F91 seam) |

Registration: `modules.script` `LoadScheme(file, scheme, stype_item)` rows; `enable_generic_schemes`
auto-attaches `ph_on_hit`. Base configs reference mostly doors (235×), idle (58×), buttons (21×);
car/minigun/oscillate/hit/on_death are story-mod surface (0 base refs).

## How it fits

Engine `CPhysicObject` (F28) → `bind_physic_object`/`generic_physics_binder` (F89) → `xr_logic`
schemes `stype_item` (F66) → these actions. Use/hit/death engine callbacks arrive via the binder's
`issue_event`; high-rate work (door completion, car driving) runs on the `set_fastcall` engine channel.

## Subsystem deep-dives (delegates to flows)

- All `ph_*` mechanics + phantom_manager → `flows/physics-object-schemes.md` (F93) ✅
- The binder that runs them → `flows/world-object-binders.md` (F89), `flows/binder-architecture.md` (F88)
- Engine physics / vehicles / phantom → `flows/physics-objects-ragdoll.md` (F28),
  `flows/vehicles-holder.md` (F29), `flows/special-actors.md` (F36)

## Maintenance

- Re-list `unpacked/scripts/ph_*` + grep `gamedata/scripts/ph_*` for overrides on pickup (none at writing).
- Status lives in `ROADMAP-gameplay.md` (Phase G4); keep the table in sync if files are added/renamed.
