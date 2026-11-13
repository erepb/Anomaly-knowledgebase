---
domain: flows/ai-perception
tags: [engine, cpp, ai, perception, memory, vision, sound, flow]
sources: [src/xrGame/memory_manager.cpp, src/xrGame/memory_manager.h, src/xrGame/visual_memory_manager.cpp, src/xrGame/visual_memory_manager.h, src/xrGame/sound_memory_manager.cpp, src/xrGame/hit_memory_manager.cpp, src/xrGame/danger_manager.h, src/xrGame/enemy_manager.h, src/xrGame/enemy_manager.cpp, src/xrGame/Actor.cpp, src/xrGame/Actor.h, src/xrGame/CustomMonster.cpp, gamedata/scripts/lua_help_ex.script]
related: [flows/ai-planner.md, flows/agent-squad-combat.md, flows/damage-hit.md, flows/object-lifecycle.md, flows/cscript-game-object.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
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

## Repo-added per-NPC perception levers (verified)

This repo grafts **per-object multipliers/overrides onto the vanilla formulas**, all exposed on
`game_object` and all documented from the Lua side in `gamedata/scripts/lua_help_ex.script` (that file
is the authoritative catalog; the mechanism notes here are what the KB adds). None of them are
serialized — they live on the manager instance, so they survive while the NPC is online and must be
re-applied on spawn (the memory manager / condition is rebuilt when it comes back online).

Vision, on `CVisualMemoryManager` (both fields are deliberately *not* cleared by
`reinit`/`reload` — see the comment on `m_vision_speed`):

- `npc:set_vision_speed(k)` → `m_vision_speed`, a factor on `get_visible_value` applied *after* the
  global `ai_vision_speed_boost` cvar and after the optional Lua `visual_memory_manager.get_visible_value`
  override. It scales how fast the accumulate-to-threshold counter fills, i.e. time-to-spot.
- `npc:set_view_distance_factor(k)` → `m_view_distance_factor`, a factor on
  `object_visible_distance`'s result, so it composes with the per-mental-state `max_view_distance`
  LTX multiplier. This is the *range* sibling of vision speed; negatives clamp to 0 in both setters.

Enemy selection, on `CEnemyManager::evaluate` (lower score wins; vanilla subtracts fixed penalties):

- `npc:set_visible_enemy_bias(actor_bias, npc_bias)` replaces the hardcoded `900` subtracted when the
  **actor** currently sees this NPC and the `300` for another NPC that does. Each argument is
  independent; a negative one keeps that side vanilla, `0` removes the pull entirely.
- `npc:set_hit_redirect(max, falloff)` replaces the vanilla flat CQB nudge toward
  `hit().last_hit_object_id()` with a distance-decaying bonus: within `falloff` metres it subtracts
  `max * (1 - d²/falloff²)`, so a close attacker is prioritised and a distant one fades to nothing.
  `max < 0` keeps vanilla (the −5 CQB step under 30 m plus the standard aggro advantage beyond).
- `npc:make_enemy_visible(enemy)` calls the engine's own
  `CMemoryManager::make_object_visible_somewhen` (the same call squad wounded-target distribution
  uses) to plant the target in the **seen** class. This matters because `evaluate` scores "currently
  seen" far above hit/sound memory, so an attacker injected only into hit memory can never win
  selection. Visibility then decays by the normal memory rules. Stalker receiver, alive-entity
  argument, error log otherwise.

Actor-side: the **"how visible am I" value** the HUD noise/luminosity dial shows moved out of the HUD
widget into `CActor` (`m_npc_visibility` / `GetVisibility` / `SetVisibilityValue` /
`SetVisibilityFromObject`, with free functions `SetActorVisibility(who_id, value)` /
`GetActorVisibility()` as the seam; `CVisualMemoryManager` reports per-observer values into it). It is
the max over observers, smoothed toward the target per frame. Scripts read it as
`get_actor_visibility()`; `get_actor_ui_luminosity()` remains as a deprecated alias. Previously the
state lived in `CUIMotionIcon`, so it only existed while that HUD element did.

## Sight-ray fixes (merged 2026.11.09)

- **Dynamic objects occlude again** — `Vision::o_trace` (`xrEngine/Feel_Vision.cpp`) used to pad the
  target distance by 0.2 m *before* dividing the direction by it, so the ray direction was not unit
  length; it now normalises by the true distance and casts exactly that far (PRs #818/#828).
- **Anomaly volumes don't block sight** — `CSpaceRestrictor` marks zone trigger spatials
  `STYPE_FEELVISIONIGNORE` (`space_restrictor.cpp`, under `SPATIAL_CHANGE`), regardless of
  `g_ai_die_in_anomaly`; previously NPCs could not see through anomalies, even disabled ones (#825).
- `CHitMemoryManager` tolerates a null `m_hits` (MT crash guard, `hit_memory_manager.cpp`).

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
- **`CEnemyManager`'s hit-redirect fields are not initialized by the constructor.**
  `m_hit_redirect_max` / `m_hit_redirect_falloff` are assigned only inside `set_visible_enemy_bias` and
  `set_hit_redirect`, and neither the ctor nor the header default-initializes them, so `evaluate()`
  reads an indeterminate value on any NPC where no script ever called either setter (the read happens
  whenever the candidate is the last-hit object). Related order dependence:
  `set_visible_enemy_bias` **resets** hit-redirect to its defaults, so calling it after
  `set_hit_redirect` silently discards the redirect tuning.
- **Per-NPC levers are not saved.** They must be re-applied on every spawn/online transition; nothing
  warns when they are lost.
- **`remove_links` no longer assumes its list exists.** All three sense managers (`visual`, `sound`,
  `hit`) replaced `VERIFY(m_<list>)` with a null check — a `CMemoryManager::remove_links` crash in the
  MT build, i.e. relcase can reach a manager whose list is already gone.

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
