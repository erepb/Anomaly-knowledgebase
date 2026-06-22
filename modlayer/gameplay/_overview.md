---
domain: modlayer/gameplay
tags: [modlayer, gameplay, unpacked, lua, schemes, map]
sources: [unpacked/scripts/, unpacked/configs/, gamedata/scripts/, gamedata/configs/]
related: [INDEX.md, GLOSSARY.md, ROADMAP-gameplay.md, METHODOLOGY.md, modlayer/_overview.md, modlayer/scripting.md, modlayer/config-ltx.md, flows/gameplay-boot.md, flows/scheme-engine-core.md, flows/db-registry.md, flows/ai-squad-logic.md, flows/script-npc-actions.md, flows/dialog-trade.md, flows/dialog-system-lua.md, modlayer/gameplay/dialogs-info.md, flows/trade-economy.md, modlayer/gameplay/trade-economy.md, flows/item-behaviors-lua.md, modlayer/gameplay/items-lua.md, flows/actor-survival-status.md, flows/actor-menu-inventory-ui.md, modlayer/gameplay/actor-survival.md, flows/warfare-metagame.md, modlayer/gameplay/warfare.md, flows/game-modes.md, modlayer/gameplay/gamemodes-services.md, flows/gameplay-ui-screens.md, modlayer/gameplay/ui-scripts.md, flows/pda-tabs.md, flows/ui-debug-tools.md, flows/ui-news-tips.md, flows/axr-behaviour-addons.md, flows/axr-managers.md, flows/companion-system.md, modlayer/gameplay/axr-addon.md, flows/utils-libraries.md, modlayer/gameplay/utils-db.md, flows/db-registry.md, flows/sound-music.md, flows/treasures-secrets.md, flows/creatures-actor-configs.md, flows/zones-anomaly-configs.md, flows/items-weapons-outfits-configs.md, flows/per-smart-logic-configs.md, flows/plugins-tweaks-presets-configs.md, flows/spawns-alife-graphs-configs.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, flows/tasks.md, flows/relations-info.md, flows/faction-relations-lua.md, flows/special-actors.md, flows/smart-cover.md, flows/alife-services.md]
updated: 2026-06-12
verified: map
---

# Gameplay corpus (`unpacked/`) — L1 map

The game's **gameplay half**: ~456 base-game Lua scripts in `unpacked/scripts/` + the
`unpacked/configs/` data tier. Where `src/` is the machine, this corpus is what the machine *plays* —
NPC behavior, jobs, squads, tasks, dialogs, trade, survival, the warfare metagame, and UI. This map is
the routing hub; the full plan (23 maps M1–M23 + 59 flows F65–F123) is **`ROADMAP-gameplay.md`**, the
file cold sessions execute. Engine cross-references are normal `related:` links.

**Read first:** `METHODOLOGY.md` — the `unpacked/` (canonical, read-only) vs `gamedata/` (modded delta,
wins at runtime) dual-cite rule. Always grep `gamedata/scripts/` for an override of any `unpacked/` file
before trusting it.

## The architecture (how the corpus fits together)

1. **Boot** (`flows/gameplay-boot.md`, F65) — the engine calls `_g.start_game_callback`, which runs
   `axr_main.on_game_start` (module orchestrator) and boots the core managers; `class_registrator` binds
   C++ classes to Lua server/client classes; `_g.LoadScheme` registers behavior schemes.
2. **Global state** (`flows/db-registry.md`, F69) — `db.script`: the per-object `storage[id]` blackboard
   + the actor ref + category lookup tables. Binders write it; everything reads it.
3. **Scheme engine** (`flows/scheme-engine-core.md`, F66) — `xr_logic` + the per-NPC `xr_motivator`
   binder: holds the active scheme, evaluates condlist triggers, switches logic sections each frame.
4. **Schemes** (roadmap F74–F82) — the `xr_*` catalog (walker, camper, combat, animpoint, sleeper,
   meet, …): each registers a scheme name and an action/evaluator bound on the NPC.
5. **Jobs & squads** (F72/F73/F85/F86) — `smart_terrain` + the gulag job board assign which logic LTX an
   NPC runs; `sim_board`/`sim_squad_*` simulate squads. (Bridge: `flows/ai-squad-logic.md` F12.)
6. **Binders & object schemes** (F88–F93) — `bind_*` attach Lua to engine objects; `sr_*` restrictor and
   `ph_*` physics-object schemes are logic on non-NPC objects.
