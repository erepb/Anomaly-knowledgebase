---
domain: flows/outfit-helmet
tags: [xrgame, outfit, armor, helmet, protection, immunity, condition, flow]
sources: [src/xrGame/CustomOutfit.cpp, src/xrGame/CustomOutfit.h, src/xrGame/ActorHelmet.cpp, src/xrGame/ActorHelmet.h, src/xrGame/EntityCondition.cpp, src/xrGame/Actor.cpp, src/xrGame/BoneProtections.h, src/xrGame/hit_immunity.h]
related: [flows/inventory.md, flows/damage-hit.md, flows/actor.md, flows/items-weapons-outfits-configs.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: outfits & helmets (body armor)

How body armor (outfit) and helmet items equip, **reduce incoming damage**, wear out, and grant
passive bonuses (restore, immunities, night-vision, power-loss). They are inventory items that live in
dedicated slots; the damage pipeline (`flows/damage-hit.md`) consults them, and the actor condition
tick (`flows/actor.md`) reads their restore speeds. This flow is the equipment + protection layer that
sits between inventory and the hit/condition systems.

**Provenance:** spine personally code-verified — `CCustomOutfit::GetDefHitTypeProtection`/
`GetHitTypeProtection`/`GetBoneArmor`/`HitThroughArmor`/`Hit`/`OnMoveToSlot`/`OnMoveToRuck`/
`ApplySkinModel`/`get_artefact_count`, `CEntityCondition::HitOutfitEffect`/`HitPowerEffect`, the
passive restore tick in `CActor` (`UpdateInventoryOwner` area), `CExoOutfit` (confirmed pure-config),
and that `CHelmet` carries its own `Load`/`OnMoveToSlot`/`HitThroughArmor`. The full `CHelmet` bodies,
`CCustomOutfit::Load`/`install_upgrade_impl` param reads, `CHitImmunity::LoadImmunities`, and the
`SBoneProtections::reload` internals are confirmed homes but *(agent-traced — not personally re-read)*.

## Type tree (verified from headers)

- **`CCustomOutfit`** ← `CInventoryItemObject` (← `CInventoryItem` which is `CHitImmunity`) — the body
  armor base; holds the whole protection/restore/bone-armor model below.
  - **`CExoOutfit`**, **`CMilitaryOutfit`**, **`CScientificOutfit`**, **`CStalkerOutfit`** ←
    `CCustomOutfit` — **pure config** (ctor/dtor + luabind registration only; `CExoOutfit` adds *no*
    code — exoskeleton stamina/move behavior is config via `control_inertion_factor`/`power_loss`, not
    a C++ override). Differentiation is entirely in the `.ltx` section.
- **`CHelmet`** ← `CInventoryItemObject` — a **sibling**, not an outfit subclass. It re-implements the
  same protection/restore/bone-armor shape (own `Load`/`OnMoveToSlot`/`HitThroughArmor`).

Accessor asymmetry: `CInventoryOwner::GetOutfit()` exists (casts `ItemFromSlot(OUTFIT_SLOT)`), but there
is **no `GetHelmet()`** — helmet access is always `ItemFromSlot(HELMET_SLOT)` + `smart_cast<CHelmet*>`.
Slot constants (`inventory_space.h`): `OUTFIT_SLOT`, `HELMET_SLOT`, `BACKPACK_SLOT`.

## Equip / unequip (verified)

`CCustomOutfit::OnMoveToSlot(prev)` (fires when the item is placed in `OUTFIT_SLOT`, via the inventory
slot machine `flows/inventory.md`):
1. `ApplySkinModel(pActor, dress=true)` — swap the actor's third-person visual (team-aware: look up
   `cNameSect()` in the team section under `mp_skins_path`, else use `m_ActorVisual`) via
   `ChangeVisual`; for the local player also `g_player_hud->load(player_hud_section)` (the first-person
   arms/HUD model).
2. **Compatibility eject:** if `!bIsHelmetAvaliable`, force any equipped helmet to the ruck
   (`inventory().Ruck`); same for backpack via `bIsBackpackAvaliable`. So an outfit that disallows a
   helmet auto-removes one.
3. night-vision re-evaluated (`SwitchNightVision`) when the outfit itself owns NV (no helmet slot).

`OnMoveToRuck(prev)` mirrors it: `ApplySkinModel(dress=false)` restores `GetDefaultVisualOutfit` +
`g_player_hud->load_default()`, and disables NV if the outfit owned it. `CHelmet::OnMoveToSlot`/
`OnMoveToRuck` toggle NV similarly (only if the outfit doesn't already block it). *(helmet bodies
agent-traced.)*

## Protection model (verified) — two independent axes

The crucial distinction for modding/debugging:

