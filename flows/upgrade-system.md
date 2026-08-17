---
domain: flows/upgrade-system
tags: [xrgame, upgrade, inventory, weapon, outfit, ltx, money, flow]
sources: [src/xrGame/inventory_upgrade_manager.cpp, src/xrGame/inventory_upgrade_manager.h, src/xrGame/inventory_upgrade.cpp, src/xrGame/inventory_upgrade_group.cpp, src/xrGame/inventory_upgrade_base.h, src/xrGame/inventory_item_upgrade.cpp, src/xrGame/inventory_item_inline.h, src/xrGame/WeaponUpgrade.cpp, src/xrGame/CustomOutfit.cpp, gamedata/scripts/aaaa_script_fixes_mp.script]
related: [flows/weapon-hierarchy-addons.md, flows/outfit-helmet.md, flows/inventory.md, flows/dialog-trade.md, flows/items-weapons-outfits-configs.md, modlayer/unlocalizer.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: inventory upgrade system

How an item upgrade (the mechanic's weapon/outfit upgrade grid) is defined in config, gated, and
**applied as a stat change** to an item. This is the system F22 and F23 left as *(agent-traced)* — the
`install_upgrade_impl` overrides for weapons (`flows/weapon-hierarchy-addons.md`) and outfits
(`flows/outfit-helmet.md`) are the *leaves* of this flow; here is the manager/base spine that drives
them. The mechanic UI and money/quest gates ride this too (`flows/dialog-trade.md`).

**Provenance:** spine personally code-verified — `Manager::upgrade_install`/`upgrade_verify`/
`init_install`, `CInventoryItem::install_upgrade`/`verify_upgrade`/`install_upgrade_impl` (base)/
`has_upgrade`/`add_upgrade`/`pre_install_upgrade`, the `process_if_exists` (additive) vs
`process_if_exists_set` (replace) templates, and the existence of `Upgrade::can_install`/`run_effects`,
`Group::can_install`, and the `UpgradeStateResult` enum. The data-model loaders
(`Upgrade::construct`/`Property::construct`/`Manager::load_all_*`), the `can_install`/`run_effects`
bodies, the Lua functor contracts, and the per-item `install_upgrade_impl` *bodies* (weapon/outfit) are
confirmed homes but *(agent-traced — not personally re-read)*. All upgrade code lives in
`namespace inventory`.

## Data model (manager-owned tree; loaders agent-traced)

`inventory::Manager` (the upgrade manager, reached via `ai().alife().inventory_upgrade_manager()`) owns
id-keyed maps of four element types (all derive from `UpgradeBase`):

- **Root** — one per upgradable item section; holds the item's `m_upgrade_scheme` (UI grid name) and
  its child Upgrades. Created by `load_all_inventory` for every `.ltx` item section that has
  `upgrades = …` + `upgrade_scheme = …`.
- **Group** — a **mutually-exclusive** bundle of Upgrades (only one installable per group) that also
  records its **parent Upgrades** (cross-group prerequisites). Shared/reused across Upgrades.
- **Upgrade** — one grid cell: a `section()` (the settings section holding the actual stat deltas), a
  `scheme_index` (grid x,y), a parent group, up to 4 referenced **Properties**, a `known` flag, and Lua
  functors (`precondition`, `effect`, `prerequisite`).
- **Property** — **UI/tooltip only.** It carries a name/icon + a Lua description functor; it is *not*
  applied to the item. The stat change comes from the Upgrade's `section()`, not from Properties — a
  common point of confusion.

*(The construct/load bodies are agent-traced; the containment above is consistent with the verified
manager access + `upgrade_verify` + the `UpgradeStateResult` enum.)*

## Install sequence (verified)

`inventory::Manager::upgrade_install(item, upgrade_id, loading)` (called from the mechanic UI, or with
`loading=true` from save-restore / `init_install`):

1. **`upgrade_verify(item.m_section_id, upgrade_id)`** — look the Upgrade up and confirm it belongs to
   this item's Root; null → bail.
2. **`upgrade->can_install(item, loading)`** → a `UpgradeStateResult`. Gates (verified to exist;
   bodies agent-traced): `UpgradeBase` `known` check, `Group::can_install` (parent upgrades installed +
   group exclusivity), and — only when **not** loading — the Lua **precondition** functor (money /
   quest). Result enum: `result_ok`, `result_e_parents`, `result_e_group`,
   `result_e_precondition_money`, `result_e_precondition_quest`, `result_e_unknown`,
   `result_e_installed`.
3. if `result_ok`: when not loading, **`item.pre_install_upgrade()`** (e.g. `UnloadMagazine` for a
   weapon, so an upgrade can't be installed with a loaded mag).
4. **`item.install_upgrade(upgrade->section())`** → applies the stat deltas (the override chain below).
   Returns bool. If it returns **false the game `FATAL`s** ("Upgrade … is EMPTY or FAILED") — an
   upgrade whose section sets nothing is a hard config error.
5. on success: **`upgrade->run_effects(loading)`** (Lua **effect** functor — deducts money, fires
   script side-effects) then **`item.add_upgrade(upgrade_id, loading)`**.

`init_install(item)` seeds an item's default upgrades from its `installed_upgrades` config line (used
on spawn) via the same path with `loading`-style application.

## Applying the stat change (verified) — the core mechanism

`CInventoryItem::install_upgrade(section)` = `install_upgrade_impl(section, test=false)`;
`verify_upgrade(section)` = `install_upgrade_impl(section, test=true)` — **`test` is the dry-run flag**
(read + validate, don't mutate; used to check an upgrade is non-empty).

`CInventoryItem::install_upgrade_impl(section, test)` (base) applies generic fields and **returns true
iff at least one field existed** (the emptiness check step 4 relies on): `cost` (`r_u32 → m_cost`),
`inv_weight` (`→ m_weight`), `default_to_ruck`/`sprint_allowed` flags, `control_inertion_factor`,
`immunities_sect`/`immunities_sect_add` (→ `CHitImmunity::Load/AddImmunities`). Subclasses **call the
base first** then add their own fields:
- **`CWeapon::install_upgrade_impl`** (`WeaponUpgrade.cpp`) — dispersion, recoil/cam, hit power, RPM,
  ammo class, and the **addon** sub-installer (`install_upgrade_addon`: scope/silencer/GL status +
  `m_scopes`/names, then `InitAddons` — the F22 seam). *(body agent-traced.)*
- **`CCustomOutfit::install_upgrade_impl`** (`CustomOutfit.cpp`) — the per-type `*_protection`, bone
  armor (`bones_koeff_protection`/`_add` → `ReloadBonesProtection`/`AddBonesProtection`), restore
  speeds, power-loss, weights, artefact slots (the F23 seam). *(body agent-traced.)*

**The application primitives** (`inventory_item_inline.h`, verified) — every field above goes through
one of two templates that read a `CInifile` member-pointer getter from the section:
- **`process_if_exists`** → `value = value + read` — **additive** (multiple installed upgrades *stack*).
- **`process_if_exists_set`** → `value = read` — **replace** (for flags/strings/enums: scopes,
  immunities section, ruck/sprint flags).
Both no-op (return false) if the line is absent or empty, and when `test` is set they read without
mutating. This additive-vs-replace split is the single fact that explains all upgrade math.

## Mountable scopes granted by an upgrade (repo addition, Lua layer)

The engine already lets an upgrade `process_if_exists_set` the weapon's `scopes` field, but the *mod
layer* gated attachment separately: `item_weapon.check_scope(addon_sect, weapon_sect)` only consults the
weapon **section's** `scopes` list, so a scope granted by an upgrade could not be mounted. The repo
patches this in `gamedata/scripts/aaaa_script_fixes_mp.script` (base-script override territory —
`modlayer/unlocalizer.md`):

- `item_weapon.attach_scope` and `item_weapon.on_item_drag_dropped` are wrapped to stash the
  (item, weapon) pair in module locals for the duration of the call, because `check_scope` receives only
  *sections* and needs the live weapon to inspect its upgrades;
- the wrapped `check_scope` falls back to iterating `weapon:iterate_installed_upgrades`, reading each
  upgrade's `section` → that section's `scopes` list (`get_scopes_from_upgrade`), and accepting the addon
  if it appears there;
- `ActorMenu_on_item_focus_receive` highlights those upgrade-granted scopes in every relevant drag-drop
  list when the weapon is focused, so the player can see what now fits.

Consequence for content: an upgrade can hand a weapon a new mountable scope by declaring `scopes` on
the section it sets — no weapon-section edit and no engine change needed.

## Storage & persistence (verified)

Installed upgrades live in `CInventoryItem::m_upgrades` (a vector of upgrade-id `shared_str`).
`has_upgrade(id)` = `id == m_section_id || id ∈ m_upgrades`; `add_upgrade(id, loading)` pushes it and,
**when not loading**, emits `GE_INSTALL_UPGRADE` (`u_EventGen` + `w_stringZ(id)` + `u_EventSend`) so the
install replicates / persists. `has_upgrade_group`, `get_upgrades_str`, `equal_upgrades` are queries
over the same vector. Save/load of `m_upgrades` rides the item's server-entity serialization
(`flows/save-load.md`).

## Isolation ledger (the seams)

- **Inventory item:** the whole flow mutates a `CInventoryItem` via its virtual `install_upgrade_impl`
  (`flows/inventory.md`); weapon/outfit overrides are F22/F23.
- **Money / quest:** the Lua **precondition** functor reads actor money / quest state to gate install;
  the **effect** functor deducts money on success — the economic seam (`flows/dialog-trade.md`).
- **UI:** the mechanic upgrade window queries `Manager` for the Root/scheme/Groups/Upgrades, renders
  Properties as tooltips via their Lua description functor, and calls `upgrade_install` on click.
- **Lua script surface** *(agent-traced)*: per-upgrade `precondition_functor` (→ int result),
  `effect_functor` (money/side-effects), `prereq_functor` (→ tooltip string); per-property description
  functor. Bound at element `construct` time from the manager's script engine.
- **Network/save:** `GE_INSTALL_UPGRADE` event; `m_upgrades` serialized with the item.
- **Config read:** item section (`upgrades`, `upgrade_scheme`, `installed_upgrades`); `[upgrade_*]`
  (`section`, `scheme_index`, the three functor+param pairs, `property`, `effects`, `known`);
  `[upgrade_group_*]` (`elements`); the Upgrade's **settings `section`** (the actual `cost`/stat
  deltas applied by `process_if_exists`); `[prop_*]` + the `upgrades_properties` list.

## Threading

Main thread (UI → manager → item). Single-threaded with the rest of gameplay/Lua.

## Failure & edge cases

- **Empty upgrade section is FATAL:** if `install_upgrade` returns false (the section set *nothing*
  recognized), the game hard-faults — the most common upgrade config error. Use `verify_upgrade`
  (`test=true`) to detect it.
- **Additive stacking:** because `process_if_exists` is `+=`, two installed upgrades that both set
  `hit_power` add up; re-applying (e.g. double install) would double-count — `add_upgrade` guards via
  `has_upgrade`, but config that lists the same delta in multiple installed upgrades stacks
  intentionally.
- **Group exclusivity / parents:** `can_install` returns `result_e_group` (a sibling already installed)
  or `result_e_parents` (a required prior upgrade missing) — the grid's branching is enforced here, not
  in the item.
- **Properties ≠ effects:** editing a `[prop_*]` changes only the tooltip; the stat change is in the
  Upgrade's `section`. Editing the wrong one is a frequent mistake.
- **`loading` skips gates + money:** on save-restore / defaults, preconditions and the effect functor
  are bypassed so a saved build is restored exactly without re-charging the player.

## Deeper

- Manager + sequence: `src/xrGame/inventory_upgrade_manager.cpp` (`upgrade_install`, `upgrade_verify`,
  `init_install`, `load_all_inventory`, `load_all_properties`), `…_manager.h` (`get_root`/`get_upgrade`/
  `get_group`/`get_property`)
- Elements: `src/xrGame/inventory_upgrade.cpp` (`Upgrade::construct`/`can_install`/`run_effects`),
  `inventory_upgrade_group.cpp` (`Group::can_install` exclusivity), `inventory_upgrade_property.cpp`,
  `inventory_upgrade_root.cpp`, `inventory_upgrade_base.h` (`UpgradeBase`, `UpgradeStateResult`)
- Item side: `src/xrGame/inventory_item_upgrade.cpp` (`install_upgrade`, `verify_upgrade`,
  `install_upgrade_impl`, `has_upgrade`, `add_upgrade`, `pre_install_upgrade`, `get_upgrades_str`),
  `inventory_item_inline.h` (`process_if_exists`, `process_if_exists_set`)
- Per-item overrides: `src/xrGame/WeaponUpgrade.cpp` (`install_upgrade_addon`) — `flows/weapon-hierarchy-addons.md`;
  `src/xrGame/CustomOutfit.cpp` (`install_upgrade_impl`) — `flows/outfit-helmet.md`

## Maintenance

- Re-grep `Manager::upgrade_install` if the install sequence changes — it is the single orchestrator
  (verify → can_install → pre_install → install_upgrade → run_effects → add_upgrade).
- Keep the `process_if_exists` (additive) vs `process_if_exists_set` (replace) distinction exact — it
  is the one fact that explains every upgrade stat result; re-grep `inventory_item_inline.h` if it moves.
- Promote the *(agent-traced)* loaders/functor bodies and the weapon/outfit `install_upgrade_impl`
  bodies to verified when an upgrade-authoring task needs the exact field lists (the field lists in
  `flows/weapon-hierarchy-addons.md` / `flows/outfit-helmet.md` are the per-domain references).
