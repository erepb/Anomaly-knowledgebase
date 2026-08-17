---
domain: flows/damage-hit
tags: [engine, cpp, damage, hit, condition, death, flow, callbacks, alife]
sources: [src/xrGame/Hit.h, src/xrGame/Hit.cpp, src/xrGame/Entity.cpp, src/xrGame/entity_alive.cpp, src/xrGame/entity_alive.h, src/xrGame/EntityCondition.cpp, src/xrGame/EntityCondition.h, src/xrGame/damage_manager.cpp, src/xrGame/Wound.h, src/xrGame/GameObject.cpp, src/xrGame/Level_Bullet_Manager.cpp, src/xrGame/Level_Bullet_Manager.h, src/xrGame/character_hit_animations.cpp, src/xrGame/character_hit_animations.h, src/xrEngine/xr_object.cpp, src/xrEngine/xr_object.h]
related: [flows/weapon-fire-callbacks.md, flows/actor.md, flows/object-lifecycle.md, flows/creatures-actor-configs.md, flows/ai-perception.md, flows/stalker-execution.md, engine/xrGame.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: damage / hit resolution (hit → condition → death)

The **shared** path a hit takes after it lands: from an `SHit` delivered as a `GE_HIT` event,
through bone scaling and condition (health/wound) application, to death and the `eDeath` callback.
Every creature shares this — the actor (`flows/actor.md`) and weapon-fire (`flows/weapon-fire-
callbacks.md`) flows both *feed into* it. The bullet→`eHit` side is the START documented elsewhere;
this is what happens **after** `Hit()` is called.

**Provenance:** researched with a read-only agent, then spot-verified against code. The hit struct
(`SHit`/`GenHeader`), the `CEntity`→`CEntityAlive::Hit` chain, `CDamageManager::HitScale`,
`CEntityCondition::ConditionHit`, the `_G.CBeforeHitAfterCalcs` Lua hook, and the `eDeath` callback
in `CEntityAlive::Die` are personally code-verified (incl. `CActorCondition::ConditionHit` chaining
to the generic one). Deep bodies (`UpdateCondition` health math, wound/bleeding, the `GE_DIE`/
`GE_ASSIGN_KILLER` server side) are *(agent-traced — not personally re-read)*.

## The hit struct & delivery (verified)

A hit is an **`SHit`** (`Hit.h`): power, direction, impulse, bone element, `hit_type`
(burn/wound/strike/shock/radiation/explosion…), initiator (`who`/`whoID`), `weaponID`,
`armor_piercing`, `add_wound`, `aim_bullet`. `SHit::GenHeader(PacketType, ID)` (verified) stamps it
for the network as a `GE_HIT` (or `GE_HIT_STATISTIC`) event with the server time; `Write_Packet`
serializes it. So damage is delivered as an **event**, not a direct call — the same event-ordered
pattern as spawn/destroy (`flows/object-lifecycle.md`).

`CGameObject::OnEvent` (verified present in `GameObject.cpp`) reconstructs the `SHit` from the packet,
re-resolves `who`/`weapon` from the object registry, sets the hit info, and calls `Hit(&SHit)`.

## The Hit override chain (verified)

`Hit` is virtual and each layer adds to it:

1. **`CEntity::Hit(SHit*)`** (`Entity.cpp`) — converts the impulse to local space, applies
   `HitImpulse` (physics push), reduces health via `CalcCondition`, plays the hit signal, and if the
   entity dies locally with no killer yet assigned calls `KillEntity(whoID)`; chains
   `inherited::Hit`. *(agent-traced body.)*
2. **`CEntityAlive::Hit(SHit*)`** (verified, `entity_alive.cpp`) — the living-entity override: makes
   a working copy, normalizes the hit type, calls **`CDamageManager::HitScale(boneID, hit_bone_scale,
   wound_bone_scale, aim_bullet)`** (verified) to scale damage by the struck bone, then
   `conditions().ConditionHit(&HDS)` (the real damage), spawns fire particles for burns, adds bloody
   wallmarks, registers the fight/attack relation in `RELATION_REGISTRY`, and chains `inherited::Hit`.
3. **`CActor::Hit`** (the actor leaf — `flows/actor.md`) pre-processes (the `eHit` callback + artifact
   reduction) then chains down into this generic chain via `inherited::Hit`. Verified that
   `CActorCondition::ConditionHit` chains to `CEntityCondition::ConditionHit` — the actor reuses the
   shared condition math.

## Bone scaling (verified home)

`CDamageManager::HitScale` (`damage_manager.cpp`) looks up the struck bone in the skeleton and reads
its hit/wound scale coefficients (and a separate aim-bullet coefficient for an aimed first hit) from
the entity's `damage` config section; falls back to default factors for an unbound bone. This is the
"headshots hurt more" mechanism — per-bone multipliers from `.ltx`. *(home verified; param-index
detail agent-traced.)*

## Condition application (verified entry)

**`CEntityCondition::ConditionHit(SHit*)`** (verified, `EntityCondition.cpp`) is the damage core:
records the initiator, applies outfit/helmet armor, dispatches by `hit_type` applying the matching
immunity, **optionally calls the Lua functor `_G.CBeforeHitAfterCalcs`** (verified site) which can
inspect/modify the hit power via a `CScriptHit` before it lands, splits the result between health and
power (`m_fHealthHitPart`/`m_fPowerHitPart`), and — if `add_wound` and the entity is still alive —
creates a `CWound` via `AddWound`. Radiation is special (only accrues a radiation delta). *(entry +
Lua hook verified; the per-type math and `AddWound`/`UpdateCondition`/bleeding bodies agent-traced.)*

The accrued deltas are applied over time in `CEntityCondition::UpdateCondition` (the once-per-tick
condition update — the actor's is in `flows/actor.md`): bleeding from wounds, radiation, natural
regen, and the kill-threshold / last-chance-invulnerability logic; it can fire the virtual
`OnCriticalHitHealthLoss`/`OnCriticalWoundHealthLoss`/`OnCriticalRadiationHealthLoss` hooks.
*(agent-traced.)*

## Repo additions on the hit path (verified)

- **Visual-mesh hit validation** (`bullet_check_visual`, build-gated on `CBULLETMANAGER_EX`). A bone
  hitbox is a coarse capsule, so a bullet can "hit" a thin model through empty space. When an object's
  section sets `bullet_check_visual = true` (`CObject::Load`, defaults false, also settable at runtime
  via `obj:SetBulletCheckVisual`), `CBulletManager::firetrace_callback` additionally requires the ray
  to hit a **triangle of that bone's visual mesh** (`ValidateHitDynamicVisualMesh` → `PickBone`); on
  failure it returns `TRUE`, i.e. ignores the hit and lets the bullet keep flying, and on success it
  moves `collide_position` to the real mesh intersection. `bullet_test_callback_data` carries the
  trajectory's `max_distance` so the pick has a range.
- **Hit-flinch reader** `npc:is_hit_anim_playing()`. `character_hit_animation_controller::PlayHitMotion`
  now stamps an end time (`stamp_hit_anim_end`, the longest remaining blend converted to
  `Device.dwTimeGlobal` ms, taken on both the effected and non-effected paths) and
  `is_hit_anim_playing` is a single clock compare against it — no blend walk. Reachable for any alive
  entity with character physics support; false otherwise.
- **Passive regen boost** `npc:set_health_restore_boost(v)` stores `m_health_restore_boost`, added to
  `change_v().m_fV_HealthRestore` in `CEntityCondition::UpdateHealth`, i.e. HP trickles back with no
  item. Any `CEntityAlive`; negatives clamp to 0; not serialized, so re-apply on spawn (the
  condition is rebuilt when the entity comes back online).
- **Enemy-selection hit levers** (`set_hit_redirect`, and its interaction with `set_visible_enemy_bias`)
  read `hit().last_hit_object_id()` — documented with the hazard in `flows/ai-perception.md`.

## Death (verified callback)

When health crosses zero:

- **`CEntity::KillEntity(whoID)`** (`Entity.cpp`) records the killer; for the actor it can fire
  `eActorBeforeDeath` (a script veto point) and sends `GE_DIE`. *(agent-traced; actor veto under a
  build define.)*
- **`CEntityAlive::Die(who)`** (verified, `entity_alive.cpp`) registers the KILL relation, chains
  `inherited::Die` (sets death time, health = -1, unregisters from team/squad/group), fires the
  per-object **`callback(GameObject::eDeath)(self, killer)`** (verified site), disables sound
  reaction, and triggers the ragdoll via `character_physics_support()->in_Die()`. The same `eDeath`
  pattern is used by `CAI_Crow::Die` and `CDestroyablePhysicsObject` (verified sites).
- Server side, `GE_DIE` reaches `xrServer` → `game->on_death(victim, killer)` and rebroadcasts.
  *(agent-traced server path; see `engine/server-entities.md`.)*

## Isolation ledger (the seams)

- **Callbacks fired:** per-object `callback(GameObject::eDeath)(self, killer)` (verified in
  `entity_alive.cpp`); `eHit` is fired by the *leaf* (e.g. `CActor::Hit` — `flows/actor.md`), not in
  this generic chain; `eActorBeforeDeath` (actor, build-define veto). Lua functor
  `_G.CBeforeHitAfterCalcs(CScriptHit, target, bone)` (verified) mid-`ConditionHit`. Virtual
  `OnCritical*HealthLoss` hooks from `UpdateCondition`. *(non-`eDeath` enum/functor names beyond the
  verified sites are agent-traced.)*
- **Config read:** the `damage` section (per-bone hit/wound scales + defaults), hit-type immunities,
  the condition section (`health_hit_part`/`power_hit_part`, bleed speed, kill threshold, last-chance
  health/invulnerable time), bloody-wallmark + fire-particle sections, outfit/helmet armor. *(key
  names agent-traced; the `damage`/condition reads are confirmed via the verified call sites.)*
- **Event / network seam:** `GE_HIT`/`GE_HIT_STATISTIC` deliver the hit (built via
  `SHit::GenHeader`/`Write_Packet`, dispatched in `CGameObject::OnEvent`); `GE_ASSIGN_KILLER`
  persists the killer; `GE_DIE` notifies the server. Server is authoritative in MP. See
  `engine/server-entities.md`.
- **ALife / server seam:** killer id persists on the `CSE_ALifeCreatureAbstract`; `game->on_death`
  drives relations/quests; corpse removal is scheduled post-death.
- **Lua seam:** `_G.CBeforeHitAfterCalcs` (modify incoming hit); `CScriptHit` (power/dir/bone/
  draftsman/weapon); `get_killer`/`get_killer_id`, health getters; `eDeath` passes the killer object.
- **Other domains:** weapons as the hit source (`flows/weapon-fire-callbacks.md`), actor condition
  (`flows/actor.md`), physics (impulse + ragdoll), AI (`RELATION_REGISTRY` fight/kill registration
  → enemy awareness, `flows/ai-planner.md`), particles/wallmarks for feedback.

## Threading

Main thread; hits ride server-ordered events, so applied damage follows the authoritative server
decision in MP.

## Failure & edge cases

- **Damage is event-delivered** (`GE_HIT`) and re-resolves `who`/`weapon` from IDs on receipt — a
  hit whose initiator despawned arrives with a null `who`.
- **`_G.CBeforeHitAfterCalcs` can rewrite or cancel damage** before it applies — a mod hook here
  changes the effective power of every hit on every creature.
- **Two-stage death:** damage accrues as deltas in `ConditionHit`, but the actual zero-crossing /
  kill is resolved in `UpdateCondition`/`KillEntity` — the entity isn't dead the instant `Hit`
  returns. A kill-threshold + last-chance window can spare a lethal hit.
- **Per-bone scaling** means raw weapon power ≠ applied damage; the `damage` section governs it.

## Deeper

- Hit struct/events: `src/xrGame/Hit.h`, `src/xrGame/Hit.cpp` (`SHit`, `GenHeader`, `Write_Packet`),
  `src/xrGame/GameObject.cpp` (`OnEvent` GE_HIT)
- Hit chain: `src/xrGame/Entity.cpp` (`CEntity::Hit`, `KillEntity`, `Die`),
  `src/xrGame/entity_alive.cpp` (`CEntityAlive::Hit`, `Die`)
- Condition/wounds: `src/xrGame/EntityCondition.cpp/.h` (`ConditionHit`, `AddWound`,
  `UpdateCondition`), `src/xrGame/Wound.h`
- Bone scaling: `src/xrGame/damage_manager.cpp` (`HitScale`)
- Death server side: `xrServer` GE_DIE handler (`engine/server-entities.md`)

## Maintenance

- Re-grep `CEntityAlive::Hit`, `ConditionHit`, `HitScale`, and `callback(GameObject::eDeath)` if the
  chain moves; re-grep `_G.CBeforeHitAfterCalcs` to confirm the Lua hook fires.
- Promote the *(agent-traced)* bodies (`CEntity::Hit`, `UpdateCondition` health/bleeding math,
  `AddWound`, `KillEntity`/`GE_DIE` server path, config-key lists) to verified by reading them when a
  task needs the detail.
- Keep the "damage is event-delivered + two-stage death + per-bone scaling" notes — all three break
  naive "Hit() = instant death by raw power" assumptions.