7. **Systems on top** — tasks (F97/F98, `flows/tasks.md`), dialogs/info (F99 ✅
   `flows/dialog-system-lua.md` + F100 ✅ `flows/characters-info-encyclopedia.md`, engine
   `flows/dialog-trade.md`), trade/economy (F101 ✅ `flows/trade-economy.md`),
   items (F102 ✅ `flows/item-behaviors-lua.md`), actor survival (F103 ✅
   `flows/actor-survival-status.md`) + inventory UI (F104 ✅
   `flows/actor-menu-inventory-ui.md`) + UI screens/PDA/debug/news (F109–F112 ✅,
   M20 `modlayer/gameplay/ui-scripts.md`), warfare (F105 ✅ `flows/warfare-metagame.md`),
   gamemodes (F106 ✅ `flows/game-modes.md`) + game services (F107 ✅
   `flows/game-services.md`), UI/PDA (F109/F110), the `axr_*` addon layer (F113/F114), `utils_*`
   (F115), and the config/data tier (F118–F123).

## Subsystem deep-dives (delegates to flows)

The L1 sub-maps M2–M23 are scaffolded **just-in-time** as each domain's first flow lands (rather than
23 upfront stubs — see ROADMAP-gameplay maintenance note). Until a sub-map exists, route via the flow:

- Boot / load order → `flows/gameplay-boot.md` (F65) ✅
- Global registry → `flows/db-registry.md` (F69) ✅
- Scheme engine → `flows/scheme-engine-core.md` (F66) ✅
- AI/job seam (existing bridge) → `flows/ai-squad-logic.md` (F12), `flows/script-npc-actions.md` (F37)
- Smart-cover (existing) → `flows/smart-cover.md` (F33); special actors → `flows/special-actors.md` (F36)
- Dialog/trade (existing) → `flows/dialog-trade.md` (F18); tasks → `flows/tasks.md` (F38)
- Relations/info engine tier (existing) → `flows/relations-info.md` (F40); Lua faction relations
  (DoctorX dynamic relations + goodwill API) → `flows/faction-relations-lua.md` (F108) ✅; ALife
  services → `flows/alife-services.md` (F41)
- Lua server entities (`se_*`) → `modlayer/gameplay/server-entities-lua.md` (M6) ✅ + `flows/se-stalker.md`
  (F83) ✅ + `flows/se-others.md` (F84, whole family) ✅
- Squad simulation (`sim_*`) → `modlayer/gameplay/squads-simulation.md` (M5) ✅ + `flows/scripted-squad.md`
  (F85) ✅ + `flows/sim-board.md` (F86) ✅ + `flows/offline-combat.md` (F87) ✅ — domain complete
- Binders (`bind_*`, how Lua attaches to objects) → `modlayer/gameplay/binders.md` (M7) ✅ +
  `flows/binder-architecture.md` (F88) ✅ + `flows/world-object-binders.md` (F89) ✅ +
  `flows/anomaly-field-binders.md` (F90) ✅ — domain complete
- Restrictor schemes (`sr_*`) → `modlayer/gameplay/restrictors-sr.md` (M10) ✅ +
  `flows/restrictors-passive-sr.md` (F91) ✅ + `flows/restrictors-active-sr.md` (F92) ✅ — family complete
- Physics-object schemes (`ph_*` + phantom_manager) → `modlayer/gameplay/physics-schemes-ph.md` (M11) ✅ +
  `flows/physics-object-schemes.md` (F93) ✅ — family complete
- Monster schemes (`mob_*` + trader schemes) → `modlayer/gameplay/monster-schemes.md` (M9) ✅ +
  `flows/monster-schemes.md` (F94) ✅ — family complete
- Vehicles (heli stack + car binding) → `modlayer/gameplay/vehicles-heli.md` (M12) ✅ +
  `flows/helicopter-ai.md` (F95) ✅ (F96 cars merged into M12) — domain complete
- Tasks (Lua quest tier) → `modlayer/gameplay/tasks-lua.md` (M13) ✅ + `flows/task-system-lua.md`
  (F97) ✅ + `flows/dynamic-task-types.md` (F98) ✅; AXR F114 ⬜
- Dialogs & character info → `modlayer/gameplay/dialogs-info.md` (M14) ✅ +
  `flows/dialog-system-lua.md` (F99) ✅ + `flows/characters-info-encyclopedia.md` (F100) ✅ —
  domain complete
