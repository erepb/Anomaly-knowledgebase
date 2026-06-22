---
domain: flows/ai-perception
tags: [engine, cpp, ai, perception, memory, vision, sound, flow]
sources: [src/xrGame/memory_manager.cpp, src/xrGame/memory_manager.h, src/xrGame/visual_memory_manager.cpp, src/xrGame/sound_memory_manager.cpp, src/xrGame/hit_memory_manager.cpp, src/xrGame/danger_manager.h, src/xrGame/enemy_manager.h, src/xrGame/CustomMonster.cpp]
related: [flows/ai-planner.md, flows/damage-hit.md, flows/object-lifecycle.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: AI perception / memory

How an NPC senses the world (vision, sound, hit) and stores it as **memory** that the GOAP planner's
property evaluators read. This is the input side of `flows/ai-planner.md`: the planner asks
`memory().enemy().selected()` / `visible_now(...)` / `danger().selected()`; this flow is what
populates those.

**Provenance:** researched with a read-only agent, then spot-verified against code. The manager
homes (`CMemoryManager::update` + the visual/sound/hit/enemy/danger sub-managers, the
`CCustomMonster::shedule_Update` driver, `CEnemyManager : CObjectManager<const CEntityAlive>`) are
personally code-verified. The detailed bodies (visibility-accumulation formula, danger scoring,
enemy evaluation weights) are *(agent-traced — not personally re-read)*; the agent marked several
SKIMMED.

## The aggregator (verified home)

**`CMemoryManager`** (`memory_manager.cpp/.h`) owns the sub-managers and is the per-tick entry:
`CMemoryManager::update(time_delta)` (verified) updates the raw-sense managers (visual → sound → hit)
then aggregates their object lists into the higher-level enemy and danger managers
(`update_enemies` verified present). Accessors `enemy()`/`visual()`/`sound()`/`hit()`/`danger()` are
what the planner's evaluators call. It is driven from **`CCustomMonster::shedule_Update`** (verified)
— i.e. on the **deferred** scheduler channel (the backbone, `flows/object-lifecycle.md`), after
`Exec_Visibility`.

## The perception → memory → planner pipeline

Three raw senses arrive via engine **Feel** callbacks and are smoothed into memory:

1. **Vision** — `CVisualMemoryManager::update` (verified home) pulls the engine's currently-visible
   object set (`Feel::Vision`), runs a per-object visibility test (FOV angle, distance, luminosity,
   material transparency) that **accumulates** a visibility value over frames, and once it crosses a
   threshold stores a remembered `CVisibleObject`. Stale entries are marked not-visible after a
   still-visible timeout but linger until offline. Query side: `visible_now`/`visible_right_now`.
   *(formula agent-traced; the accumulate-then-threshold mechanism is the load-bearing idea.)*
2. **Sound** — `CSoundMemoryManager::feel_sound_new` (verified home) is the `Feel::Sound` callback:
   it applies a per-type factor (weapon/item/npc/anomaly/world), runs the script `sound_callback`
   hook, and above a power threshold stores a `CSoundObject` (filtering own/teammate/friendly
   sounds). *(body agent-traced.)*
3. **Hit** — `CHitMemoryManager::add` (verified home) records the attacker when the entity is hit
   (linked to the damage flow's `eHit`, `flows/damage-hit.md`): stores a `CHitObject` (attacker,
   amount, direction, bone) and the last-hit id/time. *(body agent-traced.)*

These feed two **derived** managers the planner actually reads:

- **Enemy** — `CEnemyManager` (verified: extends `CObjectManager<const CEntityAlive>`) picks the
  single `selected()` enemy by evaluating candidates (visibility, recent hits, distance) and only
  re-selecting when needed (`need_update`). *(evaluation weights agent-traced.)*
- **Danger** — `CDangerManager` synthesizes `CDangerObject`s from visible corpses, threatening
  sounds (ricochet/attack/death), and hits, scores them by type + age, and exposes the top one as
  `selected()`. *(scoring agent-traced.)*

So: **raw Feel callback → manager update → memory entry stored → derived enemy/danger selection →
planner evaluator reads `selected()`/`visible_now()`.**

## Isolation ledger (the seams)

- **Scheduling:** `memory().update(dt)` runs every `CCustomMonster::shedule_Update` tick (deferred
  channel, serial scheduler).
- **Config read:** vision params (min/max view distance, FOV, visibility threshold, always-visible
  distance, time quant, velocity/luminosity factors, still-visible time) from the character `.ltx`
  vision sections (free vs danger, switched by mental state); sound perceive params (threshold, per-
  type factors, capacity); hit/enemy capacities + ignore thresholds. *(key names agent-traced.)*
- **Engine Feel seam:** `Feel::Vision` (engine supplies the visible set), `Feel::Sound`
  (`feel_sound_new`), and the hit path from the damage flow. `CCustomMonster` inherits the Feel
  interfaces.
- **Feeds other flows:** the GOAP evaluators in `flows/ai-planner.md` read `enemy().selected()`,
  `danger().selected()`, `visual().visible_now()`; hit memory links back to `flows/damage-hit.md`
  (`eHit`); selected enemy drives mental-state → combat.
- **Lua exposure:** `obj:memory()` exposes the visible/sound/hit/danger object lists and
  `enemy():selected()`/`danger():selected()`; script hooks `sound_callback`, the `eHit` callback, and
  an optional `get_visible_value` override of the vision formula. *(surface agent-traced.)*

## Threading

Main thread (deferred scheduler channel, serial).

## Failure & edge cases

- **Vision is accumulated, not instant:** an object in FOV isn't "seen" until its accumulated value
  crosses the threshold (distance/luminosity/velocity-dependent) — low light / long range delays
  detection; `always_visible_distance` is the point-blank shortcut.
- **Mental state swaps the vision params** (free vs danger section) — the same NPC perceives
  differently in combat.
- **Danger is the bridge for sound:** planners read `danger().selected()`, not sound memory directly;
  a heard threat matters only once it becomes a danger object.
- **Memory lingers:** an unseen enemy stays remembered (decaying) until offline — `visible_now` ≠ "in
  FOV this instant" (use `visible_right_now` for the strict check).

## Deeper

- Aggregator: `src/xrGame/memory_manager.cpp/.h` (`update`, `update_enemies`, accessors)
- Vision: `src/xrGame/visual_memory_manager.cpp` (`update`, `visible`, `visible_now`)
- Sound: `src/xrGame/sound_memory_manager.cpp` (`feel_sound_new`, `add`)
- Hit: `src/xrGame/hit_memory_manager.cpp` (`add`)
- Derived: `src/xrGame/enemy_manager.h`, `src/xrGame/danger_manager.h` (`selected`, `update`)
- Driver: `src/xrGame/CustomMonster.cpp` (`shedule_Update` → `memory().update`)

## Maintenance

- Re-grep `CMemoryManager::update`, `CVisualMemoryManager::update`,
  `CSoundMemoryManager::feel_sound_new`, `CHitMemoryManager::add` if managers move.
- Promote the *(agent-traced)* bodies (vision formula, danger scoring, enemy evaluation weights,
  config keys) to verified by reading them when a task needs the tuning detail.
- Keep the "accumulated vision + danger-as-sound-bridge + memory lingers" notes — all three correct
  naive "the NPC instantly sees/hears what's in range" assumptions.
