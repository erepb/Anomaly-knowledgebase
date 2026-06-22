---
domain: flows/items-weapons-outfits-configs
tags: [modlayer, gameplay, unpacked, config, ltx, data, items, weapons, outfits, ammo, flow]
sources: [unpacked/configs/items/item_base.ltx, unpacked/configs/items/weapons/base.ltx, unpacked/configs/items/weapons/w_ak74.ltx, unpacked/configs/items/weapons/weapon_ammo.ltx, unpacked/configs/items/weapons/addon_scopes.ltx, unpacked/configs/items/outfits/base.ltx, unpacked/configs/items/outfits/base_includes.ltx, unpacked/configs/items/outfits/o_seva.ltx, unpacked/configs/items/items/base.ltx, unpacked/configs/item_upgrades.ltx, unpacked/configs/items/trade/trade_bandit.ltx]
related: [flows/weapon-hierarchy-addons.md, flows/outfit-helmet.md, flows/item-base.md, flows/upgrade-system.md, flows/consumables-explosives.md, flows/detector-artefact.md, flows/item-behaviors-lua.md, flows/trade-economy.md, flows/creatures-actor-configs.md, flows/zones-anomaly-configs.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: items / weapons / outfits configs (F120)

The **`configs/items/` data tier** — the LTX that defines every weapon, outfit, helmet, ammo, addon,
upgrade, and inventory item the engine and Lua corpus read as content. This is the largest config cluster
(weapons alone ≈ 134 files). The flow documents the **include graph, the wildcard auto-include pattern, the
section inheritance + variant/attach-state convention, and which cluster each engine/Lua flow owns** — not
a per-field dump; what each field does at runtime is owned by F22 (weapons), F23 (outfits), F21 (item
base), F24 (artefacts/devices), F25 (consumables/explosives), F26 (upgrades), F101 (trade), F102 (item Lua
behaviors).

**Provenance:** **verified, no agent** — `item_base.ltx`, `weapons/base.ltx`, `outfits/base.ltx` +
`base_includes.ltx`, `items/items/base.ltx`, `item_upgrades.ltx` read in full; `w_ak74.ltx` (representative
weapon, full), `o_seva.ltx` (representative outfit), `weapon_ammo.ltx` head, `addon_scopes.ltx` head,
`trade_bandit.ltx` head read; the shared base parents located in `configs/defines.ltx`; directory tree +
counts listed. The ~130 `w_*` / 22 `o_*` / 29 `items_*` are leaves on the same per-family pattern (not each
re-read); runtime meaning routed to the engine flows. **No `gamedata/` override** of `configs/items/` at
writing (no `gamedata/configs/items/` dir).

## Include graph (verified)

`configs/system.ltx` `#include`s `items\item_base.ltx`, a tiny manifest that pulls the four family roots:
`weapons\action_fuse.ltx` + `weapons\base.ltx`, `outfits\base.ltx` + `outfits\base_includes.ltx`, and
`items\base.ltx`. (MP weapons/ammo/outfit live separately under `mp\weapons_mp\`.) Each family root then
fans out — see the auto-include pattern below. The shared **base parent sections** every item inherits from
(`default_weapon_params`, `identity_immunities`, `hud_base`, `global_item_actions`, `default_item_params`,
`weapon_probability`) live in **`configs/defines.ltx`** (F122), not here.

## The wildcard auto-include pattern (verified — the defining convention)

The cluster is built so **a new file dropped in with the right prefix is loaded automatically** — no edit
to existing files, no merge conflict with other mods. Each family root ends with a glob include:

- `weapons/base.ltx` → `#include "w_*.ltx"` (guns) + `#include "m_*.ltx"` (misc: knife, bolt, grenades,
  binoc, mounted).
- `outfits/base_includes.ltx` → `#include "o_*.ltx"`.
- `items/items/base.ltx` → `#include "items_*.ltx"`.

Each root carries an explicit **"NOTICE FOR MODDERS: do NOT edit the `w_*`/`items_*` files, make a new
`w_mygun.ltx`/`items_mymod.ltx`"** comment. The long *commented-out* explicit include lists below the glob
are legacy documentation of what ships, not active. `weapons/base.ltx` also includes the addon/upgrade
support files (`weapon_addons`/`silencers`/`scopes`/`sounds`/`ammo`, `upgrades_properties`,
`upgrades\upgrade_presets`) and the global `[bullet_manager]` + tracer-color tables before the gun glob.

## Weapons — `w_*.ltx` (verified on `w_ak74`)

Each gun file defines a **root section** `[wpn_xxx]:identity_immunities,weapon_probability,
default_weapon_params,wpn_xxx_sounds` plus a parallel **HUD section** `[wpn_xxx_hud]:hud_base`, then a
fan of **variant subsections by inheritance**:

- **Cosmetic/stat variants** `[wpn_ak74_alt]:wpn_ak74`, `_custom`, `_modern`, `_old`, `_rpk` — each
  overriding visual/cost/`installed_upgrades`/ballistics deltas.
- **Attach-state variants** `[wpn_ak74_1p29]:wpn_ak74`, `_kobra`, `_ps01` — the same gun pre-fitted with a
  scope (different `scope_status`/`scope_texture`/`visual`), and the matching `_hud` chain.

Parameter clusters on the root: **identity** (`class = WP_*`, `slot`, `kind`, `weapon_class`, `cost`,
`inv_weight`, `repair_type`), **ballistics** (`hit_power`/`hit_type`/`fire_distance`/`bullet_speed`/`rpm`/
`fire_dispersion_*`), **magazine/ammo** (`ammo_class` list, `ammo_mag_size`, `fire_modes`), **attach
slots** (`scopes`+`scope_status`, `silencer_name`+`silencer_status`, `grenade_launcher_name`+
`grenade_class`+`grenade_launcher_status`), **condition/jam** (`misfire_*`, `condition_*_shot_dec`),
**camera recoil** (`cam_*`/`zoom_cam_*`/`PDM_*`), **AI hit chance** (`hit_probability_gd_novice/stalker/
veteran/master`), **upgrades** (`upgrades`/`installed_upgrades`/`upgrade_scheme` + the `upgr_icon_*` grid),
and the **HUD** section (model `item_visual`, `hands_position`, `aim_hud_offset_*`, the `anm_*` animation
name map driving the weapon state machine). Runtime owner: F22 (hierarchy/addons) + the weapon state
machine + fire callbacks; the Lua overheat/ammo-aggregation fork is F102 (`item_weapon`).

`attach status` convention (scopes/silencer/GL): `0` = attachable, `1` = permanently fitted on this
section, `2` = integral/built-in.

## Outfits & helmets — `o_*.ltx` (verified on `o_seva`)

`outfit_base` (in `outfits/base.ltx`, `:global_item_actions`) sets `class = EQU_STLK`, `slot 6`,
`repair_type outfit`, `helmet_avaliable`/`backpack_avaliable`/`sprint_allowed` defaults, and the
**`use1_functor = gameplay_disguise.menu_patch`** disguise context-menu hook (Lua, F102/F104). Each outfit
is `[xxx_outfit]:outfit_base[,outfit_restrictions_N]` + an inline `[sect_xxx_immunities]`. Per-outfit
clusters: visuals (`visual`/`actor_visual`/`npc_visual`/`character_portrait`/`player_hud_section`),
`community`/`kind`/`cost`/`inv_weight`, the **protection block** (`burn_/shock_/radiation_/chemical_burn_/
telepatic_/strike_/explosion_/wound_/fire_wound_protection`) **plus** a separate `immunities_sect` of
`*_immunity` coeffs, `bones_koeff_protection` (→ a `damages.ltx` bone profile, F118), `hit_fraction_actor`,
`additional_inventory_weight`(2), `artefact_count` (container slots), `control_inertion_factor`,
`power_loss`, and the upgrade tree. `outfit_restrictions_1..5` are helmet/backpack/sprint gating presets;
helmets are `o_helmet.ltx`. Runtime owner: F23. **Quirk:** `outfit_restrictions_temp` is a documented dummy
"leave it last" workaround — the engine fails to read the last section before the includes.

## Items — `items_*.ltx` (verified manifest)

The consumable/misc catalog, one file per category: `items_food`/`drink`/`drugs`/`medical`/`cooking`/
`repair`/`tools`/`parts`/`money`/`quest`/`letters`/`trash`/`monster`/`explosives`(+`_new_mines`)/
`attachments`/`container_*`/`patches`/`devices`/`artefacts`(+`_junk`/`_soc`)/`anim`(ations)/`upgrades`.
Generic item base params live in `default_item_params` (defines.ltx, F122). Cross-tier owners: artefacts →
F24; consumables/drugs/food/explosives → F25; devices/detectors → F24/F102; the Lua use-behavior plugins
that hook these sections → F102.

## Ammo & addons (verified)

- **Ammo** `weapon_ammo.ltx`: `[ammo_base]:identity_immunities,default_weapon_params` (`class = AMMO`,
  `kind = w_ammo`, `tier`, `box_size`) + per-cartridge leaves with the **`k_*` ballistic multipliers**
  (`k_hit`/`k_impulse`/`k_ap`/`k_dist`/`k_disp`/`k_bullet_speed`/`k_air_resistance`), `buck_shot`,
  `tracer`/`tracer_color_ID`, `disassemble_parts` (crafting). The `k_*` are what the bullet sim reads
  (F22).
- **Addons** `weapon_addons`/`silencers`/`scopes`/`sounds`.ltx + `addon_scopes.ltx` (scope behavior
  presets: `scope_detector`, `scope_nightvision`/`_2gen` with `pp_eff_name = *.ppe` NV post-process refs),
  `upgrades_properties.ltx`. Runtime owner: F22.

## Upgrades (verified)

Top-level `item_upgrades.ltx` is the master that `#include`s `weapons/action_fuse.ltx`,
`weapons/upgrades_properties.ltx`, `weapons/upgrades/upgrade_presets.ltx`, and
`outfits/upgrades/upgrade_includes.ltx`. Per-item upgrade trees are `up_gr_*` section families with an
`upgrade_scheme_*`; an item's root lists its `upgrades`/`installed_upgrades`/`upgrade_scheme`. Runtime
owner: F26 (`flows/upgrade-system.md`).

## Cross-tier ownership (route, don't duplicate)

- **Trade** `items/trade/trade_*.ltx` (35 profiles) + `presets/` (`trade_presets.ltx` buy/sell stock,
  `trade_discounts.ltx`) — each is a `[trader]` section (`buy_condition`/`sell_condition`/`buy_supplies`
  condlist + exponents + discounts) over `supplies_N` condlist-gated stock lists. **Owned by F101**
  (`flows/trade-economy.md`).
- **`items/settings/`** — data for various Lua systems, mostly already owned: `treasure_manager.ltx`
  (F117), `itms_manager.ltx` (F102), `death_generic`/`death_items`/`death_outfits.ltx` (death drops, F102
  death_manager), `mutant_loot.ltx`, `ph_box_*.ltx` (box loot, F89), `dynamic_item_spawn.ltx` (F106
  game_setup), `new_game_loadouts.ltx` + `npc_loadouts/` (spawns/ALife, F123), `craft.ltx`/`cook.ltx`/
  `parts.ltx`/`devices.ltx`/`artefacts.ltx` (item subsystems). Route to each owner.
- **Shared base parents** (`default_weapon_params`/`identity_immunities`/`hud_base`/`global_item_actions`/
  `default_item_params`/`weapon_probability`) → `defines.ltx` (F122).

## Isolation ledger (the seams)

- **Weapon engine (F22):** `class = WP_*` + ballistics/ammo/addon/condition params → the weapon hierarchy +
  fire callbacks + state machine; `k_*` ammo coeffs feed the bullet sim.
- **Outfit engine (F23):** `class = EQU_STLK` + protection/immunities/bones_koeff blocks.
- **Item base (F21) / consumables (F25) / artefacts+devices (F24):** the `items_*` categories.
- **Upgrades (F26):** `up_gr_*` trees + `upgrade_scheme_*`.
- **Lua item behaviors (F102):** `use1_functor`/use-action functors (e.g. `gameplay_disguise.menu_patch`
  on outfits), the `item_weapon` overheat/ammo-aggregation fork, death-drop + itms_manager settings.
- **Trade (F101):** `items/trade/` profiles.
- **Damage (F118):** outfit `bones_koeff_protection` → `damages.ltx` bone profiles; immunities mirror the
  creature immunity model.
- **Defines/presets (F122):** the shared base parent sections.
- **LTX mechanism:** `#include` + glob auto-include + `[child]:parent` inheritance + DLTX deltas →
  `modlayer/config-ltx.md`.

## Failure & edge cases

- **Add content via a new prefixed file**, never by editing `w_*`/`items_*`/`o_*` — the glob picks it up;
  editing shipped files breaks mod compatibility (the corpus says so explicitly).
- **Read variants bottom-up:** a gun's `_alt`/`_custom`/scope subsections inherit the root; effective stats
  are the merge. Same for outfit `:outfit_base,outfit_restrictions_N`.
- **`outfit_restrictions_temp` dummy must stay last** — an engine last-section-before-includes parse bug.
- **Two protection mechanisms on outfits:** the `*_protection` block and the separate `immunities_sect`
  `*_immunity` coeffs — both exist; don't assume one.
- **Trade and settings files live here but aren't owned here** — route to F101/F117/F102/F123 etc.
- **DLTX, not full overrides:** mods patch with deltas; no `gamedata/configs/items/` at writing. A full
  override dir would be heavy-handed — flag it.

## Deeper

- Include roots: `items/item_base.ltx`, `items/weapons/base.ltx`, `items/outfits/base.ltx` +
  `base_includes.ltx`, `items/items/base.ltx`, `item_upgrades.ltx`
- Exemplars: `items/weapons/w_ak74.ltx` (gun + variants + HUD), `items/outfits/o_seva.ltx` (outfit +
  immunities), `items/weapons/weapon_ammo.ltx` (ammo `k_*`), `items/weapons/addon_scopes.ltx` (scope/NV
  presets), `items/trade/trade_bandit.ltx` (trade profile, F101)
- Runtime owners: `flows/weapon-hierarchy-addons.md` (F22), `flows/outfit-helmet.md` (F23),
  `flows/item-base.md` (F21), `flows/upgrade-system.md` (F26), `flows/consumables-explosives.md` (F25),
  `flows/detector-artefact.md` (F24), `flows/item-behaviors-lua.md` (F102), `flows/trade-economy.md` (F101)
- Map: `modlayer/gameplay/configs-data.md` (M23); LTX mechanism: `modlayer/config-ltx.md`

## Maintenance

- Roots + one exemplar per family read in full; the rest are leaves on the same pattern. If a new
  `class = WP_*`/`EQU_STLK`/etc. appears, confirm the engine subclass in the owning flow before
  documenting.
- Re-check `gamedata/configs/items/` for overrides per session (none at writing).
- Keep the load-bearing notes: the **glob auto-include** modding contract, the **variant/attach-state
  subsection** inheritance convention, and the **cross-tier ownership** of trade/settings/defines (route,
  don't dup).
