---
domain: engine/xrGame
tags: [engine, cpp, actor, weapons, inventory, ai, ui, luabind, callbacks]
sources: [src/xrGame/, src/xrGame/ai/, src/xrGame/ui/, src/xrGame/Actor.cpp, src/xrGame/Weapon.cpp, src/xrGame/script_game_object.h]
related: [engine/_overview.md, engine/server-entities.md, modlayer/scripting.md, flows/object-lifecycle.md, flows/actor.md, flows/actor-condition.md, flows/weapon-state-machine.md, flows/inventory.md, flows/damage-hit.md, flows/ai-planner.md, flows/ai-perception.md, flows/ai-squad-logic.md, flows/lua-callback-lifecycle.md, flows/weapon-fire-callbacks.md, flows/weapon-hierarchy-addons.md, flows/outfit-helmet.md, flows/detector-artefact.md, flows/upgrade-system.md, flows/anomaly-zones.md, flows/relations-info.md, flows/tasks.md, flows/item-base.md, flows/camera-pp-effectors.md, flows/cscript-game-object.md, flows/consumables-explosives.md, flows/pda-map-news.md, flows/alife-services.md, flows/physics-objects-ragdoll.md, flows/vehicles-holder.md, flows/monster-ai-control.md, flows/stalker-execution.md, flows/smart-cover.md, flows/script-npc-actions.md, flows/agent-squad-combat.md, flows/special-actors.md, flows/level-runtime.md, flows/wallmarks-particles.md, flows/save-load.md, flows/alife-online-offline.md, flows/dialog-trade.md, flows/hud-ui-load.md, flows/animation-callbacks.md, GLOSSARY.md]
updated: 2026-06-05
verified: map
---

# xrGame

The gameplay DLL — largest module (~2686 files, verified by listing). This is a **verified L1 map**,
not a full code digest: file/dir existence and named symbols are verified by reading, and **every
major mechanism AND gameplay subsystem is delegated to a code-verified flow** under `flows/` — the
cross-cutting mechanisms (object lifecycle, actor, weapons fire + state machine, inventory, damage, the
three AI flows, save/load, ALife transition, dialog/trade, HUD/UI load, animation callbacks) **plus**
the concrete subsystems (weapon/outfit/detector/upgrade/zone/consumable-explosive/item-base; actor
condition; relations/tasks; PDA/map; ALife services; physics/ragdoll; vehicles; monster + stalker +
smart-cover + script-control + squad AI; special actors; level runtime; wallmarks/particles; the Lua
`game_object` surface). See the `related:` list. **No residual `(INFERRED)` markers remain** — the last
one (rest of `Level.cpp`) was resolved by `flows/level-runtime.md`. Only multiplayer netcode (OOS) is
undocumented, by design.

## Major areas (existence verified by listing)

- **Actor** — `Actor.cpp/.h` + partials: `Actor_Movement`, `Actor_Weapon`, `Actor_Network`,
  `Actor_Feel`, `Actor_Sleep`, `ActorCondition*`, `ActorInput`, `ActorCameras`, `ActorHelmet`,
  `ActorNightVision`, `ActorBackpack`, `ActorEffector*`. The player lifecycle (spawn; control on the
  deferred channel = input→movement→physics vs HUD/camera on real-time; condition; `eHit`; death) is
  verified in **`flows/actor.md`**.
- **Weapons** — `Weapon.cpp/.h` + `WeaponMagazined.cpp`. Two verified flows: the **state machine**
  (show/idle/fire/reload/zoom/hide, HUD-animation-driven via `UpdateCL`→`OnAnimationEnd`→
  `SwitchState`) in **`flows/weapon-state-machine.md`**, and the **fire→bullet→hit→callback** chain
  (`FireStart`→`state_Fire`→`OnShot`→`FireTrace`→`FireBullet`; `eOnWeaponFired`/`eOnWeaponJammed`/
  `eOnWeaponMagazineEmpty`; misfire/jam) in **`flows/weapon-fire-callbacks.md`**. The **class tree**
  (CWeapon→CWeaponMagazined→pistol/shotgun/SVD/WGrenade/…; leaves are config) and the
  **scope/silencer/GL addon** attach/detach + koeff re-parameterization + ammo/magazine are verified in
  **`flows/weapon-hierarchy-addons.md`**.
- **Inventory** — `Inventory.cpp/.h`, `InventoryOwner*`, `InventoryBox*`. Item take/place(slot/belt/
  ruck)/activate/drop, the deferred active-slot machine, and the `GE_OWNERSHIP_*` event seam are
  verified in **`flows/inventory.md`**. Body armor — outfit/helmet protection (×condition), the
  outfit→helmet damage seam, wear, and passive restore — is **`flows/outfit-helmet.md`**.
