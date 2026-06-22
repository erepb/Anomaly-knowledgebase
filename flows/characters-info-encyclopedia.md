---
domain: flows/characters-info-encyclopedia
tags: [modlayer, gameplay, unpacked, lua, character, info-portion, encyclopedia, pda, flow]
sources: [unpacked/configs/gameplay/npc_profile.xml, unpacked/configs/gameplay/character_desc_general.xml, unpacked/configs/gameplay/info_portions.xml, unpacked/configs/system.ltx, unpacked/scripts/info_portions.script, unpacked/scripts/ui_pda_encyclopedia_tab.script, unpacked/configs/plugins/encyclopedia.ltx, unpacked/configs/plugins/encyclopedia_interactivity.ltx, src/xrServerEntities/character_info.cpp, src/xrServerEntities/xrServer_Objects_ALife_Monsters.cpp]
related: [flows/relations-info.md, flows/dialog-system-lua.md, flows/dialog-trade.md, flows/pda-map-news.md, flows/lua-callback-lifecycle.md, flows/se-stalker.md, modlayer/gameplay/dialogs-info.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: characters, info portions & encyclopedia (F100)

Three sibling identity/knowledge systems: **character profiles** (how a spawned NPC gets its
name/icon/community/rank/dialogs from the `character_desc_*` data), **info portions** (the
registration + Lua side of the F40 flag store), and the **encyclopedia** (Anomaly's Lua PDA-article
plugin — *not* the dormant vanilla XML article system).

**Provenance: verified, no agent** — `info_portions.script` READ-IN-FULL;
`ui_pda_encyclopedia_tab.script` spine read (state tables, unlock/is_unlocked, `parse_categories`,
`detect_intereaction`/`create_interaction`, all callback bodies, save/load, `get_ui`; the CUIWindow
tab classes SKIMMED); engine selection read (`CSE_ALifeTraderAbstract::specific_character`,
`CCharacterInfo::Init`/accessors); data shapes sampled (`system.ltx [profiles]`/`[info_portions]`,
`npc_profile.xml`, `character_desc_general*.xml`, `info_portions.xml`, both encyclopedia LTX). This
session also **corrected F40's** "no on-info callback" claim from this trace (see
`flows/relations-info.md` — the actor *does* fire `eInventoryInfo`). **No `gamedata/` overrides**
of these scripts (verified by listing).

## Character profiles — data → engine selection (verified)

- **Registration:** `system.ltx [profiles]`: `files = npc_profile, npc_profile_mlr,
  npc_profile_lostzone` (profile index) and `specific_characters_files = character_desc_general,
  character_desc_<level>…` (the catalog). The per-faction packs
  (`character_desc_general_stalker` etc.) are **not** listed — they are `#include`d by
  `character_desc_general.xml`.
- **Profile (`npc_profile.xml`):** `<character id="…">` with `<class>` (a template class tag)
  and/or `<specific_character>` (a direct pick). The NPC's spawn section / server entity carries a
  `character_profile` naming one of these ids.
- **Specific character (`character_desc_*.xml`):** `<specific_character id team_default>` with
  `<name>` (`GENERATE_NAME_<pool>` = random name), `<icon>`, `<map_icon>`, `<bio>`, `<class>`
  (one or more template tags it answers to), `<community>`, `<terrain_sect>`, `<snd_config>`,
  `<rank>`, `<reputation>`, `<visual>`, `<supplies>` (spawn loadout), then
  `#include character_criticals.xml` + `character_dialogs.xml` — the include that gives every NPC
  `hello_dialog` + the universal dialog list (F99 tier 1).
- **Selection (`CSE_ALifeTraderAbstract::specific_character`,
  `xrServer_Objects_ALife_Monsters.cpp`, read):** cached value wins; else load the profile —
  if it names a `<specific_character>` directly, use it; **else template pick**: scan every
  specific character, skip `no_random` ones, require the profile `<class>` to appear in the
  candidate's class list (or the profile to have no class), rank within `RANK_DELTA` and
  reputation within `REPUTATION_DELTA` of the profile's values, and **exclude ids already used**
  (`CSpecificCharacterRegistry` — an identity is handed out once per ALife world); random-pick
  from the survivors, falling back to the `default_for_community`-flagged set (`VERIFY3` if a
  class has no default at all).
