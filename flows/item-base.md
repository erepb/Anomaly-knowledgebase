---
domain: flows/item-base
tags: [xrgame, item, inventory, physics, attach, condition, hierarchy, flow]
sources: [src/xrGame/inventory_item.h, src/xrGame/inventory_item.cpp, src/xrGame/attachable_item.h, src/xrGame/attachable_item.cpp, src/xrGame/physic_item.h, src/xrGame/physic_item.cpp, src/xrGame/inventory_item_object.h, src/xrGame/inventory_item_object.cpp, src/xrGame/hud_item_object.h]
related: [flows/inventory.md, flows/weapon-hierarchy-addons.md, flows/outfit-helmet.md, flows/detector-artefact.md, flows/object-lifecycle.md, engine/physics-collision.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: item base hierarchy

The class foundation every gameplay item sits on, and the **ownership transitions** (in-hand ↔
in-inventory ↔ on-ground ↔ bone-attached) that toggle its physics and visibility. This is the tree
*under* `flows/inventory.md` (take/place/activate/drop) and under weapons/outfits/artefacts/detectors
(F22–F24). Read this to know which base a new item type should pick and what `OnH_*` does.

**Provenance:** spine personally code-verified — the inheritance tree (all class headers),
`CPhysicItem::OnH_B_Independent`/`OnH_B_Chield` (the physics activate/deactivate on drop/pickup),
`CAttachableItem::enable` (bone attach via `CAttachmentOwner`), and `CInventoryItemObject`'s diamond
resolution (calls both `CPhysicItem::` and `CInventoryItem::` `OnH_*`). The `CInventoryItem::Load`
param list, `ChangeCondition`/`Hit` (condition math), `net_Spawn`/save, and the full member set are
confirmed homes but *(agent-traced — not personally re-read)*.

## Inheritance tree (verified from headers)

- **`CAttachableItem`** (`attachable_item.h`) — standalone mixin (no `CObject`); holds a back-ref
  `CInventoryItem* m_item`, a bone name/id, an offset matrix, and `m_enabled`. Provides **bone
  attachment** of a whole item's visual to a parent's skeleton (helmet on head, detector on belt).
- **`CInventoryItem`** (`inventory_item.h`: `class CInventoryItem : public CAttachableItem, public
  CHitImmunity`) — the **"can be carried"** capability base. **Not a `CGameObject`** — a mixin a real
  object inherits. Holds `m_pInventory`, `m_section_id`, name/cost/weight, `m_ItemCurrPlace`
  (slot/belt/ruck), `m_fCondition`, flags, `m_upgrades`. `CHitImmunity` gives it the per-hit-type
  immunity used by armor/artefacts.
- **`CPhysicItem`** (`physic_item.h`: `class CPhysicItem : public CPhysicsShellHolder, public
  CPHShellSimpleCreator`) — the **"has a physics body in the world"** base; `CPhysicsShellHolder` is
  the actual `CGameObject` chain. Owns `m_pPhysicsShell`.
- **`CInventoryItemObject`** (`inventory_item_object.h`: `: public CInventoryItem, public CPhysicItem`)
  — the **diamond**: a concrete game object that is both carriable *and* a physics object. Most
  items' real base. (Outfits descend here: `CCustomOutfit : CInventoryItemObject`.)
- **`CHudItemObject`** (`hud_item_object.h`: `: public CInventoryItemObject, public CHudItem`) — adds
  first-person HUD model + the HUD-item animation/state spine (`flows/weapon-state-machine.md`,
  `flows/animation-callbacks.md`). Weapons/torches/artefacts descend here (`CWeapon`, `CArtefact`).
- **`CEatableItem`** (`eatable_item.h`: `: public CInventoryItem`) — a carriable mixin that delegates
  its world physics to a separate `CPhysicItem*` (the food-item gimmick; F25 covers the use path).

So a concrete item picks: **carriable + physics** → `CInventoryItemObject`; **+ first-person HUD** →
`CHudItemObject`. The condition system and bone-attach come "for free" from `CInventoryItem`/
`CAttachableItem`.

## Condition (shared durability) (agent-traced math)

`m_fCondition` (`[0,1]`) is the universal durability metric on every `CInventoryItem`.
`GetCondition`/`SetCondition` accessors; `ChangeCondition(delta)` clamps to `[0,1]`; `Hit(SHit*)` —
when `IsUsingCondition()` (the `use_condition` `.ltx` flag) — wears the item by
`ChangeCondition(-hit·GetHitImmunity(type))`. There is **no time-based decay**; condition is
event-driven (firing, taking hits). This is the same `× GetCondition()` factor weapons scale
dispersion by and outfits/artefacts scale protection/restore by (`flows/outfit-helmet.md`,
`flows/detector-artefact.md`). *(Load/Hit bodies agent-traced; the `× GetCondition()` consumption is
verified in those flows.)*

## Ownership transitions — the `OnH_*` callbacks (verified)

`CObject::H_SetParent` fires four item-side callbacks when an item changes parent; they are the
item-side of the inventory `GE_OWNERSHIP_*` events (`flows/inventory.md`). On the diamond
(`CInventoryItemObject`) each calls **both** bases in order:

- **`OnH_B_Independent` (dropped to the world):** `CPhysicItem::OnH_B_Independent` →
  `setVisible(TRUE)` + `setEnabled(TRUE)` + **`activate_physic_shell()`** (unless `m_ready_to_destroy`
  or `just_before_destroy`). The item becomes a live physics body — collides, falls, can be picked up.
- **`OnH_B_Chield` (picked up / put in inventory):** `CPhysicItem::OnH_B_Chield` →
  `setVisible(FALSE)` + `setEnabled(FALSE)` + **`deactivate_physics_shell()`**. The item freezes and
  hides — it lives logically in the inventory, not the world.
- **`OnH_A_Chield` (attached to a parent's bone, e.g. equipped/held):** routes to
  `CAttachableItem::OnH_A_Chield` — if the parent has marked it attached, make it visible at the bone.
- **`OnH_A_Independent` (detached from a parent bone):** routes to `CAttachableItem::OnH_A_Independent`
  → `enable(false)` (hide + detach from bone), and stamps the "just went independent" despawn timer.

## Bone attach — `CAttachableItem::enable` (verified)

`enable(value)`: if the item has no parent, just store the flag. If `value && parent`: `smart_cast` the
parent to `CAttachmentOwner` and `owner->attach(&item())` — this parents the item's 3D model to a
named skeleton **bone** (the offset matrix from `attach_position_offset`/`attach_angle_offset`/
`attach_bone_name` in `.ltx`). `enable(false)` calls `owner->detach`. This is **distinct from weapon
addon attach** (`flows/weapon-hierarchy-addons.md`): that installs one item *inside* another's addon
slots; this hangs a whole item's visual on a bone (helmet, detector, torch, PDA).

## Isolation ledger (the seams)

- **Inventory:** `m_pInventory`/`m_ItemCurrPlace`; the take/place/drop machine fires the `OnH_*`
  callbacks via `GE_OWNERSHIP_*` (`flows/inventory.md`).
- **Physics:** `CPhysicItem` owns the shell; drop activates it, pickup deactivates it
  (`engine/physics-collision.md`). `UpdateCL` interpolates the render transform from the shell when in
  the world.
- **Damage/condition:** `m_fCondition` + `Hit`/`ChangeCondition` + `CHitImmunity` — the shared
  durability consumed by weapon/outfit/artefact flows.
- **Bone attach / HUD:** `CAttachmentOwner` (parent) ↔ `CAttachableItem` (item) for worn/held visuals;
  `CHudItemObject`/`CHudItem` for the first-person model.
- **Upgrades:** `m_upgrades` + `install_upgrade*` live on `CInventoryItem` (`flows/upgrade-system.md`).
- **Lua:** `CInventoryItem`'s own `script_register` is essentially empty — items are reached from Lua
  via `obj:cast_inventory_item()`/`cast_weapon()`/… on the `game_object` wrapper
  (`flows/cscript-game-object.md`); per-type callbacks live on subclasses.
- **Config (`Load`, agent-traced list):** `inv_name`/`inv_name_short`, `inv_weight`, `cost`, `slot`,
  `description`, `can_trade`/`can_take`/`belt`, `use_condition`, `quest_item`, `default_to_ruck`,
  `sprint_allowed`, `control_inertion_factor`, `icon_name`, `power_critical`, `immunities_sect`, and
  the `attach_*_offset`/`attach_bone_name` bone params.

## Threading

Main thread (inventory/physics-shell/HUD). The physics shell itself is stepped by `xrPhysics`
(`engine/physics-collision.md`); the item interpolates its render transform on the main thread.

## Failure & edge cases

- **Drop = physics on, pickup = physics off:** a "frozen" dropped item or a still-colliding carried
  item is almost always a missed `OnH_B_*` transition (e.g. `m_ready_to_destroy` set, or
  `just_before_destroy` skipping `activate_physic_shell`).
- **Diamond order matters:** `CInventoryItemObject` calls `CPhysicItem::` then `CInventoryItem::` (or
  vice-versa per callback) — overriding only one base, or in the wrong order, leaves visibility/physics
  and inventory-place out of sync.
- **`use_condition` gates wear:** without it, `m_fCondition` stays at 1.0 and `Hit` doesn't wear the
  item — a frequent "why doesn't my armor degrade" config cause.
- **Bone attach ≠ addon attach:** confusing `CAttachableItem::enable` (bone visual) with weapon
  `Attach` (addon slot) is a common mistake; they are unrelated mechanisms.
- **`CInventoryItem` isn't an object:** you can't `smart_cast` a raw `CInventoryItem*` to `CGameObject*`
  directly — go through `object()`/the concrete type.

## Deeper

- Base item: `src/xrGame/inventory_item.cpp`/`.h` (`Load`, `ChangeCondition`, `Hit`, `GetCondition`,
  `OnH_*`, `m_fCondition`/`m_ItemCurrPlace`/`m_upgrades`)
- Bone attach: `src/xrGame/attachable_item.cpp`/`.h` (`enable`, `load_attach_position`, `OnH_A_Chield`/
  `OnH_A_Independent`), `attachment_owner.h` (`CAttachmentOwner::attach`/`detach`)
- Physics item: `src/xrGame/physic_item.cpp`/`.h` (`OnH_B_Independent`, `OnH_B_Chield`,
  `activate_physic_shell`, `create_physic_shell`)
- Diamond: `src/xrGame/inventory_item_object.cpp`/`.h` (`OnH_*` calling both bases)
- HUD base: `src/xrGame/hud_item_object.h` (`CHudItemObject`), `HudItem.h` (`CHudItem`)
- Consumers: `flows/inventory.md`, `flows/weapon-hierarchy-addons.md`, `flows/outfit-helmet.md`,
  `flows/detector-artefact.md`, `flows/upgrade-system.md`

## Maintenance

- Re-grep the class declarations (`^class CInventoryItem`, `inventory_item_object.h`,
  `hud_item_object.h`) if the tree changes — note the multiple-inheritance/diamond (decls span lines).
- Keep the "drop activates physics / pickup deactivates" `OnH_*` facts and the bone-attach-vs-addon
  distinction accurate — both are frequent confusion points.
- Promote the *(agent-traced)* `Load` param list + `ChangeCondition`/`Hit` math to verified when an
  item-authoring task needs the exact fields/clamps.
