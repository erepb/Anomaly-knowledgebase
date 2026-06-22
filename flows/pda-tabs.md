---
domain: flows/pda-tabs
tags: [modlayer, gameplay, unpacked, lua, ui, pda, tabs, ranking, radio, flow]
sources: [unpacked/scripts/pda.script, unpacked/scripts/pda_actor.script, unpacked/scripts/pda_flagger.script, unpacked/scripts/ui_pda_contacts_tab.script, unpacked/scripts/ui_pda_npc_tab.script, unpacked/scripts/ui_pda_radio_tab.script, unpacked/scripts/ui_pda_relations_tab.script, unpacked/scripts/ui_pda_warfare_tab.script, src/xrGame/ui/UIPdaWnd.cpp, src/xrGame/ui/UIRankingWnd.cpp]
related: [flows/pda-map-news.md, flows/gameplay-ui-screens.md, flows/actor-menu-inventory-ui.md, flows/game-services.md, flows/faction-relations-lua.md, flows/warfare-metagame.md, flows/characters-info-encyclopedia.md, flows/companion-system.md, modlayer/gameplay/ui-scripts.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: PDA & its tabs (F110)

The PDA screen's Lua layer: a hub (`pda.script`) the engine calls into, the per-tab UI modules,
and two warfare map-menu plugins. The engine `CUIPdaWnd` (F19) owns the window chrome and the
fixed tabs; Lua supplies the *content* of the script-driven tabs. Spine (`pda.script` +
`pda_actor` + `pda_flagger`) READ-IN-FULL and the engine bridge grepped; the five tab modules
agent-traced with their spines spot-verified. No `gamedata/` override of any file.

## pda.script — the engine↔Lua hub (read in full)

The engine (`UIPdaWnd.cpp`) calls **`set_active_subdialog(section)`** when a tab is selected; each
script tab returns a `CUIScriptWnd` from its `get_ui()` singleton. The section map:

- `eptNPC` → `ui_pda_npc_tab` (the dead-NPC PDA reader; needs a PDA device in hand)
- `eptRelations` → `ui_pda_relations_tab`
- `eptContacts` → `ui_pda_contacts_tab` **or `ui_pda_warfare_tab` when `_G.WARFARE`** (warfare
  mode replaces the contacts tab with the war board — verified in `pda.script`)
- `eptRadio` → `ui_pda_radio_tab`
- `eptEncyclopedia` → `ui_pda_encyclopedia_tab` (the F100 module,
  `flows/characters-info-encyclopedia.md`)
- `eptTasks` / `eptRanking` / `eptLogs` are engine-native (no script subdialog)

The hub also carries several engine-called surfaces:

- **PDA-map context menu** — `property_box_add_properties` / `property_box_clicked` fan out to the
  `map_spot_menu_add_property` / `map_spot_menu_property_clicked` bus (F39), the same seam the
  travel features (F107) and the warfare plugins below hook.
- **Stats panel** — `get_stat(index)` returns playtime + `game_statistics` counters + achievement
  and visited-level fractions (F107) for the engine info panel.
- **Top-100 ranking board** — `calculate_rankings` (called from `info_portions`) builds a
  rank-sorted list of all non-zombied/non-trader/non-story stalker server objects; the engine
  `UIRankingWnd`/`UIRankingsCoC` then call `get_rankings_array_size` + `coc_rankings_can_show` /
  `_set_name` / `_set_hint` / `_set_description` / `_set_icon` / `_show_border` to render it (the
  hint pulls last-seen level/time and `m_data` kill/loot tallies from F107). This is the same
  ranking window that polls the achievement functors (F107).
- **Map spots** — `fill_primary_objects` adds anomaly/"primary object" and sleep-zone map spots
  from a large static table when the matching info portion is held; `discover_spots` (a non-warfare
  `ScanForSpots` TimeEvent) reveals them by proximity, granting rank and a fade message.
- **Contacts capture** — `actor_menu_mode(10/11)` (talk dialog show/hide) records the talked-to NPC
  into `m_data.actor_contacts` (the contacts-tab data source) and fires `actor_on_leave_dialog`.
- **Quick slots** — `add_quick_slot_items_on_game_start` seeds `slot_0..3` from the actor's
  `quick_item_N` config (F104).

## pda_actor / pda_flagger — warfare map-menu plugins (read in full)

Despite the `pda_actor` name, both are **warfare-only** PDA-map context-menu plugins on the F39
seam, registered only when `enable_warfare_mode`:

- **pda_actor** — right-clicking the actor's own map spot shows the commander controls: influence
  points, take/release faction control (`manual_control`), and toggle smart-terrain visibility
  (`smart_terrain_warfare.hide_smarts`).
- **pda_flagger** — right-clicking an owned squad or smart flags it as the
  `warfare_selected_target` map spot (`get_selected_object`/`is_object_selected`/`clear`), the
  commander's "direct this unit" selection.

Both delegate the actual war mechanics to the warfare modules (`flows/warfare-metagame.md`, F105).

## The tabs (agent-traced, spines spot-verified)

- **Contacts** (`ui_pda_contacts_tab`, `pda_contacts.xml`) — a grid of nearby online stalkers
  within the `device_pda` `radius` (default 68, squared for the distance test), each row colored by
  faction relation (`xr_conditions.is_factions_friends/enemies`) with rank and reputation; two
  checkboxes (auto-refresh, show-known) persist in `axr_main.config`'s `pda_contacts` section. Pulls
  from `db.OnlineStalkers`/`db.storage` plus the recorded `actor_contacts`, with Sidorovich/Forester
  forced in via story ids.
- **NPC PDA** (`ui_pda_npc_tab`, `pda_npc.xml`) — the reader for a *looted dead NPC's* PDA.
  `register_pda` is called from `death_manager` on NPC death to procedurally build and `se_save_var`
  the PDA's contents: faction-flavored messages from `faction_expansions.faction[comm]`, stash hints
  (`treasure_manager`), map-route rewards, and RF-frequency sources (`item_radio.RF_stashes`).
  Messages may be "encrypted" per a `faction_encyption` table, and there is a verified **1% "kill
  the Strelok" easter-egg PDA** gated by the `kill_the_strelok_pda_found` info (`itm_pda_rare`
  always generates encrypted content).