- **Level runtime** — `Level.cpp/.h`, `Level_Bullet_Manager.cpp/.h`. `CBulletManager` is verified as
  the ballistics pipeline (`AddBullet`→`CommitEvents`→`UpdateWorkload`→`process_bullet`→`RayQuery`,
  parabolic ballistics, optional MT via `Device.seqParallel`) — see `flows/weapon-fire-callbacks.md`.
  The **rest of `CLevel`** (per-frame object-collection tick, network spawn/event processing, map/task
  update, input routing, level load) is verified in **`flows/level-runtime.md`**; gameplay decals/
  particles via material pairs are **`flows/wallmarks-particles.md`**.
- **AI** (`ai/`) — subfolders `stalker/`, `monsters/`, `crow/`, `phantom/`, `trader/`. Three verified
  flows: the think loop + GOAP planner (`CAI_Stalker::Think`→`brain().update()` → A* over
  `StalkerDecisionSpace`) in **`flows/ai-planner.md`**; sense→memory (Feel vision/sound/hit →
  `CMemoryManager` → enemy/danger selection) in **`flows/ai-perception.md`**; and the Lua
  smart_terrain→`xr_logic`→binder orchestration in **`flows/ai-squad-logic.md`**. Hit/damage/death
  (shared by all creatures) is **`flows/damage-hit.md`**. **Monsters use a different architecture** —
  a per-species FSM (`StateMan`) + pluggable control elements, *not* GOAP — verified in
  **`flows/monster-ai-control.md`**. NPC combat *execution* (move/aim/fire below the planner) is
  **`flows/stalker-execution.md`**; smart-cover use **`flows/smart-cover.md`**; script-control
  **`flows/script-npc-actions.md`**; squad coordination **`flows/agent-squad-combat.md`**; the special
  actors (trader/crow/phantom, none GOAP/FSM) are **`flows/special-actors.md`**.
- **UI** (`ui/`, 312 files, verified count) — XML-driven; `ui/UIXmlInit.cpp` confirmed present
  (the DXML round-trip runs around XML load — see `flows/dxml-xml-interception.md`).
- **Lua object binding** — `CScriptGameObject` class in `script_game_object.h` (+ `2/3/4`
  partials) is the Lua `game_object` wrapper; `script_engine_help.cpp` assists. The script engine
  itself lives in `xrServerEntities` (`engine/server-entities.md`). The wrapper architecture (split
  registration, smart_cast delegation, per-object `set_callback` seam) is verified in
  **`flows/cscript-game-object.md`**.

## Object lifecycle (the backbone)

Every gameplay class derives from **`CGameObject : public CObject, CUsableScriptObject, CScriptBinder`**
(`GameObject.h`). Its spawn → per-frame (real-time `UpdateCL` vs deferred `shedule_Update`) → destroy
spine, and how it wires into the scheduler, ALife server entities, and the Lua binder, is the verified
**`flows/object-lifecycle.md`** — read that before any per-domain xrGame work; the domain digests below
attach to its spine.

## Engine → Lua callbacks (corrected, verified)

xrGame uses **per-object** callbacks, not the global bus directly. C++ fires them as
`callback(GameObject::eEnum)(args)` — verified in `Actor.cpp` (`eHit`, `eExitLevelBorder`,
`eEnterLevelBorder`). These are bound from Lua per object via `obj:set_callback(callback.xxx,
fn)`. The script `gamedata/scripts/callbacks_gameobject.script` bridges common per-object callbacks
into the global `RegisterScriptCallback`/`SendScriptCallback` bus (see
`flows/lua-callback-lifecycle.md`, `modlayer/scripting.md`). *(Previously this file wrongly said
xrGame fires `SendScriptCallback` directly.)*

When adding a feature, decide: pure engine; engine + a new per-object `GameObject::e*` callback;
engine + a global bus callback; or a new value read from config (`flows/dltx-config-load.md`).

## Deeper

- Actor: `src/xrGame/Actor*.cpp`, callback sites grep `callback(GameObject::` in `Actor.cpp`
- Weapons: `src/xrGame/Weapon*.cpp`, misfire grep `Misfire`/`misfire` in `src/xrGame/`
- AI: `src/xrGame/ai/stalker/`, `src/xrGame/ai/monsters/`
- UI + XML init: `src/xrGame/ui/`, `src/xrGame/ui/UIXmlInit.cpp`, `src/xrGame/ScriptXMLInit.cpp`
- Lua object wrapper: `src/xrGame/script_game_object*.cpp/.h`, `*_script.cpp` files

## Maintenance

- Prefer grepping a symbol/class over file lists here; this module is too large to enumerate.
- Promote an *(INFERRED)* item to verified only after reading the relevant `file:symbol`; cite it.
- Keep the callback correction intact — the per-object `GameObject::e*` mechanism vs the global Lua
  bus is a real distinction (bridge in `callbacks_gameobject.script`).