- **Consumption (`CCharacterInfo::Init`, `character_info.cpp`, read):** copies
  community/rank/reputation from the server entity, loads the profile + specific character;
  `Name`/`Bio`/`IconName`/`StartDialog`/`ActorDialogs` feed the talk window (F18/F99), trade,
  PDA contacts (F39).

## Info portions — registration + Lua side (verified)

- **Definitions:** `system.ltx [info_portions] files = info_portions, info_upgrades,
  info_alife_switch, info_yantar, …` → `configs/gameplay/info_*.xml`, each
  `<game_information_portions>` holding `<info_portion id="…"/>` entries — in this corpus they
  are **pure flag declarations** (empty elements; the vanilla child actions are unused). Counts:
  59 in `info_portions.xml`, hundreds per story pack (462 in `info_zaton.xml`).
- **Store + callback:** engine-side `KNOWN_INFO_VECTOR` per `CInventoryOwner` —
  `flows/relations-info.md` (F40), including the **refined fact established this session**: NPCs
  get no callback (poll `has_info`), the **actor** fires `eInventoryInfo` on *new* infos →
  `bind_stalker` routes it to the `actor_on_info_callback` bus event.
- **`info_portions.script`** (READ-IN-FULL): consumes that bus event for (a) an
  **info-switch table** — giving e.g. `ui_pda` runs a condlist that disables `ui_pda_hide` (UI
  open/close pairs, arena chains) via `xr_logic.pick_section_from_condlist`; (b) `ui_pda` →
  `pda.calculate_rankings()`; (c) `proceed()` — polls the nine main-storyline `*_task_done`
  infos and, complete, grants `warlab_password_info` + gives the `sar_warlab`/
  `sar_monolith_destroy` tasks (F97). `on_game_load` resets UI/sleep infos and retro-fixes
  companion `npcx_*` infos on the actor.

## Encyclopedia — the Lua PDA-article plugin (verified)

The vanilla XML article system is **dormant**: `encyclopedia_infos.xml` ships empty. The live
system is `ui_pda_encyclopedia_tab.script` (note: CR-only line endings) + two plugin LTX files.

- **Data:** `plugins/encyclopedia.ltx` — `[encyclopedia]` lists category sections
  (notes/achievements/anomalies/artefacts/characters/factions/features/historic/items/locations/
  mutants/options); each category section lists article ids with optional flags:
  `unlocked=1` (pre-unlocked), `tier=n` (memory-stick random-unlock weighting), mode gates
  (`ironman=0|1`, `story=`, `debug=`, …), `special=1`. `plugins/encyclopedia_interactivity.ltx` —
  `[encyclopedia_interactivity]` maps interaction keys (notes/anomalies/artefacts/characters/
  factions/items/levels/mutants/smarts/vehicles) to sections of `trigger-name = article(s)` pairs.
- **Boot:** `on_game_load` → `parse_categories()` — reads both LTX into `categories[]` +
  `interactivity{}`; `evaluate_article` drops articles whose mode flags don't match the current
  game mode (evaluated once per load).