- **Radio** (`ui_pda_radio_tab`, `pda_radio.xml`) — an in-world radio + music player, not comms.
  Two **radio channels** (with interstitial "commercial sessions") plus two music-player
  **playlists**, all enumerated from `$game_sounds$\radio\...` at load via `getFS():file_list_open`;
  hotkeys come from `plugins\radio_zone_fm.ltx`. `actor_on_update` runs playback, drains the PDA
  battery (gated by `item_device.is_pda_charged`), and modulates volume with interference noise
  during emissions/psi-storms and on underground levels. Channel count is **hardcoded `num_of_ch =
  2`** (adding channels needs code edits, not config). This is the Zone-FM front end over
  `item_radio` (F102).
- **Relations** (`ui_pda_relations_tab`, `pda_relations.xml`) — two panels. A faction×faction
  goodwill **matrix** from `relation_registry.community_relation`, with the actor row from
  `community_goodwill(faction, AC_ID)` and `game_relations.FRIENDS/ENEMIES` thresholds for coloring
  (a button toggles text labels vs raw numbers) — the read surface of F108/F40. And a **companion
  roster** from `axr_companions.list_actor_squad_by_id`, showing each companion's location/health
  and behavior flags read from the `npcx_beh_*` info portions (F82/F114).
- **Warfare** (`ui_pda_warfare_tab`, `pda_warfare.xml`, warfare-only) — a faction-dominance board
  ranking factions by captured-smart count from `warfare.registered_squads` +
  `SIMBOARD.smarts_by_names` (`owning_faction`), with squad counts and a dominance % progress bar
  (friend/enemy/neutral styled via `relation_registry`). An all-level / current-level toggle
  (`btn_map`). It **sets the `warfare_actor_faction_domination` info once the actor's faction holds
  ≥50% of smarts** — which is exactly the gate for F107's `iron_curtain` achievement.

## Quirks (verified / agent-flagged)

- The contacts tab is *replaced* by the warfare tab under `_G.WARFARE` (verified in `pda.script`).
- `is_widescreen` is hardcoded `true` in the contacts and relations tabs (the
  `utils_xml.is_widescreen()` call is commented out) — the non-widescreen branch is dead.
- Radio channel count is hardcoded (2); the warfare dominance sort has no tie-break key
  (faction order is undefined on equal smart counts). *(both agent-flagged)*
- `ui_pda_encyclopedia_tab.script` is a single-line file (`wc -l` reports 0 — a false-empty), not a
  stub; it is the populated F100 encyclopedia module.

## Deeper

- PDA map-location / news data layer (engine tier) → `flows/pda-map-news.md` (F39).
- The ranking board's achievement poll + stats source → `flows/game-services.md` (F107).
- Faction matrix data tier → `flows/faction-relations-lua.md` (F108); warfare board mechanics →
  `flows/warfare-metagame.md` (F105); encyclopedia tab → `flows/characters-info-encyclopedia.md`
  (F100); companion roster → `flows/companion-system.md` (F82).
- The UI-scripts family map → `modlayer/gameplay/ui-scripts.md` (M20).
