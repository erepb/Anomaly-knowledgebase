---
domain: flows/inventory
tags: [engine, cpp, inventory, items, flow, callbacks, alife]
sources: [src/xrGame/Inventory.h, src/xrGame/Inventory.cpp, src/xrGame/inventory_item.cpp, src/xrGame/inventory_item.h, src/xrGame/InventoryOwner.cpp, src/xrGame/Actor_Events.cpp, src/xrGame/script_game_object_inventory_owner.cpp]
related: [flows/object-lifecycle.md, flows/actor.md, flows/weapon-state-machine.md, engine/xrGame.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: inventory & items (take / place / activate / drop)

How items enter an owner's inventory, move between the three containers (slot / belt / ruck), become
the active in-hands item, and leave. Items and owners are ordinary game objects (see
`flows/object-lifecycle.md`); this flow is the **container logic** and the **ownership event seam**
layered on top.

**Provenance:** researched with a read-only agent, then spot-verified against code. The `CInventory::
Take` placement cascade and the `Activate` deferred active-slot state machine are personally
code-verified. The container helpers (`Slot`/`Belt`/`Ruck`/`DropItem`), the owner callbacks, the
`Actor_Events` ownership handlers, and the script bridge are *(agent-traced — not personally
re-read)* and marked.

## The three actors

- **`CInventory`** (`Inventory.h`/`.cpp`) — the container, owned by a `CInventoryOwner`. Holds
  `m_all` (everything) plus three placements: `m_slots[]` (equipment slots), `m_belt`
  (quick-access), `m_ruck` (backpack, effectively unlimited). Tracks the active-slot machine
  (`m_iActiveSlot` / `m_iNextActiveSlot` / `m_iPrevActiveSlot`).
- **`CInventoryItem`** (`inventory_item.cpp/.h`) — the item mixin; carries `m_pInventory`
  back-pointer, `m_ItemCurrPlace` (which container + slot id), weight/cost/condition, and the
  `BaseSlot()`/`RuckDefault()`/`Belt()` capability flags from its `.ltx`.
- **`CInventoryOwner`** (`InventoryOwner.cpp`) — actor/NPC base that holds the `CInventory` and
  receives the `OnItem*` callbacks. The actor reaches it via its inheritance (`flows/actor.md`).

## Take — add an item (verified)

**`CInventory::Take(CGameObject* pObj, bool bNotActivate, bool strict_placement)`** (verified body):

1. Cast to `CInventoryItem`; assert `m_pInventory == NULL` and `CanTakeItem`. Set `m_pInventory =
   this`, clear manual-drop, allow trade, remove from the correction-prediction list
   (`RemoveObject_From_4CrPr`), push into `m_all`.
2. **Placement.** If not `strict_placement`, reset the item's place to `eItemPlaceUndefined` so the
   cascade decides. Then switch on the item's `m_ItemCurrPlace.type` (Belt → `Belt()`, Ruck →
   `Ruck()`, Slot → `Slot()`); on failure reset to undefined.
3. **Fallback cascade** (if still undefined): if the item is `RuckDefault()` → `Ruck()`; else try
   `Slot(BaseSlot())` if `CanPutInSlot`, else `Belt()` if `CanPutInBelt`, else `Ruck()`. (Each
   fallback is `VERIFY`-asserted to succeed — ruck is the always-works terminal.)
4. `m_pOwner->OnItemTake(pIItem)` → `CalcTotalWeight` → `InvalidateState` →
   `processing_deactivate`. If a game UI is up and this owner is the viewed/searched one, fire the UI
   `OnInventoryAction(pIItem, GE_OWNERSHIP_TAKE)`.

So **ruck is the guaranteed terminal placement** and the item's stored place is honored first, then a
slot→belt→ruck preference cascade.

## Place — move between containers (agent-traced)

