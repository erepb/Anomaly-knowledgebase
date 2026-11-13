---
domain: modlayer/config
tags: [modlayer, config, dltx, ltx]
sources: [gamedata/configs/, gamedata/configs/mod_system_anomaly_fixes.ltx, gamedata/scripts/ltx_help_ex.script, src/xrCore/Xr_ini.cpp]
related: [modlayer/_overview.md, engine/xrCore.md, flows/dltx-config-load.md, flows/vehicles-holder.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# LTX configs & DLTX

How the mod layer changes game config without overwriting base files. DLTX mechanics verified in
`flows/dltx-config-load.md`; the extra-param catalog verified against `ltx_help_ex.script` + engine
consumers (below).

## LTX format

`.ltx` = sectioned ini: `[section]`, `key = value` (comma lists), `#include`, inheritance
`[child]:parent`. Parsed by `src/xrCore/Xr_ini.cpp` (`CInifile`). Read from Lua via
`ini_file`/`ini_file_ex` (e.g. `axr_main.script` uses `ini_file_ex("axr_options.ltx", true)`).

## DLTX patching (key-level merge, not replace)

> Full parse/merge sequence + proof: **`flows/dltx-config-load.md`**.

Mods drop `mod_<basefile>_<modname>.ltx` next to the base; the engine discovers and merges them at
load. Section operators merge **key-by-key onto the base** (`MergeSections`/`EvaluateSection`):
`![section]` = override fragment for an existing section, `@[section]` = same but creates the section
if missing; listed keys replace the base's, unmentioned base keys are kept, new keys added. Value
operators: `>key = a,b` append to a list, `<key = a` remove from a list, `key = DLTX_DELETE` delete.
A duplicate plain `[section]` (no `!`/`@`) is a fatal.

This repo's own `mod_*.ltx` (root of `configs/`, re-listed 2026-11-13): `mod_system_base_hud.ltx`,
`mod_system_knife.ltx`, `mod_system_flashlight_definition.ltx`, `mod_system_shader_crosshair.ltx`,
`mod_system_spawn_antifreeze_ignore.ltx`, `mod_system_anomaly_fixes.ltx`, `mod_script_dxml.ltx`,
`mod_spawn_overlays_meadow.ltx` (a DLTX mod of `spawn_overlays.ltx`, see `features/spawn-overlays.md`),
plus the non-mod roots `patrol_paths.ltx` and `spawn_overlays.ltx`. Bolts are no longer in the spawn
antifreeze ignore list. `mod_system_signal_light.ltx` was **deleted** upstream (signal
light / lamp LTX fix) — don't look for it.

`mod_system_anomaly_fixes.ltx` is a deliberate collection point: base-Anomaly config *bugs* fixed from
the engine repo rather than in the mod's own feature files. Its first (and so far only) fragment is
`![ammo_base]` pinning the five ballistic factors explicitly — `k_dist`/`k_disp`/`k_hit`/`k_impulse` at
1.0 and `k_ap` at 0.0. Base `ammo_base` (`configs/items/weapons/weapon_ammo.ltx`) declares the first
four but **not `k_ap`**, so every ammo section inheriting it relied on the engine's default; writing all
five out keeps the inherited baseline explicit. Add further base-game config fixes here, one `!`/`@`
fragment per section.

## Extra params added by modded exes

`gamedata/scripts/ltx_help_ex.script` is a **comment-only documentation file** cataloguing LTX params
the modded exes added that base Anomaly doesn't read. Its categories (re-read 2026-08-17): Objects,
Outfits/Actor, Weapons, Grenades, Explosives, PDAs, Ammo, Detectors, Flashlight definitions, Anomalies,
Particles (`.pe`), `WeaponStatMgun`, Projector, and **Car** (the `CAR_NEW` drone / visual-camera /
camera-scope blocks, part in the car's own section and part in the visual's `userdata` — see
`flows/vehicles-holder.md`). Recent additions to watch: `bullet_check_visual` under Objects
(`flows/damage-hit.md`).

The catalog reflects real engine reads — spot-verified (10/≈80 params traced to their C++ consumer):

| param | engine consumer (verified) |
|---|---|
| `ignore_collision` | `src/xrGame/PhysicsShellHolder.cpp`, `PhysicObject.cpp` |
| `legs_visual` | `src/xrGame/player_hud_legs.cpp` |
| `base_hud_offset_pos` | `src/xrGame/player_hud.cpp` |
| `silenced_tracers`, `modular_attachments` | `src/xrGame/Weapon.cpp` |
| `ai_rpm` | `src/xrGame/WeaponSSRS.cpp` (SSRS only) |
| `nearwall_zoomed_range` | `src/xrGame/PDA.cpp` (also weapons) |
| `frags_ap` | `src/xrGame/Explosive.cpp` |
| `tele_type` | `src/xrGame/TeleWhirlwind.cpp` |
| `detector_ui_bone` | `src/xrGame/EliteDetector.cpp` |
| `update_step` | `src/Layers/xrRender/ParticleEffectDef.cpp` |

Treat `ltx_help_ex.script` as the catalog; to confirm a specific param is read, grep its name in
`src/` (consumer is usually `xrGame`). *(The remaining ≈70 documented params were not each
individually traced this session, but every one of the 10 sampled across different categories
resolved to a real consumer.)*

## Deeper

- Parser + DLTX engine code: `src/xrCore/Xr_ini.cpp` → `flows/dltx-config-load.md`, `engine/xrCore.md`
- Param catalog (comments): `gamedata/scripts/ltx_help_ex.script`
- Repo's config patches: `gamedata/configs/mod_*.ltx`, `gamedata/configs/ui/`, `gamedata/configs/text/`
- Lua config access: `ini_file_ex` usage in `gamedata/scripts/axr_main.script`

## Maintenance

- `ltx_help_ex.script` is comments only — when documenting a new param, grep its name in `src/` to
  confirm/locate the consumer before claiming the engine reads it.
- Keep DLTX operator wording identical to `flows/dltx-config-load.md`, `engine/xrCore.md`,
  `GLOSSARY.md#dltx`.
