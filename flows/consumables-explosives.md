---
domain: flows/consumables-explosives
tags: [xrgame, consumable, eatable, explosive, grenade, missile, bolt, flow]
sources: [src/xrGame/eatable_item.cpp, src/xrGame/eatable_item.h, src/xrGame/FoodItem.cpp, src/xrGame/Explosive.cpp, src/xrGame/Explosive.h, src/xrGame/Grenade.cpp, src/xrGame/Grenade.h, src/xrGame/Missile.cpp, src/xrGame/Missile.h, src/xrGame/Bolt.cpp]
related: [flows/item-base.md, flows/actor-condition.md, flows/damage-hit.md, flows/weapon-fire-callbacks.md, flows/inventory.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: consumables & explosives

Two item-use families: **consumables** (food/meds/drinks — `use` applies condition influence + timed
boosters then decrements uses) and **throwables/explosives** (grenades/bolts/missiles — throw a flying
clone, then a grenade detonates an area blast). Consumables feed the actor condition tick
(`flows/actor-condition.md`); explosions feed the damage pipeline (`flows/damage-hit.md`) via fragments
(the bullet manager, `flows/weapon-fire-callbacks.md`) and a blast wave. Both build on the item base
(`flows/item-base.md`).

**Provenance:** spine personally code-verified — the inheritance (`CGrenade : CMissile, CExplosive`;
`CMissile : CHudItemObject`; `CExplosive : IDamageSource`; `CBolt : CMissile, IDamageSource`;
`CEatableItem : CInventoryItem`) and the existence of `CEatableItem::UseBy`, `CExplosive::Explode`,
`CMissile::Throw`. The `UseBy` body (influence/booster/uses), `Explode` (fragments + blast wave), the
throw `spawn_fake_missile`/`GE_OWNERSHIP_REJECT` flow, the grenade cook timer, `CBottleItem` break, and
the optional `EXPLOSIVE_CHANGE` Lua callbacks are confirmed homes but *(agent-traced — not personally
re-read)*.

## Consumables

### Type tree (verified)

- **`CEatableItem`** ← `CInventoryItem` — the "use to apply effects" mixin; a separate `CPhysicItem*`
  holds its world physics (`flows/item-base.md`). Holds `m_iMaxUses`/`m_iRemainingUses`.
  - **`CFoodItem`** (eat animation/sound) → **`CBottleItem`** (drink; breaks on hard impact). *(agent-traced.)*

### Consume sequence (agent-traced; condition seam verified-by-reference)

`CEatableItem::UseBy(CEntityAlive*)` (triggered from the inventory UI / Lua use):
1. `SMedicineInfluenceValues::Load(section)` reads `eat_health`/`eat_power`/`eat_satiety`/`eat_radiation`/
   `wounds_heal_perc`/`eat_max_power_up`/`eat_alcohol`/`eat_time`.
2. `entity->conditions().ApplyInfluence(values, section)` — the immediate health/power/satiety/radiation/
   wound-heal change (the `Change*` accumulators of `flows/actor-condition.md`).
3. for each `boost_*` section present, build an `SBooster` and `entity->conditions().ApplyBooster(B,
   section)` — the timed boost (same flow). (MP also fires `GEG_PLAYER_USE_BOOSTER`.)
4. decrement `m_iRemainingUses` (skip if `-1` = infinite); scale `m_fCondition`/weight; when empty and
   `remove_after_use`, the item is destroyed (`OnH_A_Independent`).

`CBottleItem` additionally overrides `Hit` — a hit above a break threshold fires a break (particles +
sound, `GE_GRENADE_EXPLODE`-style local destroy). *(agent-traced.)*

## Throwables & explosives

### Type tree (verified)

- **`CMissile`** ← `CHudItemObject` — the **throwable base**: a HUD item that, on throw, spawns a
  physics-enabled **fake missile** clone that flies. `CBolt : CMissile, IDamageSource` is the simplest
  (kinetic only, optionally reusable).
- **`CExplosive`** ← `IDamageSource` — a **mixin** (not a `CObject`) providing `Explode()` (blast +
  fragments). Combined into:
- **`CGrenade` : `CMissile, CExplosive`** — a thrown HUD item *and* an explosive: throw → cook timer →
  `Explode`. (`CExplosiveItem` is the placed/timed explosive variant.)

### Grenade throw → cook → explode (agent-traced; entries verified)

1. **Cook/throw:** holding the throw key runs the HUD state machine (`flows/weapon-state-machine.md`):
   `eThrowStart` → `eReady` (accumulate `m_fThrowForce` from `force_min`→`force_max` by
   `force_grow_speed`) → on release, the throw animation mark calls **`CMissile::Throw`**:
   `setup_throw_params` (parent fire pos/dir), `spawn_fake_missile` (clone with physics), transfer the
   force/direction, and **`GE_OWNERSHIP_REJECT`** to detach from inventory. The clone is now an
   independent flying physics object (`flows/item-base.md` `OnH_B_Independent`).
2. **Detonate trigger:** the grenade's `destroy_time` cook timer (`shedule_Update` vs
   `Level().timeServer()`) — or, with the optional `explode_on_contact` build, a surface-contact
   callback — calls `Destroy` → `CExplosive::GenExplodeEvent(pos, normal)` → broadcasts
   `GE_GRENADE_EXPLODE`.
