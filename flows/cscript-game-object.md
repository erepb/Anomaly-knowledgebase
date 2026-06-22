---
domain: flows/cscript-game-object
tags: [xrgame, lua, luabind, game_object, callback, binding, flow]
sources: [src/xrGame/script_game_object.h, src/xrGame/script_game_object.cpp, src/xrGame/script_game_object_script.cpp, src/xrGame/script_game_object_use.cpp, src/xrGame/script_game_object_inventory_owner.cpp, src/xrGame/GameObject.h, src/xrGame/game_object_space.h]
related: [flows/lua-callback-lifecycle.md, flows/object-lifecycle.md, flows/relations-info.md, engine/server-entities.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: CScriptGameObject — the Lua `game_object` surface

How the C++ engine exposes a `CGameObject` to Lua as `game_object` (the `obj` mods call
`obj:health()`, `obj:give_info_portion()`, `obj:set_callback()` on). This documents the **binding
architecture** — the wrapper, the split registration, the smart-cast delegation, and the per-object
callback seam — not the ~1000-method catalog. The luabind boot itself is `engine/server-entities.md` /
`flows/lua-callback-lifecycle.md`.

**Provenance:** spine personally code-verified — the `CScriptGameObject` ctor (`m_game_object` +
`R_ASSERT2`), the `script_register` registration **chaining** across partials, and `SetCallback` →
`object().callback(type).set(functor)` (the per-object callback seam). The `BIND_FUNCTION*`
smart-cast-delegation macros, the full method catalog across the partials, and the
`CScriptCallbackEx` storage are confirmed homes but *(agent-traced — not personally re-read)*.

## What it is (verified)

`CScriptGameObject` wraps one `CGameObject*` (`m_game_object`; ctor `R_ASSERT2`s it non-null);
`object()` returns the `CGameObject&`. The back-reference is `CGameObject::lua_game_object()` — so an
engine object and its Lua wrapper point at each other (the binder creates the wrapper for each scripted
object; `flows/object-lifecycle.md`). Lua `game_object` *is* this class.

## Split registration (verified)

The method surface is too large for one translation unit, so `CScriptGameObject::script_register`
(`script_game_object_script.cpp`) creates one `class_<CScriptGameObject>("game_object")` and **chains
it through partial registrars**, each `extern` function taking the `class_<>` rvalue, adding a batch of
`.def`s, and returning it:

`script_register_game_object2( script_register_game_object1( script_register_game_object_trader(
std::move(instance)) ) )` — verified.

Two distinct file families (don't conflate them):
- **registration** partials `script_game_object_script.cpp` / `_script2` / `_script3` /
  `_script_trader` — the luabind `.def` lists.
- **implementation** partials `script_game_object.cpp` / `2` / `3` / `4` + domain files
  `_inventory_owner` / `_use` / `_use2` / `_smart_covers` / `_trader` — the C++ method bodies.

## Delegation pattern (verified ctor/seam; macro agent-traced)

A bound method delegates to the concrete engine type via `smart_cast` on `object()`. Most are generated
by `BIND_FUNCTION*` macros (`smart_cast<Concrete*>(&object())` → call the real method, log if the cast
fails). Representative chain *(macro body agent-traced)*:

`obj:health()` → `CScriptGameObject::GetHealth` → `smart_cast<CEntityAlive*>(&object())->conditions()
.GetHealth()`. Manual (non-macro) examples verified: `GiveInfoPortion` → `smart_cast<CInventoryOwner*>
(&object())->TransferInfo(id,true)` (`flows/relations-info.md`); `Parent()` → `H_Parent()` →
`lua_game_object()` of the parent. So one `game_object` fronts *every* domain (actor, AI, inventory,
relations, tasks, devices, vehicles) by casting to whatever concrete class the underlying object is.

## Per-object callback seam (verified)

This is the per-object half of `flows/lua-callback-lifecycle.md`. Each `CGameObject` holds
`m_callbacks` — a map `ECallbackType → CScriptCallbackEx<void>` (`GameObject.h`); `callback(type)`
returns the slot. `obj:set_callback(callback.xxx, fn[, self])` binds it via
**`CScriptGameObject::SetCallback(type, functor[, object])` → `object().callback(type).set(functor[,
object])`** (verified), and `set_callback(type)` (no fn) clears it. The engine fires it as
`object().callback(eXxx)(lua_game_object(), args…)` from the relevant C++ site (e.g. `callback(
GameObject::eHit)(…)` in the actor). The `ECallbackType` enum (`game_object_space.h`) lists ~60 events
(`eDeath`/`eHit`/`eUseObject`/`eOnWeaponFired`/`eZoneEnter`/`eTaskStateChange`/…). The Anomaly mod layer
bridges common ones to the global bus via `callbacks_gameobject.script` (`modlayer/scripting.md`).

## Isolation ledger (the seams)

- **Every gameplay domain:** the surface delegates (via `smart_cast`) to actor/condition, inventory/
  info/relations (`flows/relations-info.md`), AI memory/movement, weapons/items, devices, anomalies,
  vehicles, doors, tasks (`flows/tasks.md`) — `game_object` is the universal script handle.
- **Per-object callbacks:** `m_callbacks` map; `set_callback` binds, engine fires; the per-object
  mechanism of `flows/lua-callback-lifecycle.md` (vs the global `RegisterScriptCallback` bus the mod
  layer bridges to).
- **Object lifecycle:** the wrapper is created/destroyed with the binder for each scripted object
  (`flows/object-lifecycle.md`); `lua_game_object()` is the engine→Lua back-ref.
- **Lua surface (representative groups):** state (`health`/`power`/`position`/`alive`), inventory/info
  (`give_info_portion`/`has_info`/`money`/`transfer_item`), relations (`set_relation`/`general_goodwill`),
  AI (`set_item`/`best_enemy`/memory), callbacks (`set_callback`), monster/smart-cover specials,
  character (`character_community`/`character_rank`), devices/vehicles/doors.
- **Cast helpers:** `obj:cast_weapon()`/`cast_inventory_item()`/… narrow to a concrete wrapper where a
  domain method isn't on the base surface.

## Threading

Main thread; Lua is single-threaded (`flows/lua-callback-lifecycle.md`). Callbacks fire synchronously
from the C++ site that owns the event.

## Failure & edge cases

- **Wrong concrete type → no-op/log:** a method that `smart_cast`s to a type the object isn't (e.g.
  `obj:health()` on a non-`CEntityAlive`) returns a default and logs — it doesn't crash, but does
  nothing. Mods must call domain methods only on the right object kind (hence the `cast_*` helpers).
- **One functor per (object, callback type):** `set_callback` replaces the previous binding for that
  type on that object; re-binding without clearing drops the old one. Multiple listeners need the mod
  layer's global bus (`callbacks_gameobject.script`).
- **Stale wrapper:** after the object dies, its `CScriptGameObject` is invalid — holding a Lua ref to a
  destroyed object's `game_object` and calling it is a classic mod crash (`is_valid()` guards some
  paths).