- Trade & economy → `modlayer/gameplay/trade-economy.md` (M15) ✅ + `flows/trade-economy.md`
  (F101) ✅ (engine tier F18, trader schemes F94; F114 adjacency ⬜)
- Item behaviors → `modlayer/gameplay/items-lua.md` (M16) ✅ + `flows/item-behaviors-lua.md`
  (F102) ✅ (engine inventory F8; device binder pattern F88; **gamedata fork: item_weapon**)
- Actor survival & status → `modlayer/gameplay/actor-survival.md` (M17) ✅ +
  `flows/actor-survival-status.md` (F103) ✅ (engine condition tick F30; psy hit hijack — engine
  psy bar is display-only) + `flows/actor-menu-inventory-ui.md` (F104) ✅ (engine ActorMenu
  vetoed; Lua UIInventory; `useN_*` LTX context rows) — domain complete
- Warfare metagame → `modlayer/gameplay/warfare.md` (M18) ✅ + `flows/warfare-metagame.md`
  (F105) ✅ (stock squad AI locked out per-squad via `__lock`; capture/economy/commander;
  `IsWarfare()` root)
- Game modes & services → `modlayer/gameplay/gamemodes-services.md` (M19) ✅ +
  `flows/game-modes.md` (F106) ✅ (flag seeding, factor families, world-item loop,
  ironman/azazel death rules) + `flows/game-services.md` (F107) ✅ (rank/reputation engine,
  achievements [pull-based PDA-tab poll], autosave, fast/backpack travel) — domain complete
- UI scripts → `modlayer/gameplay/ui-scripts.md` (M20) ✅ + `flows/gameplay-ui-screens.md` (F109) ✅
  (main menu + save-veto/callback seams, HUD widgets, message-box toolkit; dead ui_extra, dormant
  Lua dosimeter) + `flows/pda-tabs.md` (F110) ✅ (PDA hub set_active_subdialog dispatch + ranking
  board + tabs: contacts/npc/radio/relations/warfare) + `flows/ui-debug-tools.md` (F111) ✅
  (DEV_DEBUG-gated launcher + xrs_debug_tools HUD inspector + command registry + gamedata debug
  delta) + `flows/ui-news-tips.md` (F112) ✅ (news_manager tip/showcase dispatcher + Dynamic News
  ambient generator: channels/ticks/NewsToggle gate) + `flows/actor-menu-inventory-ui.md` (F104,
  inventory window) — ui_* flow coverage complete
- AXR addon layer → `modlayer/gameplay/axr-addon.md` (M21) ✅ **domain complete** +
  `flows/axr-behaviour-addons.md` (F113) ✅ (Alundaio's scripted GOAP schemes on the F37 interface:
  `axr_beh` companion escort/patrol brain + `axr_beh_patches` hook + combat/utility schemes [cover-fight,
  panic, anti-heli, campfire] + `axr_keybind`) + `flows/axr-managers.md` (F114) ✅ (the three managers:
  `axr_task_manager` DRX-storyline task generator on the F97 core [bounty/hostage registries, honcho task
  menus]; the `axr_companions` manager layer [roster + command system writing `npcx_beh_*` → drives
  `axr_beh`, special escort squads, inventory, command wheel] deepening F82; `axr_trade_manager` → F101);
  `axr_main` boot = F65
- Utility & registry foundation → `modlayer/gameplay/utils-db.md` (M22) ✅ + `flows/utils-libraries.md`
  (F115) ✅ (the six `utils_*` helper libraries — the cross-cutting layer ~everything calls into:
  `utils_obj` navigation + cover/movement primitives, `utils_item` pricing + addons, `utils_data` parsing
  + the `cfg_file` write-INI class, `utils_xml` UI scaling, `utils_ui` the inventory-cell engine,
  `utils_stpk` server-packet read/write) + `db` registry = F69 (`flows/db-registry.md`)
- Sound & dynamic music → `flows/sound-music.md` (F116) ✅ (two music systems — engine ambient from
  `game_levels_music.ltx` vs `xrs_dyn_music` Lua combat music that ducks it, combat themes hardcoded +
  only stalker enemies trigger them; the `xr_sound`/`sound_theme` playback + theme registry; the
  `aaa_sound_object_patch` `sound_object` lifetime guard; scheme sounds `sound_manager` campfire stories /
  `sound_ambient` / `ph_sound` / `xr_zones_sound` / `mob_sound`; `level_weathers` weather sibling)
