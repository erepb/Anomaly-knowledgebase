---
domain: modlayer/gameplay/configs-data
tags: [modlayer, gameplay, config, ltx, data, map]
sources: [unpacked/configs/creatures, unpacked/configs/zones, unpacked/configs/items, unpacked/configs/scripts, unpacked/configs/plugins, unpacked/configs/ai_tweaks, unpacked/configs/gameplay, unpacked/configs/defines.ltx]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, modlayer/config-ltx.md, flows/creatures-actor-configs.md, flows/zones-anomaly-configs.md, flows/items-weapons-outfits-configs.md, flows/per-smart-logic-configs.md, flows/plugins-tweaks-presets-configs.md, flows/spawns-alife-graphs-configs.md, flows/dxml-xml-interception.md, GLOSSARY.md]
updated: 2026-06-21
verified: map
---

# Config / data tier — L1 map (M23)

The **data tier**: the LTX (and a little XML) under `configs/` that the engine and the Lua corpus read
as content — creature stats, zone params, item/weapon/outfit definitions, per-smart logic, plugin/tweak
settings, and the spawn/ALife graph data. This is not code; it is the parameter surface mods edit. This
map routes the Phase G9 flows; the **mechanism** for how LTX is parsed, inherited (`[child]:parent`,
`#include`), and patched (DLTX deltas) is `modlayer/config-ltx.md` + `flows/dxml-xml-interception.md`.

## The pieces

| Cluster | Path(s) | Role | Flow |
|---------|---------|------|------|
| creatures & actor | `configs/creatures/` | monster species (`m_*`), `actor.ltx`, `immunities`/`damages` profiles, `game_relations`, `spawn_sections_*`, `helicopter` | F118 ✅ (`flows/creatures-actor-configs.md`) |
| zones & anomalies | `configs/zones/`, `sr_teleport_sections.ltx` | anomaly/zone field params + the level-changer table | F119 ✅ (`flows/zones-anomaly-configs.md`) |
| items / weapons / outfits | `configs/items/` (weapons/`w_*`, outfits/`o_*`, items/`items_*`, ammo, addons, upgrades) | item & weapon & outfit & ammo section definitions; glob auto-include modding contract | F120 ✅ (`flows/items-weapons-outfits-configs.md`) |
| per-smart logic | `configs/scripts/*.ltx` (+ per-level subdirs, `smart/`, templates) | object logic + gulag job catalogs + smart respawn defs (the `custom_data` / logic-LTX tier) | F121 ✅ (`flows/per-smart-logic-configs.md`) |
| plugins / tweaks / presets | `configs/plugins/`, `ai_tweaks/`, `presets/`, `defines.ltx` | the constants tier: inheritance root (`defines.ltx`), preset bundles, per-system tuning data (`axr_options.ltx` empty) | F122 ✅ (`flows/plugins-tweaks-presets-configs.md`) |
| spawns / ALife / graphs | `spawns/all.spawn` (binary), `alife.ltx`, `game_graphs.ltx`, `game_story_ids.ltx`, `game_maps_single.ltx` | the world-instance + ALife tier: object placements, simulator tuning, graph/story/level registries | F123 ✅ (`flows/spawns-alife-graphs-configs.md`) |

## How it fits

The LTX tier sits **under** everything else: the engine reads `system_ini` (all the `#include`-d
configs) at load, and the Lua corpus reads named sections/keys through `ini_file` / the `utils_data`
parsers (`flows/utils-libraries.md`, F115) and the `cfg_file` writer for runtime stores. Section
inheritance (`[child]:parent`) and includes assemble the final values; mods layer changes as **DLTX**
deltas (small `.ltx` patches) rather than full-file overrides, which is why most of this tier has **no
`gamedata/` full copies**. `defines.ltx` holds the shared base sections (e.g. `monster_base`) and global
constants the rest inherit from.

## Subsystem deep-dives (delegates to flows)

- Creatures & actor stats / immunities / damages → `flows/creatures-actor-configs.md` (F118) ✅
- Zones & anomaly params (`zone_*.ltx` class mapping + param groups) + the `sr_teleport_sections.ltx`
  level-changer table → `flows/zones-anomaly-configs.md` (F119) ✅
- Items/weapons/outfits/ammo/addons/upgrades (`configs/items/` + the glob auto-include contract) →
  `flows/items-weapons-outfits-configs.md` (F120) ✅
- Per-smart logic (`configs/scripts/` custom_data / logic-LTX: object logic + gulag job catalogs + smart
  respawn defs) → `flows/per-smart-logic-configs.md` (F121) ✅
- Plugins/tweaks/presets (the constants tier: `defines.ltx` inheritance root + `presets/` bundles +
  `plugins/`/`ai_tweaks/` per-system tuning) → `flows/plugins-tweaks-presets-configs.md` (F122) ✅
- Spawns/ALife/graphs (`all.spawn` binary placements + `alife.ltx` tuning + graph/story/level registries)
  → `flows/spawns-alife-graphs-configs.md` (F123) ✅ — the final M23 flow
- The LTX/DLTX parse + inherit + patch mechanism → `modlayer/config-ltx.md`,
  `flows/dxml-xml-interception.md`

## Maintenance

- **M23 DOMAIN COMPLETE.** F118 ✅ (creatures & actor), F119 ✅ (zones & anomalies), F120 ✅ (items/weapons/
  outfits), F121 ✅ (per-smart logic), F122 ✅ (plugins/tweaks/presets), F123 ✅ (spawns/ALife/graphs — the
  final flow). All six config/data-tier flows are written; this map routes them. The whole gameplay corpus
  (M1–M23 + F65–F123) is now complete.
- Cross-tier ownership: `game_relations.ltx` is F40/F108, not re-documented under F118; similarly when a
  config file is already owned by an engine/gameplay flow, route to it rather than duplicating.
- DLTX means **no `gamedata/` full overrides** are expected here — if one appears, it is a heavy-handed
  mod; flag it. Re-check `gamedata/configs/` on pickup.