**Axis 1 — protection (reduces the damage you take), scaled by wear:**
- `m_HitTypeProtection` (`HitImmunity::HitTypeSVec`, indexed by `ALife::EHitType` — burn/strike/shock/
  wound/radiation/telepatic/chemical/explosion/fire_wound/…) is the configured protection per hit type.
- `GetDefHitTypeProtection(hit_type)` = `m_HitTypeProtection[hit_type] * GetCondition()` — **scaled by
  the item's condition** (a worn-out 10%-condition suit gives ~10% of its rated protection).
- `GetHitTypeProtection(hit_type, bone)` = that, times the per-bone koeff
  (`m_boneProtection->getBoneProtection(bone)`) — different bones armor differently.

**Axis 2 — immunity (scales how fast the item itself wears out):** `CHitImmunity::GetHitImmunity`
(loaded from `immunities_sect`). Used only in `Hit` (below), **not** to reduce player damage.

### `HitThroughArmor` — the damage-reduction function (verified)

`CCustomOutfit::HitThroughArmor(hit_power, bone, ap, add_wound, hit_type)` returns the *reduced* hit:
- **`eHitTypeFireWound` (bullets):** AP-vs-bone-armor ballistics. `BoneArmor = GetBoneArmor(bone) *
  GetCondition()` (returns hit unchanged if bone armor ≤ 0). If `ap <= BoneArmor` (didn't penetrate):
  `NewHitPower *= m_fHitFracActor` (small residual). Else a penetration fraction
  `(ap - BoneArmor)/(ap * APScale)` clamped to `[m_fHitFracActor, 1]`. An **alternate "Jurko" formula**
  runs instead when `m_fireWoundParam1`/`m_fireWoundParam2` are set (optional `.ltx` params).
- **other hit types:** flat subtraction — `NewHitPower -= GetDefHitTypeProtection(hit_type) * one`,
  where `one = 1.0` for strike/wound/wound_2/explosion and `0.1` for everything else, floored at 0.
- then **`Hit(hit_power, hit_type)`** degrades the item's own condition, and returns `NewHitPower`.

`CHelmet::HitThroughArmor` is a separate method of the same shape (verified to exist).

`CCustomOutfit::Hit(hit_power, hit_type)` = `ChangeCondition(-hit_power * GetHitImmunity(hit_type))` —
this is how taking hits **wears out** the armor (immunity axis), independent of how much it protected.

## Damage-pipeline seam (verified)

`CEntityCondition::HitOutfitEffect(hit_power, hit_type, bone, ap, add_wound)` is the single call the
hit pipeline (`flows/damage-hit.md`) makes: fetch `OUTFIT_SLOT` outfit + `HELMET_SLOT` helmet; run the
hit through **outfit first, then helmet** (`new = outfit->HitThroughArmor(...); new =
helmet->HitThroughArmor(new, ...)`), returning the doubly-reduced power. Both slots stack, both wear.

`CEntityCondition::HitPowerEffect(power_loss)` = stamina drain scaling: averages `m_fPowerLoss` of
outfit + helmet + backpack (`/3`), times `f_power_loss_factor`, plus bias, clamped `[0,1]`. This is the
"heavy armor tires you faster" knob.

## Passive bonuses — restore tick (verified)

In the actor condition tick (`CActor`, `UpdateInventoryOwner` path): if an outfit and/or helmet is
worn, sum their restore speeds and apply per frame — `conditions().ChangeHealth(((outfit?
m_fHealthRestoreSpeed:0) + (helmet? m_fHealthRestoreSpeed:0)) * f_update_time)` and likewise
ChangePower / ChangeSatiety / ChangeBleeding / ChangeRadiation. Worn **artefacts** add their own
restore (scaled by artefact condition) in the same loop, *separately*. So passive regen = outfit +
helmet + artefacts, all additive.

`get_artefact_count()` returns `m_artefact_count` but lets a Lua functor
`_G.CCustomOutfit_get_artefact_count` override it (how mods change how many artefacts a suit holds).

## Isolation ledger (the seams)

- **Inventory:** outfit/helmet are `CInventoryItemObject`s in `OUTFIT_SLOT`/`HELMET_SLOT`; equip/unequip
  is the slot machine firing `OnMoveToSlot`/`OnMoveToRuck` (`flows/inventory.md`). Outfit can force
  helmet/backpack to ruck on equip.
- **Damage/condition:** `HitOutfitEffect` (outfit→helmet reduction) and `HitPowerEffect` (stamina) are
  called from `CEntityCondition` in the hit pipeline (`flows/damage-hit.md`); `Hit` wears the item.
- **Actor:** passive restore tick reads `m_f*RestoreSpeed`; visual swap (`ChangeVisual`) + first-person
  `g_player_hud->load(player_hud_section)`; night-vision toggle (`flows/actor.md`).
- **Bone armor:** `SBoneProtections` (`BoneProtections.h`) maps bone→{koeff, armor, pass-bullet,
  `m_fHitFracActor`, `APScale`}, reloaded from the `bones_koeff_protection` section against the actor
  skeleton (`ReloadBonesProtection`). `BonePassBullet(bone)` lets some bones pass bullets through.
- **Artefacts/upgrades:** worn artefacts add restore (separate slot); `install_upgrade_impl` can modify
  every protection/restore/bone-armor/weight/power-loss param and append bone sections
  (`bones_koeff_protection_add`) then `ReloadBonesProtection`/`AddBonesProtection` *(agent-traced — see
  the F26 upgrades flow)*.
- **Lua callback:** `_G.CCustomOutfit_get_artefact_count` (artefact-capacity override). No equip/unequip
  Lua callback in the outfit layer itself.
- **Config read** (`Load`, *(agent-traced for full list)*): per-type `*_protection` (burn/strike/shock/
  wound/radiation/telepatic/chemical_burn/explosion/fire_wound), `fire_wound_param_1/2`,
  `bones_koeff_protection`, `hit_fraction_actor`, `*_restore_speed` (health/radiation/satiety/power/
  bleeding), `power_loss`, `actor_visual`, `player_hud_section`, `nightvision_sect`, `helmet_avaliable`,
  `backpack_avaliable`, `artefact_count`, `control_inertion_factor`, `use_condition`,
  `additional_inventory_weight*`, `immunities_sect` (inherited). Helmet adds `nearest_enemies_show_dist`.

## Threading

Main thread only (inventory/condition/HUD). No worker-thread interaction.

## Failure & edge cases

- **Condition is a damage multiplier:** all protection is `× GetCondition()`, so a near-broken suit
  protects almost nothing — a frequent "why am I dying in armor" cause.
- **`bones_koeff_protection` must match the actor skeleton:** `ReloadBonesProtection` reloads against
  the *current view entity's* `IKinematics`; a bone name in the section not present on the skeleton
  yields no armor for that bone. Bone armor ≤ 0 makes `HitThroughArmor` (fire-wound) return the hit
  unchanged.
- **Protection vs immunity confusion:** `*_protection` (`m_HitTypeProtection`) reduces *your* damage;
  `immunities_sect` (`GetHitImmunity`) only scales *the item's wear*. Editing the wrong one is a common
  config mistake.
- **`helmet_avaliable=false` ejects helmets:** equipping such an outfit silently moves a worn helmet to
  the ruck (and re-routes NV to the outfit).
- **Outfit + helmet stack:** damage passes through both, both wear; restore from both sums. Two armors
  on one hit is intended, not double-counting a bug.

## Deeper

- Outfit protection/equip: `src/xrGame/CustomOutfit.cpp` (`GetDefHitTypeProtection`,
  `GetHitTypeProtection`, `GetBoneArmor`, `HitThroughArmor`, `Hit`, `OnMoveToSlot`, `OnMoveToRuck`,
  `ApplySkinModel`, `ReloadBonesProtection`, `install_upgrade_impl`, `get_artefact_count`),
  `src/xrGame/CustomOutfit.h` (`m_HitTypeProtection`, `m_boneProtection`, restore-speed members)
- Helmet: `src/xrGame/ActorHelmet.cpp` (`Load`, `OnMoveToSlot`, `OnMoveToRuck`, `HitThroughArmor`),
  `src/xrGame/ActorHelmet.h` (`CHelmet`)
- Damage seam: `src/xrGame/EntityCondition.cpp` (`HitOutfitEffect`, `HitPowerEffect`) — `flows/damage-hit.md`
- Restore tick + GetOutfit: `src/xrGame/Actor.cpp` (`UpdateInventoryOwner` area),
  `src/xrGame/InventoryOwner.cpp` (`GetOutfit`)
- Bone armor + immunity: `src/xrGame/BoneProtections.h` (`SBoneProtections`),
  `src/xrGame/hit_immunity.h` (`CHitImmunity`, `LoadImmunities`)
- Subclasses (config): `src/xrGame/{Exo,Military,Scientific,Stalker}Outfit.h`

## Maintenance

- Re-grep `CEntityCondition::HitOutfitEffect` if the damage seam moves — it is the single call the hit
  pipeline makes into armor; keep the outfit→helmet order accurate.
- Re-grep `CCustomOutfit::HitThroughArmor` if the protection math changes — keep the two-axis
  (protection × condition vs immunity-as-wear) distinction explicit; it is the most-misread part.
- Re-confirm subclasses stay config-only (`^class CExoOutfit`); if any gains a code override, document
  it here.
- Promote the *(agent-traced)* items (`CHelmet` bodies, full `Load`/`install_upgrade_impl` param list,
  `SBoneProtections::reload`) to verified when an armor-tuning or upgrade task needs them.