3. **`CExplosive::Explode`** (on the event): play sound/particles/light + wallmarks; **fragments** — for
   `frags` count, `Level().BulletManager().AddBullet(...)` a ballistic shard (`frag_hit`/`fragment_speed`/
   `hit_type_frag`, into the bullet pipeline `flows/weapon-fire-callbacks.md`); **blast wave** — spatial
   sphere query (`blast_r`) collects objects, then `ExplodeWaveProcess` (a few per frame) raycasts for
   occlusion and sends each a `GE_HIT` `SHit` of `hit_type_blast` (default explosion) scaled by distance
   + occlusion, plus a physics impulse (`blast_impulse`). So a grenade deals damage **two ways**:
   line-of-sight blast hits + ballistic fragments.

`CBolt::Throw` overrides to set a short destroy time and (via `_G.CBolt__State` Lua) decide whether the
bolt is reusable; no explosion. *(agent-traced.)*

## Isolation ledger (the seams)

- **Condition/boosters:** `UseBy` → `conditions().ApplyInfluence`/`ApplyBooster`
  (`flows/actor-condition.md`).
- **Damage:** `Explode` → `GE_HIT` blast (`flows/damage-hit.md`) + `BulletManager.AddBullet` fragments
  (`flows/weapon-fire-callbacks.md`); outfit/helmet protection for `hit_type_blast`/`hit_type_frag`
  applies (`flows/outfit-helmet.md`).
- **Physics:** throw spawns a physics clone (`spawn_fake_missile`), blast applies impulses
  (`engine/physics-collision.md`); the in-hand→flying transition is the item-base `OnH_B_Independent`.
- **Inventory:** `GE_OWNERSHIP_REJECT` detaches a thrown grenade; consumables decrement/destroy
  (`flows/inventory.md`).
- **Lua (optional `EXPLOSIVE_CHANGE` build):** `on_grenade_explode`/`_G.CExplosiveItem__OnExplode`/
  `_G.CBolt__State` functors; otherwise no script callback on use/explode.
- **Config:** consumable `eat_*`/`boost_*`/`max_uses`/`remove_after_use`; explosive `blast`/`blast_r`/
  `blast_impulse`/`frags`/`frag_hit`/`fragment_speed`/`hit_type_blast`/`hit_type_frag`/`explode_particles`/
  `light_*`/`explode_duration`; throwable `force_min/const/max`/`force_grow_speed`/`destroy_time`/
  `throw_point`/`throw_dir`/`detonation_threshold_hit`.

## Threading

Main thread (use/throw/explode/condition). Fragments ride the bullet manager (optionally MT —
`flows/weapon-fire-callbacks.md`); the physics clone is stepped by `xrPhysics`.

## Failure & edge cases

- **Empty consumable destroyed:** once `m_iRemainingUses` hits 0 (and `remove_after_use`), the item is
  removed; `-1` uses = infinite (debug/special items).
- **Two damage channels:** a grenade hurts via blast (occlusion-checked, cover protects) *and*
  fragments (ballistic, can ricochet/penetrate) — tuning one without the other surprises.
- **Cook timer is server-time:** detonation is `Level().timeServer()`-based; the thrown clone, not the
  in-hand item, explodes (the original is detached via `GE_OWNERSHIP_REJECT`).
- **Contact explosion is build-gated:** `explode_on_contact` only exists in the `EXPLOSIVE_CHANGE`
  build; default grenades are timer-only.
- **Bottle break threshold:** a dropped/hit bottle breaks only above its break power — low hits don't.

## Deeper

- Consumables: `src/xrGame/eatable_item.cpp` (`UseBy`, `Useful`, `Weight`, `OnH_A_Independent`),
  `FoodItem.cpp`, `BottleItem.cpp` (break), `eatable_item_object.cpp`
- Explosive core: `src/xrGame/Explosive.cpp` (`Explode`, `ExplodeWaveProcess`, `GenExplodeEvent`,
  `StartLight`), `Explosive.h` (`m_fBlast*`/`m_iFrags*`/`m_eHitType*`)
- Throwables: `src/xrGame/Missile.cpp` (`Throw`, `spawn_fake_missile`, `shedule_Update`),
  `Grenade.cpp` (`Throw`, `Destroy`, cook timer), `Bolt.cpp` (`Throw`, `_G.CBolt__State`)
- Consumers: `flows/actor-condition.md`, `flows/damage-hit.md`, `flows/weapon-fire-callbacks.md`,
  `flows/item-base.md`

## Maintenance

- Re-grep `CExplosive::Explode` if the blast/fragment split changes — the "two damage channels" fact is
  the key gameplay behavior.
- Keep the `UseBy` → `conditions().ApplyInfluence`/`ApplyBooster` seam accurate (shared with
  `flows/actor-condition.md`).
- Promote the *(agent-traced)* `UseBy`/`Explode`/`Throw` bodies and the `EXPLOSIVE_CHANGE` callbacks to
  verified when a consumable/explosive-tuning task needs the exact math/params.