`CInventory::Slot(slot_id, item, …)`, `Belt(item, …)`, `Ruck(item, …)` each: validate via
`CanPutInSlot`/`CanPutInBelt`/`CanPutInRuck`, remove the item from its current container, insert into
the target, update `m_ItemCurrPlace`, fire the matching owner callback (`OnItemSlot`/`OnItemBelt`/
`OnItemRuck`), and toggle the item's engine processing (`processing_activate` when equipped into a
slot, `processing_deactivate` when stowed). Moving into/out of the active slot interacts with
`Activate` (below). Constraints: belt has a width limit and an item must be `Belt()`-capable; slots
can be forbidden by the worn outfit (helmet/backpack availability); ruck is unlimited. *(bodies
agent-traced; the mutual-exclusion + callback pattern is the load-bearing structure.)*

## Activate — choose the in-hands item (verified state machine)

**`CInventory::Activate(u16 slot, bool bForce)`** (verified body) is a *deferred* switch, not
immediate:

- Guarded by `OnServer()` (returns on a pure client), a blocked-slot check (stashes prev slot to
  retry after unblock), and a duplicate early-out (already active / already next).
- It sets **`m_iNextActiveSlot`** rather than swapping immediately; if an item is currently active it
  asks that item's `CHudItem` to `SendDeactivateItem()` (which plays the holster animation — see
  `flows/weapon-state-machine.md`).
- **`CInventory::Update()`** (per-frame, server side) resolves the pending switch: once the outgoing
  item reports hidden, it activates `m_iNextActiveSlot`'s item (`ActivateItem()`) and commits
  `m_iActiveSlot`, then fires the Lua functor **`_G.CActor_OnChangedSlot(slot, item, prev_slot,
  prev_item)`**. *(Update body agent-traced; the `m_iActiveSlot`/`m_iNextActiveSlot` two-phase
  mechanism is verified in `Activate`.)*

This two-phase design is why drawing a new weapon waits for the current one's holster animation — the
switch is gated on the outgoing `CHudItem` finishing its hide.

## Drop / remove (agent-traced)

`CInventory::DropItem(pObj, just_before_destroy, dont_create_shell)`: cast, re-activate engine
processing, remove from its container, if it was the active slot call `Activate(NO_ACTIVE_SLOT)`,
erase from `m_all`, null `m_pInventory`, `OnItemDrop`, then `H_SetParent(nullptr)` to unparent the
object; if a UI is up fire `OnInventoryAction(…, GE_OWNERSHIP_REJECT)`. *(body agent-traced.)*

## The ownership event seam (agent-traced)

Pickup/drop is **server-authoritative via net events**, not a direct call:

- A pickup is requested via `game_cl_GameState::SendPickUpEvent` → `GE_OWNERSHIP_TAKE`. The server's
  `xrServer::Process_event_ownership` validates ownership and `game->OnTouch(...)`, sets the server
  entity's `ID_Parent`, appends to the parent's `children`, and broadcasts.
- The client actor handles it in **`CActor::OnEvent`** (`Actor_Events.cpp`): on `GE_OWNERSHIP_TAKE`
  it `CanTakeItem` → `H_SetParent(actor)` → `inventory().Take(...)`; on failure emits
  `GE_OWNERSHIP_REJECT`. On `GE_OWNERSHIP_REJECT` it `DropItem(...)`. `GE_TRADE_BUY`/`GE_TRADE_SELL`
  are handled as take/reject for transfers between owners.
- Pickup therefore always pairs `H_SetParent` with `Take` (and `H_SetParent(nullptr)` with
  `DropItem`); an out-of-sync pair is an assert. See `engine/server-entities.md` for the event layer.

## Isolation ledger (the seams)

- **Callbacks fired:** Lua functors `eOnItemTake`/`eOnItemDrop` (always; from
  `CInventoryOwner::OnItemTake`/`OnItemDrop`), and `eItemToBelt`/`eItemToSlot`/`eItemToRuck` (only
  under the `EXTENDED_ITEM_CALLBACKS` build define); `_G.CActor_OnChangedSlot` from
  `CInventory::Update`. Net events `GE_OWNERSHIP_TAKE`/`GE_OWNERSHIP_REJECT` and
  `GE_TRADE_BUY`/`GE_TRADE_SELL` carry pickup/drop/transfer. *(callback enums agent-traced; `Take`
  firing `OnItemTake` + UI `GE_OWNERSHIP_TAKE` is verified.)*