- **Registration split is structural:** adding methods means editing the right `_script*` partial; the
  numbered impl partials hold bodies. Don't expect one file to hold the whole class.

## Deeper

- Wrapper + registration: `src/xrGame/script_game_object.h` (`m_game_object`, `object()`),
  `script_game_object_script.cpp` (`script_register` chaining `script_register_game_object1/2/trader`)
- Method bodies: `src/xrGame/script_game_object.cpp`/`2`/`3`/`4` + `_inventory_owner.cpp` (info/
  relations/money), `_use.cpp` (`SetCallback`, use/tip/doors), `_use2.cpp`/`_smart_covers.cpp`/
  `_trader.cpp` (monster/cover/trader specials)
- Callback storage/seam: `src/xrGame/GameObject.h` (`m_callbacks`, `callback(type)`),
  `game_object_space.h` (`ECallbackType`), `script_callback_ex.h` (`CScriptCallbackEx`) — see
  `flows/lua-callback-lifecycle.md`

## Maintenance

- Keep the two-file-family distinction (registration `_script*` vs implementation `.cpp`/`N`) — it's
  the structural fact for finding/adding a bound method.
- Re-grep `CScriptGameObject::SetCallback` and `CGameObject::callback(` if the per-object callback
  storage changes — it is the seam shared with `flows/lua-callback-lifecycle.md`.
- Promote the *(agent-traced)* `BIND_FUNCTION*` macro mechanics and method catalog to verified only if a
  task needs a specific binding's exact delegation; the catalog is intentionally not enumerated here.