- **Unlock paths:** `unlock_article`/`unlock_note(section)` move ids from `locked_*` to
  `unlocked_*` and bump `game_statistics`. Triggers: the **`actor_on_interaction`** bus event
  (`detect_intereaction(type, obj, name)`) fired across the corpus (`bind_monster` mutant
  encounters, anomaly binders' pulse, dialogs, `xr_effects.unlock_note_*` story effects); plus
  three callbacks **registered only ~7.5 s after `actor_on_first_update`** (deliberate, to avoid
  off-screen unlocks): `actor_on_leave_dialog` (characters), `actor_on_item_take` (items +
  artefact special-case + stalker→faction), `actor_on_feeling_anomaly` (gated on 2 in-game hours
  in the Zone, then matched against a hard-coded anomaly section list, cached per section). At
  that same moment it sweeps the actor's faction, current level and inventory for retroactive
  unlocks. Artefact detection skips `kind = i_attach`/`i_mutant_belt`.
- **Persistence:** `save_state`/`load_state` marshal `unlocked_articles`/`unlocked_notes` under
  `m_data.game_statistics.*` (F15 bus). Locked sets are *recomputed* from the LTX each load.
- **Consumers:** `pda.script` builds the tab (`get_ui` singleton `pda_encyclopedia_tab`, a
  CUIWindow tree *(SKIMMED)*) and shows unlock counts; `item_recipe`/`item_cooking` gate cooking
  recipes on `is_unlocked_note("encyclopedia__notes_<recipe>")`; a PDA-open tick clears the
  "new article" HUD notification.

## Isolation ledger (the seams)

- **Dialog (F99/F18):** the specific character's `start_dialog`/`actor_dialogs` are what the talk
  window lists; dialogs/condlists are the main info-portion writers.
- **Info store (F40):** definitions registered here; storage/attitude live engine-side;
  `actor_on_info_callback` is the actor-only push channel (F1 bus, via `bind_stalker`).
- **Callback bus (F1):** `actor_on_interaction`, `actor_on_item_take`, `actor_on_leave_dialog`,
  `actor_on_feeling_anomaly`, `save_state`/`load_state`, `on_game_load`.
- **Tasks (F97):** `info_portions.proceed` gives the warlab endgame tasks; task condlists poll
  infos.
- **PDA (F39):** encyclopedia is a PDA tab; `ui_pda`/`ui_pda_hide` info pair tracks PDA state.
- **Server entities (F83):** `character_profile`/`specific_character` live on
  `CSE_ALifeTraderAbstract`; the identity pick happens server-side at first query.
- **Stats/modes:** `game_statistics` counters; mode predicates (`IsHardcoreMode`, `IsStoryMode`,
  `_G.WARFARE`, …) gate article availability at parse time.

## Threading

Main thread; everything is bus-callback or UI driven. The only timer is the 7.5 s interaction
enable delay.

## Failure & edge cases

- **Vanilla encyclopedia XML is dead** — `encyclopedia_infos.xml` is empty; adding `<article>`
  entries there does nothing for the PDA tab. Extend `plugins/encyclopedia.ltx` (+
  `textures_descr/ui_pda_encyclopedia.xml`, per the LTX header comment) instead.
- **First-7.5-seconds blind spot:** interactions before the delay elapses don't unlock articles;
  the retroactive faction/level/inventory sweep covers most of it, but a dialog left in that
  window is missed.
- **Duplicate info gives are silent:** `eInventoryInfo` fires only when the info is new — an
  `actor_on_info_callback` listener never sees re-gives; `OnDisableInfo` never fires at all (F40).
- **Identity exhaustion:** the template pick excludes already-used specific-character ids; a class
  with all candidates consumed falls back to `default_for_community`, and a class with **no**
  default trips `VERIFY3`.
- **Mode flags are parse-time:** article availability is decided in `parse_categories` per load —
  not re-evaluated mid-session.
- **Anomaly articles need 2 in-game hours** in the Zone before `actor_on_feeling_anomaly` unlocks
  anything (also stamps the `actor_spent_2_hrs_in_zone` info).

## Deeper

- Selection: `src/xrServerEntities/xrServer_Objects_ALife_Monsters.cpp`
  (`CSE_ALifeTraderAbstract::specific_character`), `src/xrServerEntities/character_info.cpp` +
  `specific_character.cpp` (XML parse)
- Data: `unpacked/configs/gameplay/npc_profile*.xml`, `character_desc_*.xml`,
  `info_*.xml`, `unpacked/configs/system.ltx` (`[profiles]`, `[info_portions]`)
- Lua: `unpacked/scripts/info_portions.script`, `unpacked/scripts/ui_pda_encyclopedia_tab.script`;
  plugins `unpacked/configs/plugins/encyclopedia{,_interactivity}.ltx`
- Store/attitude: `flows/relations-info.md` (F40); dialogs: `flows/dialog-system-lua.md` (F99);
  map: `modlayer/gameplay/dialogs-info.md` (M14)

## Maintenance

- The PDA tab UI classes (`pda_encyclopedia_tab` controls/callbacks) are SKIMMED — promote if a
  PDA-UI task needs them.
- Re-check `gamedata/scripts/{info_portions,ui_pda_encyclopedia_tab}` per session (none at
  writing).
- If F40's info-callback wording changes again, keep this file's summary in sync (single source:
  `flows/relations-info.md`).
