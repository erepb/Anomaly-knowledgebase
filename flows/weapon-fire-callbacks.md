---
domain: flows/weapon-fire-callbacks
tags: [engine, cpp, weapons, actor, callbacks, flow, lifecycle]
sources: [src/xrGame/WeaponMagazined.cpp, src/xrGame/Weapon.cpp, src/xrGame/WeaponFire.cpp, src/xrGame/Level_Bullet_Manager.cpp, src/xrGame/Actor.cpp, src/xrGame/script_game_object_script.cpp]
related: [engine/xrGame.md, flows/lua-callback-lifecycle.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: weapon fire → bullet → hit → callbacks

The firing path for a magazined weapon and the per-object Lua callbacks it raises. The full chain is
now verified by reading the bodies: per-shot call graph
(`state_Fire`→`OnShot`→`FireTrace`→`FireBullet`→`AddBullet`) and the bullet simulation/collision
(`CommitEvents`→`UpdateWorkload`→`process_bullet`→`RayQuery`→`firetrace_callback`→`eHit`). Line numbers
are current-as-of-`updated` hints.

## Trigger / entry

Trigger pulled → `CWeaponMagazined::FireStart()` (`WeaponMagazined.cpp`), which validates and
calls `inherited::FireStart()`. The firing state advances in `state_Fire(dt)`,
which calls `OnShot()` per shot.

## Sequence (verified call-sites)

1. **`FireStart()`** — begin firing.
2. **`state_Fire(float dt)`** — firing-state loop. While the magazine has rounds and the
   queue isn't exhausted, per shot it: `++m_iShotNum`, **`CheckForMisfire()`**,
   **`OnShot()`**, then **`FireTrace(p1,d)`** (or `m_vStartPos/Dir` for the first
   `m_iBaseDispersionedBulletsCount` shots). On `bMisfire` it raises `eOnWeaponJammed` and
   `StopShooting()`.
3. **`OnShot()`** — per-shot effects, verified in order: `PlaySoundShot`, `AddShotEffector`
   (camera), `PlayAnimShoot`, `HUD_VisualBulletUpdate`, `OnShellDrop` (shell eject),
   `StartFlameParticles`/`StartSmokeParticles`, then fires the Lua callback
   **`eOnWeaponFired(parent, weapon, iAmmoElapsed)`** (under `EXTENDED_WEAPON_CALLBACKS`),
   then `AddHUDShootingEffect` (recoil). Effects only — does **not** spawn the bullet.
4. **`CWeapon::FireTrace(P,D)`** (`WeaponFire.cpp`) — the actual bullet-spawn step: sets tracer
   flags on the cartridge, applies weapon wear (`ChangeCondition(-deterioration…)`), computes
   `fire_disp`, then **loops `l_cartridge.param_s.buckShot` times** calling **`FireBullet(...)`**
   — one projectile per pellet (shot/buckshot fires many). Then `StartShotParticles`,
   muzzle `Light_Start`, grass-bender FX, pops the cartridge and decrements `iAmmoElapsed`.
5. **`FireBullet(pos, dir, disp, cartridge, parent_id, weapon_id, send_hit, ammo)`**
   (`WeaponMagazined.cpp` → `inherited::FireBullet` = base `CWeapon`) — spawns the
   projectile into the bullet manager.
6. **`CBulletManager::AddBullet(position, …)`** (`Level_Bullet_Manager.cpp`) — registers the
   bullet into `m_Bullets` (guarded by `m_Lock` critical section).
7. **Per-frame simulation** — `CommitEvents()` (start of frame) drains queued add/remove/hit
   events, then dispatches the update: if `g_mt_config.test(mtBullets)` it pushes
   `UpdateWorkload` onto `Device.seqParallel` (runs on an **MT worker thread**), else calls it
   inline. `UpdateWorkload()` holds `m_Lock`, **reverse-iterates** `m_Bullets`
   (removal swaps an index via `tgt_material`, so reverse order is required) and calls
   `process_bullet(rq_storage, bullet, time_delta * g_bullet_time_factor)`; a false return queues an
   `EVENT_REMOVE`. Ballistics integrate gravity + air resistance (`parabolic_velocity`,
   `trajectory_velocity`).
8. **Collision** — `process_bullet` ray-queries the bullet segment via
   `Level().ObjectSpace.RayQuery(…, firetrace_callback, test_callback, …)`; `firetrace_callback`
   resolves the hit.
9. **Hit** → the hit object's per-object callback **`callback(GameObject::eHit)(...)`** fires —
   verified at `Actor.cpp` (actor hit). Other objects raise their own `eHit`.

## Misfire / jam (verified)

- Misfire probability: **`CWeapon::GetConditionMisfireProbability()`** (`Weapon.cpp`) — derived
  from weapon condition vs `misfire_start_condition`/`misfire_end_condition` and
  `misfire_start_prob`/`misfire_end_prob` (read at `Weapon.cpp`).
- On a jam, `bMisfire` is set; `OnEmptyClick()` (`WeaponMagazined.cpp`) plays the misfire sound;
  the Lua callback **`eOnWeaponJammed(parent, weapon)`** fires.
- Lua side: `gamedata/scripts/xr_weapon_jam.script` (the `npc_get_misfire_probability` hook lets
  scripts set NPC misfire probability — see `engine/xrGame.md`).

## Other per-shot weapon callbacks (verified invocation sites)

- `eOnWeaponMagazineEmpty(weapon, ammo_class)` — `WeaponMagazined.cpp`.
- `eOnWeaponZoomIn(parent, weapon)` (zoom, not fire, but same per-object mechanism).
- All are **per-object** callbacks (`object->callback(GameObject::e*)(...)`), bound from Lua via
  `set_callback`; the Lua enum mapping is in `script_game_object_script.cpp` (e.g. `weapon_fired`→
  `GameObject::eOnWeaponFired`). Bridging to the global bus: `callbacks_gameobject.script`
  (see `flows/lua-callback-lifecycle.md`).

## State touched

- Weapon: `iAmmoElapsed`, `bMisfire`, condition (drives misfire prob).
- `CBulletManager` bullet list (`m_Lock`-guarded).

## Threading

`CBulletManager` uses `m_Lock` (critical section). In the MT build (`g_mt_config.test(mtBullets)`) the
bullet `UpdateWorkload` runs on a `Device.seqParallel` worker thread; otherwise inline. Fire-state
logic (`state_Fire`/`OnShot`/`FireTrace`) runs on the weapon's update (main thread).

## Failure & edge cases

- Misfire/jam short-circuits the shot (no bullet) → `eOnWeaponJammed`; empty magazine → empty click /
  `eOnWeaponMagazineEmpty`.
- The `eOnWeaponFired` callback is gated by `EXTENDED_WEAPON_CALLBACKS` (compile define).

## Deeper

- Fire path: `src/xrGame/WeaponMagazined.cpp` (`FireStart`, `state_Fire`, `OnShot`, `FireBullet`,
  `OnEmptyClick`); bullet-spawn `src/xrGame/WeaponFire.cpp` (`CWeapon::FireTrace`, buckShot loop)
- Misfire: `src/xrGame/Weapon.cpp` (`GetConditionMisfireProbability`, misfire params)
- Bullets: `src/xrGame/Level_Bullet_Manager.cpp` (`AddBullet`, `CommitEvents`, `UpdateWorkload`,
  `process_bullet`, `parabolic_velocity`/`trajectory_velocity`, `RayQuery`+`firetrace_callback`)
- Hit callback: `src/xrGame/Actor.cpp` (`callback(GameObject::eHit)`)
- Callback enum mapping: `src/xrGame/script_game_object_script.cpp`

## Maintenance

- Full chain (fire→spawn→sim→collision→hit) is verified. If lines drift, re-grep
  `callback(GameObject::eOnWeapon`, `FireTrace`, `UpdateWorkload`, `process_bullet`, and
  `GetConditionMisfireProbability`.
