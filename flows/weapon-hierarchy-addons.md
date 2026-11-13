---
domain: flows/weapon-hierarchy-addons
tags: [xrgame, weapons, addons, scope, silencer, grenade-launcher, ammo, inheritance, flow]
sources: [src/xrGame/Weapon.h, src/xrGame/Weapon.cpp, src/xrGame/WeaponMagazined.cpp, src/xrGame/WeaponMagazined.h, src/xrGame/WeaponAmmo.cpp, src/xrGame/Scope.cpp, src/xrGame/Silencer.cpp, src/xrGame/GrenadeLauncher.cpp, src/xrGame/WeaponUpgrade.cpp, src/xrGame/WeaponSSRS.cpp, src/xrGame/WeaponSSRS.h, src/xrGame/RocketLauncher.cpp, src/xrGame/inventory_item_upgrade.cpp]
related: [flows/weapon-fire-callbacks.md, flows/weapon-state-machine.md, flows/inventory.md, flows/object-lifecycle.md, flows/items-weapons-outfits-configs.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: weapon class hierarchy & addons

How the weapon **type tree** is organized and how **addons** (scope / silencer / grenade-launcher)
attach, detach, and re-parameterize a weapon. This is the *structure + equipment* flow; the firing
chain is `flows/weapon-fire-callbacks.md` and the equip/idle/fire/reload/zoom/hide state machine is
`flows/weapon-state-machine.md` — this flow only references them as attachment points. Where a weapon
lives in the inventory and how it is taken/activated is `flows/inventory.md`.

**Provenance:** spine personally code-verified — the full inheritance tree (class headers),
`CWeapon::IsScopeAttached/IsSilencerAttached/IsGrenadeLauncherAttached`, `CWeapon::GetScopeName`,
`CWeaponMagazined::CanAttach/CanDetach/Attach/Detach/DetachScope/InitAddons`, the koeff
load/apply/reset family, and `CWeaponAmmo::Load/Get` plus the `CWeapon` magazine members. The addon
**network sync** (`net_Spawn`/`net_Export`/`OnEvent` for `GE_ADDON_CHANGE`), `WeaponUpgrade.cpp`'s
`install_upgrade_addon`, the `CWeaponMagazinedWGrenade` dual-mode + `CRocketLauncher`, and
`CWeaponStatMgun` mounted-gun path are confirmed homes but *(agent-traced — not personally re-read)*.

## Inheritance tree (verified from class headers)

- **`CWeapon`** (`Weapon.h`: `class CWeapon : public CHudItemObject, …`) — the base: addon status +
  flag tracking, ammo-type list & magazine, zoom params, fire/reload entry points, `LoadFireParams`,
  net (de)serialization of addon flags. A HUD item (so it rides `flows/animation-callbacks.md`).
- **`CWeaponKnife`** ← `CWeapon` directly — melee; **no magazine/ammo/zoom** (overrides Fire to a
  strike, `IsZoomEnabled()==false`). *(agent-traced.)*
- **`CWeaponStatMgun`** ← `CPhysicsShellHolder` + `CHolderCustom` + `CShootingObject` — **not** in the
  `CWeapon` tree at all: a mounted/turret gun (multi-barrel, player mounts it as a holder, see the
  vehicle/holder flow F29 when written). *(agent-traced.)*
- **`CWeaponMagazined`** ← `CWeapon` — adds the **magazine reload machine** and the **addon
  attach/detach + koeff** system (this flow's core). Almost every usable firearm descends from here.
  - **`CWeaponAutoPistol`** (`WeaponCustomAuto.h`) ← `CWeaponMagazined` — single-shot-per-trigger.
    - **`CWeaponPistol`** ← `CWeaponAutoPistol` — `AllowFireWhileWorking()==true`.
      - `CWeaponHPSA`, `CWeaponFORT`, `CWeaponPM` — leaf config.
  - **`CWeaponCustomPistol`** (`WeaponCustomPistol.h`) ← `CWeaponMagazined` — **misnamed base**: many
    non-pistols descend from it (it just means "single fire-mode by default"). Children:
    - **`CWeaponShotgun`** ← `CWeaponCustomPistol` — tri-state per-shell reload
      (`PlayAnimOpenWeapon`/`AddCartridge`/`CloseWeapon`).
    - **`CWeaponSVD`** ← `CWeaponCustomPistol` — sniper `switch2_Fire`/`OnAnimationEnd`.
    - **`CWeaponBinoculars`** ← `CWeaponCustomPistol` — optics-only, `can_kill()==false`, custom zoom.
    - **`CWeaponRPG7`** ← `CWeaponCustomPistol` + `CRocketLauncher`.
  - **`CWeaponAutomaticShotgun`** ← `CWeaponMagazined` — auto-cycling shotgun.
  - **`CWeaponMagazinedWGrenade`** ← `CWeaponMagazined` + `CRocketLauncher` — dual-mode rifle + UGL
    (second ammo set `m_ammoType2`/`iAmmoElapsed2`/`iMagazineSize2`, `SwitchMode`, GL reload). The
    common rifle base for grenade-capable weapons; `CWeaponAK74` is a pure-config leaf on it.
    *(GL mode + CRocketLauncher mechanics agent-traced.)*

- **`CWeaponSSRS`** (`WeaponSSRS.h`) ← `CRocketLauncher` + `CWeaponMagazined` — magazine-fed grenade
  launcher; one live rocket object per loaded round. Reworked 2026.10 (PR #713): reload keeps one rocket
  per round and destroys rockets a reload strips; rounds added outside a reload spawn their rocket;
  rockets respawn from the loaded ammo type; a shot with no rocket is skipped (a round with no grenade
  fires as a dud); with `tri_state_reload` it loads one round at a time; NPCs fire at `ai_rpm`
  (defaults to `rpm`) without changing the weapon's own RPM; grass trace/spread follow the unit fire
  direction. `CRocketLauncher` marks the detached launched rocket.

**Key takeaway for modding:** leaf classes (`WeaponAK74`, `WeaponSVD`, `WeaponPM`, …) are *config on
a spine* — they exist mostly to register a luabind class and occasionally override one virtual
(`switch2_Fire`, `Load`); behavior comes from `.ltx` on the `CWeaponMagazined`/`*Pistol` spine, not
from per-weapon C++.

## Addon model (verified)

A weapon tracks three addon slots; each slot has a **status** and a **runtime on/off flag**:

- **Status** (`ALife::EWeaponAddonStatus` in `m_eScopeStatus`/`m_eSilencerStatus`/
  `m_eGrenadeLauncherStatus`, set from `.ltx` `*_status`): `eAddonDisabled` (slot doesn't exist) ·
  `eAddonPermanent` (always-on, never detachable — `InitAddons` applies it on load) · `eAddonAttachable`
  (player can attach/detach at runtime).
- **Flag** (`m_flagsAddOnState`, `u8` bitfield): `CSE_ALifeItemWeapon::eWeaponAddonScope` /
  `…Silencer` / `…GrenadeLauncher` — set when an attachable addon is currently installed.
- **`IsScopeAttached()`** (`Weapon.cpp`) = `(status==eAddonAttachable && flag set) || status==eAddonPermanent`
  — the single predicate the rest of the engine asks. (`IsSilencer…`/`IsGrenadeLauncher…` identical.)

This two-axis design is why a permanently-scoped weapon and a removable-scope weapon both answer
`IsScopeAttached()==true` but only the latter responds to attach/detach.

### Modular vs section scopes — `GetScopeName` (verified)

`m_scopes` is the list of allowed scope sections; `m_cur_scope` indexes it. `CWeapon::GetScopeName`:
- **modular** (`m_modular_attachments==true`, Anomaly modular-attachment system): the entry **is** the
  scope item section → `m_scopes[m_cur_scope]`.
- **classic**: the entry is a config key holding the field → `pSettings->r_string(m_scopes[m_cur_scope],
  "scope_name")`.
Returns empty `{}` if `m_scopes` is empty. Silencer/GL use fixed names `m_sSilencerName`/
`m_sGrenadeLauncherName` (no per-item list).

## Attach / detach sequence (verified)

`CWeaponMagazined::CanAttach(PIItem)` — `smart_cast` the item to `CScope`/`CSilencer`/
`CGrenadeLauncher`; require the matching status `eAddonAttachable`, the flag **not** already set, and a
**name match** (scope: scan `m_scopes` — modular compares the section directly, classic compares the
`scope_name` field; silencer/GL compare the fixed name). Falls through to `inherited::CanAttach`.
`CanDetach` is the mirror (flag must be set; scope additionally matches via `DetachScope`).

`CWeaponMagazined::Attach(PIItem, b_send_event)`:
1. cast + re-check status/flag/name (same gates as `CanAttach`).
2. scope: set `m_cur_scope` to the matched index; `m_flagsAddOnState |= eWeaponAddonScope`; if modular,
   `LoadScopeKoeffs()` + create a `CAnonHudItem` (`m_scopeItem`) and `g_player_hud->attach_item` it.
   silencer/GL: just OR in the flag.
3. if anything attached: when `b_send_event && OnServer()`, **destroy the addon item**
   (`pIItem->object().DestroyObject()`) — it is consumed into the weapon, not kept in inventory.
4. `UpdateAddonsVisibility()` (toggles `wpn_scope`/`wpn_silencer`/`wpn_grenade_launcher` HUD bones via
   `HudItemData()->set_bone_visible`) then **`InitAddons()`**. Else → `inherited::Attach`.

`Detach(item_section_name, b_spawn_item)` mirrors it: clear the flag, for modular scope
`g_player_hud->detach_item(m_scopeItem)` + delete it and reset zoom if needed, `UpdateAddonsVisibility`
+ `InitAddons`, then `CInventoryItemObject::Detach` **spawns the addon item back** into inventory.

## Re-parameterization — `InitAddons` + koeffs (verified)

`CWeaponMagazined::InitAddons()` is the single re-apply point (called after every attach/detach, on
load, and on `GE_ADDON_CHANGE`):
- **scope attached:** read `scope_texture`, `scope_zoom_factor`, `scope_nightvision` (postprocess),
  `scope_dynamic_zoom`, `scope_alive_detector` (binocular vision), `zoom_step_count` from
  `GetScopeName()`'s section into `m_zoom_params`; (re)build the `m_UIScope` `CUIWindow` from the scope
  texture; `ApplyScopeKoeffs()`. else `ResetScopeKoeffs()` (+ drop `m_UIScope` unless permanent).
- **silencer attached:** swap firing FX — `m_sFlameParticlesCurrent`/`m_sSmokeParticlesCurrent` to the
  silencer variants, `m_sSndShotCurrent = "sndSilencerShot"`, `LoadLights(*cNameSect(), "silencer_")`;
  `ApplySilencerKoeffs()`. else restore base FX + `ResetSilencerKoeffs()`.
- `inherited::InitAddons()` (the `CWeapon` base handles visibility/HUD).

**Koeffs** are multiplier structs the fire chain reads. `LoadSilencerKoeffs`/`LoadScopeKoeffs` read
`*_k` fields (`bullet_hit_power_k`, `bullet_speed_k`, `fire_dispersion_base_k`, `cam_dispersion_k`,
`PDM_disp_*_k`, `crosshair_inertion_k`, `zoom_rotate_time_k`, `condition_shot_dec_k`, …) from the addon
section, default `1.0f`, **clamped to `[0.01, 2.0]`**, into `m_silencer_koef`/`m_scope_koef`.
`Apply*Koeffs` copies that into the *active* `cur_silencer_koef`/`cur_scope_koef`; `Reset*Koeffs`
sets the active struct back to all-`1.0f`. The fire/dispersion math (`flows/weapon-fire-callbacks.md`)
multiplies base values by `cur_*_koef.*`, so a silencer with `fire_dispersion_base_k<1` tightens the
group; scope koeffs cover only camera/PDM/inertion (no damage multiplier — scopes don't change damage).

## Ammo & magazine (verified)

- **`CWeaponAmmo`** (`WeaponAmmo.cpp`) is an inventory **box** of rounds: `Load` reads `box_size`
  (→`m_boxSize`, `m_boxCurr`) plus the per-cartridge ballistics into `cartridge_param`
  (`k_dist`/`k_disp`/`k_hit`/`k_impulse`/`k_ap`/`k_bullet_speed`/`k_air_resistance`/`buck_shot`/
  `impair`/`wm_size` + trait flags tracer/ricochet/explosive/can_be_unlimited/magnetic). `Get(CCartridge&)`
  pops one round: returns `false` if `m_boxCurr==0`, else stamps `m_ammoSect=cNameSect()` +
  `param_s=cartridge_param` and decrements. `net_Spawn` restores `m_boxCurr` from `a_elapsed`.
- **Weapon side** (`Weapon.h`): `m_ammoTypes` (`xr_vector<shared_str>` — the `ammo_class` list the
  weapon accepts), `m_ammoType` (`u8` current index, cycled by `SwitchAmmoType`), `m_magazine`
  (`xr_vector<CCartridge>` — the loaded rounds), `iAmmoElapsed` (== `m_magazine.size()`). Reload
  (`flows/weapon-state-machine.md`) pulls matching `CWeaponAmmo` boxes from inventory and `Get`s rounds
  into `m_magazine`; firing pops the back cartridge. A weapon can hold mixed cartridge types in one
  magazine (per-round `CCartridge`), which is why ballistics are read per-round, not per-weapon.

## Isolation ledger (the seams)

- **Inventory:** addon items are `CInventoryItem`s; attach **consumes** them (server `DestroyObject`),
  detach **respawns** them (`CInventoryItemObject::Detach`). Ammo is pulled from inventory boxes by
  section match (`flows/inventory.md`).
- **HUD:** modular scope creates a `CAnonHudItem` attached to `g_player_hud`; addon bones toggled via
  `HudItemData()->set_bone_visible`; `m_UIScope` is the 2D scope overlay window
  (`flows/hud-ui-load.md`). Built from the scope section's `scope_texture`.
- **Fire chain:** `cur_silencer_koef`/`cur_scope_koef` and the swapped flame/smoke/sound/lights feed
  `flows/weapon-fire-callbacks.md`; zoom params feed the state machine's zoom state.
- **Upgrades:** `WeaponUpgrade.cpp`'s `install_upgrade_impl` → `install_upgrade_addon` can flip a slot's
  `*_status` and rebuild `m_scopes`/names, then `InitAddons()` — the runtime path that makes an upgrade
  "add a scope mount." *(agent-traced — see F26 upgrades flow when written.)*
- **Network:** `m_flagsAddOnState` is serialized in `net_Export`/`net_Spawn`; a `GE_ADDON_CHANGE`
  server event re-applies it via `OnEvent`→`InitAddons` on clients. *(agent-traced.)* This is a network
  *event*, not a Lua callback — there is no `on_weapon_addon` script callback in the weapon layer.
- **Config read:** weapon section (`*_status`, `scopes_sect`/`modular_scope_group`, `silencer_name`,
  `grenade_launcher_name`, `*_x`/`*_y` icon pos, fire params); ammo section (`box_size` + ballistics);
  addon sections (scope optics + the `*_k` koeffs).

## Threading

All of this runs on the main thread (gameplay/inventory/HUD). No worker-thread interaction beyond the
HUD item riding normal animation update (`flows/animation-callbacks.md`).

## Failure & edge cases

- **`LoadScopeKoeffs` fatal:** if `GetScopeName()` returns null for an attachable scope (bad/missing
  `scope_name` in `.ltx`, or `m_cur_scope` out of range), `Debug.fatal` hard-stops — a common config
  error for modular scopes. Empty `m_scopes` makes `GetScopeName` return `{}`.
- **Permanent vs attachable confusion:** an `eAddonPermanent` slot ignores attach/detach (no flag
  bit), so trying to detach a "built-in" scope is a silent no-op / logged error.
- **Modular requires the koeff/HUD-item path:** only `m_modular_attachments` weapons create
  `m_scopeItem` + call `LoadScopeKoeffs` at attach time; classic scopes load koeffs differently — mixing
  the two in config produces a scope with no koeffs or no visual.
- **Ammo type mismatch:** `m_ammoTypes` gates what boxes reload accepts; a magazine can still contain
  rounds of an earlier type (mixed `m_magazine`).
- **Addon name must match exactly:** `CanAttach` compares `cNameSect()` to the configured name/list;
  a renamed addon section silently fails to attach (falls through to `inherited::CanAttach`).

## Script weapon controls (merged 2026.11.09, attachments PR series)

`CWeapon` gained per-instance Lua setters so scripted attachments can replace engine addons (full list
in `lua_help_ex.script`, `class CWeapon`):

- **Zoom** — scope zoom factor / min factor / step count / dynamic zoom / zoom postprocess
  (`ResetCurrentZoomFactor` re-applies), `StopZoom`, `IsTextureZoomActive`, `IsRotatingToZoom`,
  `GetEffectiveMinScopeZoomFactor`. Zoom-inertia factor is Lua-settable (`EffectorZoomInertion`).
- **Shot FX** — `SetFlameParticles`, `SetSmokeParticles`, `SetLightShotDisabled`.
- **Magazine/ammo** — `SetAmmoMagSize`, `SetAmmoType(index|section)`, `GetAmmoName`,
  `GetAmmoCount_forType`.
- **Fire modes** — `GetFireModes()` (Lua table) and `GetFireModeIndex()` (0-based engine index).
- **HUD** — `UseCustomHudOffset` (`m_bHandleCustomHudOffset`, off by default), HUD rotation stored as
  quaternions (`m_hud_rotation`/`m_hud_aim_rotation`, `InterpolateHudRotation`), HUD-inertia
  LRUD offsets / tend-to speeds / min angle getters+setters.
- **Ownership flags** — `m_bUseEngineAttachments`, `m_bHandleZoomParameters` let a script take over
  what the engine addon code would otherwise reset.
- Explosives: `set_initiator(obj)` sets who gets credit for an explosive object.
- Upgrade icon fields `inv_grid_*` / `upgr_icon_*` are now upgradable (`inventory_item_upgrade.cpp`);
  `item:get_upgr_icon_rect()`.
- Dropped items/weapons keep correct position and direction (`inventory_item.cpp`).

## Deeper

- Script attachments that drive these controls: `features/script-attachments.md`
- SSRS: `src/xrGame/WeaponSSRS.cpp`
- Base + predicates + `GetScopeName` + magazine members: `src/xrGame/Weapon.h`, `src/xrGame/Weapon.cpp`
  (`IsScopeAttached`, `IsSilencerAttached`, `IsGrenadeLauncherAttached`, `GetScopeName`,
  `UpdateAddonsVisibility`)
- Attach/detach/koeffs/InitAddons: `src/xrGame/WeaponMagazined.cpp` (`CanAttach`, `CanDetach`, `Attach`,
  `Detach`, `DetachScope`, `InitAddons`, `LoadSilencerKoeffs`, `LoadScopeKoeffs`, `Apply*Koeffs`,
  `Reset*Koeffs`)
- Addon items: `src/xrGame/Scope.cpp`, `src/xrGame/Silencer.cpp`, `src/xrGame/GrenadeLauncher.cpp`
  (`grenade_vel`)
- Ammo: `src/xrGame/WeaponAmmo.cpp` (`Load`, `Get`, `box_size`, `cartridge_param`)
- Upgrades: `src/xrGame/WeaponUpgrade.cpp` (`install_upgrade_impl`, `install_upgrade_addon`) — F26
- Dual-mode/GL: `src/xrGame/WeaponMagazinedWGrenade.cpp` + `RocketLauncher.cpp`
- Mounted gun: `src/xrGame/WeaponStatMgun.cpp` (+ `*Barrel`/`*Fire`/`*Misc`) — F29 holder seam
- Fire/state consumers: `flows/weapon-fire-callbacks.md`, `flows/weapon-state-machine.md`

## Maintenance

- Re-grep `IsScopeAttached` if the status/flag model changes — it is the predicate the whole engine
  uses; keep the two-axis (status × flag) explanation accurate.
- Re-grep `CWeaponMagazined::InitAddons` if addon FX/koeff application moves — it is the single re-apply
  point after attach/detach/load/network.
- Re-verify the inheritance tree by grepping `^class CWeapon` across `*.h` (note: class decls often
  span two lines — `class X :\n  public Y`, so anchor on the class name, then read the next line).
- Promote the *(agent-traced)* items (network addon-sync, `install_upgrade_addon`, WGrenade/GL,
  StatMgun) to verified when F26/F29 or a netcode task needs their bodies.