- Treasures & secrets (stashes) → `flows/treasures-secrets.md` (F117) ✅ (the procedural
  `treasure_manager` that fills every `INVBOX` to a kind×tier value budget on new game and lazy-spawns on
  open; rare stashes gate toolkit rewards; predefined authored `secrets.ltx`; `tasks_stash` quest stashes;
  the `item_radio` RF stash-hint coupling)
- Config / data tier → `modlayer/gameplay/configs-data.md` (M23, the **final** gameplay map) +
  `flows/creatures-actor-configs.md` (F118) ✅ (the `configs/creatures/` LTX: the `monster_base`→`m_*`→
  `spawn_sections` inheritance chain, per-monster stat/AI/immunity anatomy, `actor.ltx`, the shared
  `immunities`/`damages` profiles; `game_relations` → F40/F108) + `flows/zones-anomaly-configs.md`
  (F119) ✅ (the `configs/zones/` LTX: `zone_base` inheritance root w/ the `script_binding` artefact-binder
  seam, `class = ZS_*` → CustomZone mapping, blowout/hit/particle param groups, field-vs-mine families;
  the `sr_teleport_sections.ltx` level-changer table → F92) + `flows/items-weapons-outfits-configs.md`
  (F120) ✅ (the `configs/items/` LTX: the `w_*`/`o_*`/`items_*` glob auto-include modding contract, weapon
  root+HUD+variant subsections, outfit protection/immunities, ammo `k_*` coeffs, upgrade trees; trade/ →
  F101, settings/ → F117/F102/F123, base parents → `defines.ltx`) + `flows/per-smart-logic-configs.md`
  (F121) ✅ (the `configs/scripts/` custom_data / logic-LTX tier: object logic FSMs + gulag `[logic@job]`
  catalogs [`prior`/`suitable`] + `smart/` respawn defs; loaded by name not `#include`; `camp.ltx` = the
  campfire storyteller registry F116) + `flows/plugins-tweaks-presets-configs.md` (F122) ✅ (the constants
  tier: `defines.ltx` inheritance root [home of `monster_base`/`default_weapon_params`/`hud_base`/`booster`]
  + `presets/` video/difficulty/warfare bundles + `plugins/`/`ai_tweaks/` per-system tuning routed to
  owners; `axr_options.ltx` empty) + `flows/spawns-alife-graphs-configs.md` (F123) ✅ (the world-instance +
  ALife tier: `spawns/all.spawn` binary placements + `alife.ltx` simulator tuning + `game_graphs`/
  `game_story_ids`/`game_maps_single` registries; templates F118–F122 × placements × logic F121 = the
  world). LTX/DLTX mechanism = `modlayer/config-ltx.md`
- **Status: ✅ GAMEPLAY CORPUS COMPLETE — phases G1–G9 all done (F65–F123 + M1/M5–M23, all sub-maps exist).
  Combined with engine (F1–F45) + renderer (F46–F64), the KB covers the whole game. Maintenance mode: fix/
  deepen on contact per `CONVENTIONS.md`.**

## Structure (verified by listing)

- `unpacked/scripts/` — ~456 `.script`/`.lua`. Prefix families: `xr_*` (schemes/engine), `state_*`
  (animation FSM), `smart_*` (terrain + covers), `se_*` (Lua server entities), `sim_*` (squad sim),
  `bind_*` (binders), `sr_*` (restrictor schemes), `ph_*` (physics-object schemes), `mob_*` (monster
  schemes), `heli_*`, `task*`/`tasks_*`, `dialogs*`, `item_*`, `actor_*`, `ui_*`, `pda*`, `axr_*`,
  `warfare*`, `gamemode_*`/`game_*`, `utils_*`, `db`/`_g`/`class_registrator`.
- `unpacked/configs/` — `gameplay/` (87: character_desc, dialogs, info, encyclopedia), `scripts/` (71:
  per-smart logic LTX), `ui/` (162 xml), `creatures/`, `zones/`, `items/`, `ai_tweaks/`, `plugins/`,
  `presets/`, `misc/`.

## Maintenance

- This map is the gameplay routing hub; `ROADMAP-gameplay.md` is the authoritative plan/status. Keep the
  "subsystem deep-dives" pointers in sync as flows land and sub-maps split out.
- When a domain accumulates several flows, scaffold its M-map (M2–M23) and move that domain's routing
  there, leaving a one-line pointer here.
- Re-list `unpacked/scripts/` prefixes if the corpus changes; always honor the `gamedata/` override rule.