- **Config read (item `.ltx`):** `inv_weight`, `cost`, `slot` (base slot; -1 = ruck default),
  `belt`, `can_take`, `can_trade`, `quest_item`, `use_condition`, `icon_name`, `default_to_ruck`,
  `sprint_allowed`; inventory section `slot_persistent_N`/`slot_active_N`/`max_weight`. *(key names
  agent-traced.)*
- **Scheduler / per-frame seam:** `CInventory::Update` runs from the owner's update (resolves the
  pending active-slot switch + drop tasks). The active-slot switch is gated on the outgoing
  `CHudItem` reporting hidden.
- **ALife / server seam:** items are `CSE_ALifeInventoryItem` (condition + upgrades restored at
  `CInventoryItem::net_Spawn`); ownership is the server entity `ID_Parent`/`children` graph mutated
  by `Process_event_ownership`. `engine/server-entities.md`.
- **Lua seam:** `obj:inventory()`, `obj:TakeItem(item)` (script bridge → trade or ownership events),
  `obj:money()`/`set_money`, the `Iterate*`/`ForEachInventoryItems` iterators; item callbacks
  `on_item_take`/`on_item_drop`. Wrappers in `script_game_object_inventory_owner.cpp`.
- **Other domains:** weapons/`CHudItem` (slot activation drives draw/holster —
  `flows/weapon-state-machine.md`), actor as the owner (`flows/actor.md`), outfit (slot
  availability), trade, attachment owner (`attach`/`detach` on take/drop).

## Threading

Main thread; inventory mutation rides server-ordered net events, so client state follows the
authoritative server decision.

## Failure & edge cases

- **`Take` requires `m_pInventory == NULL`** (asserted) — taking an already-owned item without
  dropping first asserts. Pickup must pair `H_SetParent` + `Take`; drop must pair `DropItem` +
  `H_SetParent(nullptr)`.
- **Active-slot switch is deferred**: `Activate` sets `m_iNextActiveSlot`; the swap only completes in
  `Update` after the outgoing item hides. Don't assume `ActiveItem()` changed the instant you called
  `Activate`.
- **`Activate` no-ops off the server** (`OnServer()` guard) — a pure-client call does nothing.
- **Weight is enforced only for non-actor owners** in `CanTakeItem`; the actor is intentionally
  uncapped at this layer.
- **Belt/slot can be refused by the worn outfit** (helmet/backpack availability) — `Take`'s cascade
  then falls through to ruck.

## Deeper

- Container: `src/xrGame/Inventory.h`, `src/xrGame/Inventory.cpp` (`Take`, `Slot`, `Belt`, `Ruck`,
  `DropItem`, `Activate`, `Update`, `CanPutIn*`, `CanTakeItem`)
- Item: `src/xrGame/inventory_item.cpp/.h` (`Load`, `net_Spawn`, `OnH_A_Chield`/`OnH_B_Chield`,
  `m_ItemCurrPlace`)
- Owner callbacks: `src/xrGame/InventoryOwner.cpp` (`OnItemTake`/`OnItemDrop`/`OnItem{Slot,Belt,
  Ruck}`)
- Ownership events: `src/xrGame/Actor_Events.cpp` (`CActor::OnEvent` GE_OWNERSHIP_*),
  `engine/server-entities.md`
- Script bridge: `src/xrGame/script_game_object_inventory_owner.cpp` (`TakeItem`, iterators)

## Maintenance

- Re-grep `CInventory::Take`/`Activate`/`Update` if container logic moves; re-grep
  `_G.CActor_OnChangedSlot` and `eOnItemTake`/`eOnItemDrop` to confirm the callbacks fire.
- Promote the *(agent-traced)* bodies (`Slot`/`Belt`/`Ruck`/`DropItem`, `Update`, the owner
  callbacks, `Actor_Events` handlers, the `.ltx` key list) to verified by reading them when a task
  needs the detail.
- Keep the "active-slot switch is two-phase / deferred" and "ruck is the terminal placement" notes —
  both are non-obvious and the source of common bugs.
