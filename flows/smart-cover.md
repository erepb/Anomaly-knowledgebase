---
domain: flows/smart-cover
tags: [xrgame, ai, smart-cover, loophole, cover, animation, flow]
sources: [src/xrGame/smart_cover_object.cpp, src/xrGame/smart_cover_object.h, src/xrGame/smart_cover_loophole.cpp, src/xrGame/smart_cover_loophole.h, src/xrGame/cover_manager.cpp, src/xrGame/cover_manager.h, src/xrGame/cover_evaluators.cpp]
related: [flows/ai-planner.md, flows/stalker-execution.md, flows/animation-callbacks.md, flows/ai-perception.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: smart-cover system

How NPCs use **smart covers** — designer-placed cover objects with **loopholes** (firing/peek positions
that own enter/idle/fire/lookout animations + transitions) — in combat, and how that differs from the
**auto-generated cover-point grid**. Smart covers sit under the stalker execution layer
(`flows/stalker-execution.md`, whose movement manager is the smart-cover variant) and run a small GOAP
to pick loophole actions.

**Provenance:** spine personally code-verified — `CCoverManager` owns both `m_smart_covers` and
`add_smart_cover` (the registration entry), and the auto-grid covers; `smart_cover::object` /
`smart_cover::loophole` exist. The loophole structure (fov/animations/transitions), the
`animation_planner` GOAP for loophole actions, the occupy/fire sequence, and `cover_evaluators` are
confirmed homes but *(agent-traced — not personally re-read)*. The Lua loophole descriptions are
base-game `unpacked/` (read-only reference).

## Two cover systems (registration verified; distinguish them)

`CCoverManager` (verified) holds **two** things:
- **`m_smart_covers`** — designer-placed `smart_cover::object`s, registered via **`add_smart_cover`**
  (verified) when the cover spawns; each has a **description** (cover type + named loopholes, from a
  Lua table) and enter/exit enemy-distance thresholds.
- the **auto cover grid** — `CCoverPoint`s computed from the level AI-map's per-vertex high/low cover
  values (`compute_static_cover`), a quad-tree of pure-geometry hiding spots with **no animations**.

So: **smart cover = authored, animated, has loopholes; cover-grid point = generated, geometric, no
animation.** `cover_evaluators` (`CCoverEvaluatorBase` + close/far/best subclasses) score points by
enemy line-of-sight for dynamic cover selection when no smart cover applies.

## Smart cover object & loophole (agent-traced)

`smart_cover::object` is a level-placed object; on spawn it `add_smart_cover`s its **description**
(named after a cover type like `cover_sit_1`) which lists **loopholes**. A `smart_cover::loophole`
holds: a `fov_position` (where the NPC stands), `fov_direction`/`danger_fov_direction` (where it can
shoot from there), `fov`/`range`, `enterable`/`usable`/`exitable` flags, an **action set** (idle /
lookout / fire / reload — each a list of animations), and a **transition graph** between loopholes
(with edge animations). A loophole is chosen for an enemy by whether the enemy is in its fov/range.

## NPC uses a smart cover (agent-traced)

1. **Assigned** a target cover (`set_dest_smart_cover` from script, or via smart_terrain/planner;
   `flows/stalker-execution.md`'s movement manager is the smart-cover variant).
2. **Path to a loophole:** the smart-cover movement manager finds the nearest enterable loophole and a
   loophole-path to the target loophole, playing **enter/transition animations** through intermediate
   loopholes.
3. **Occupy:** once in, a small **`animation_planner`** (a script-action GOAP over loophole evaluators —
   `in_cover`/`cover_entered`/`loophole_actual`/`can_fire_at_enemy`) selects a loophole **action**
   (idle / lookout / fire) and the **`animation_selector`** plays that action's animation from the
   loophole's set.
4. **Fire from loophole:** the fire action keeps the enemy in the loophole's fov, drives sight, and
   fires the weapon while playing the loophole's shoot animation (`flows/stalker-execution.md`).
5. **Transition/exit:** switching actions/loopholes plays transition animations; exit leaves the cover.

So a smart cover is essentially **animation-driven combat positioning**: the loophole dictates where to
stand, where you can shoot, and exactly which animations to play for each state.

## Isolation ledger (the seams)

- **Stalker execution:** the smart-cover movement manager occupies loopholes; sight/fire come from
  `flows/stalker-execution.md`.
- **Planner:** loophole action selection is a script-action GOAP (`animation_planner`) with smart-cover
  evaluators; the top-level decision to use cover is the combat planner (`flows/ai-planner.md`).
- **Animation:** loophole/transition animations via `animation_selector` → `LL_PlayCycle`
  (`flows/animation-callbacks.md`).
- **Perception:** target/enemy in fov drives loophole choice (`flows/ai-perception.md`).
- **Cover grid:** `CCoverManager` + `cover_evaluators` for dynamic (non-smart) cover points.
- **Lua / config:** smart-cover **descriptions** + loophole definitions (fov, actions, transitions) are
  Lua tables (base-game `smart_covers_*` scripts); `smart_cover.ltx` defaults (`enter/exit_min_enemy_
  distance`, `can_fire`, binder `bind_smart_cover`); server entity `se_smart_cover` stores per-loophole
  availability. Lua surface: `set_dest_smart_cover`, `in_smart_cover`, `set_smart_cover_target_*`,
  `use_smart_covers_only`.

## Threading

Main thread (AI scheduled/render updates). Animation playback rides the usual worker-thread bone calc
(`flows/animation-callbacks.md`).

## Failure & edge cases

- **Smart cover vs grid point:** only smart covers have loopholes/animations; a "cover" without
  animations is an auto-grid `CCoverPoint`. Mixing them up is the main confusion.
- **Loophole fov gates use:** an NPC only fires from a loophole if the enemy is in that loophole's
  fov/range — a cover that "doesn't work" against an enemy often has no loophole facing it.
- **Description/animation mismatch:** a loophole referencing missing animations strands the NPC in
  enter/transition; the Lua description must match the model's animations.
- **Availability:** `se_smart_cover` per-loophole availability + `enter/exit_min_enemy_distance` gate
  whether/when an NPC will take the cover.
- **Definitions are Lua/base-game:** loophole descriptions live in `unpacked/` scripts (read-only) —
  new covers are authored there, not in C++.

## Deeper

- Object/registration: `src/xrGame/smart_cover_object.cpp`/`.h` (`net_Spawn` → `add_smart_cover`),
  `cover_manager.cpp`/`.h` (`CCoverManager`, `add_smart_cover`, `m_smart_covers`, `compute_static_cover`)
- Loophole: `src/xrGame/smart_cover_loophole.cpp`/`.h` (fov/actions/transitions),
  `smart_cover_description.cpp`, `smart_cover_loophole_planner_actions.*` (loophole_fire/idle/lookout),
  `smart_cover_animation_*` (planner/selector)
- Cover scoring: `src/xrGame/cover_evaluators.cpp` (`CCoverEvaluatorBase` + close/far/best, smart-cover
  evaluators)
- Movement integration: `stalker_movement_manager_smart_cover*` — `flows/stalker-execution.md`
- Lua descriptions: `unpacked/scripts/smart_covers_*` (read-only), `bind_smart_cover.script`,
  `se_smart_cover.script`

## Maintenance

- Keep the **smart cover (authored, animated, loopholes) vs cover-grid point (generated, geometric)**
  distinction — it is the key navigation fact.
- Re-grep `CCoverManager::add_smart_cover` / `smart_cover::loophole` if the cover registration or
  loophole structure changes.
- Promote the *(agent-traced)* loophole/animation-planner bodies to verified when a cover-authoring or
  combat task needs the exact action/transition logic.
