---
domain: flows/spawns-alife-graphs-configs
tags: [modlayer, gameplay, unpacked, config, ltx, data, spawns, alife, graph, story-id, flow]
sources: [unpacked/spawns/all.spawn, unpacked/configs/alife.ltx, unpacked/configs/game_graphs.ltx, unpacked/configs/game_story_ids.ltx, unpacked/configs/game_spawn_story_ids.ltx, unpacked/configs/game_maps_single.ltx, unpacked/configs/patrol_paths.ltx, unpacked/configs/dynamic_spawn.ltx, gamedata/configs/patrol_paths.ltx, src/xrGame/patrol_point.cpp]
related: [features/spawn-overlays.md, flows/alife-online-offline.md, flows/alife-services.md, flows/sim-board.md, flows/scripted-squad.md, flows/smart-terrain-core.md, flows/pda-map-news.md, flows/gameplay-boot.md, flows/per-smart-logic-configs.md, flows/creatures-actor-configs.md, modlayer/gameplay/configs-data.md, modlayer/config-ltx.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: spawns / ALife / graphs configs (F123)

> **Since 2026.10.11 `all.spawn` is no longer edit-only-via-SDK.** The engine applies text overlays,
> SDK fragments and whole level packs on top of it at every spawn-registry load —
> `gamedata/configs/spawn_overlays.ltx` (DLTX-moddable) and `spawns\<level>\*.spawn|*.game`. See
> `features/spawn-overlays.md`. Statements below about `all.spawn` being the only placement source
> describe the shipped base file.

The **world-instance + ALife tier** — the data that places every authored object in the world and tunes
the ALife simulator: the binary `spawns/all.spawn`, the `[alife]` tuning block, the game-graph
location/level tables, the story-ID registries, and the level/map registry. Where F118–F122 provide the
*templates* (creature/item/zone sections, logic, constants), this tier provides the *placements and the
simulation* that bring them to life. This is the **final gameplay-corpus flow** (F123); completing it
closes M1–M23 + F65–F123. Runtime owners: F16 (ALife online/offline), F41 (ALife services), F86 (sim_board),
F39 (PDA map).

**Provenance:** **verified, no agent** — `alife.ltx`, `game_graphs.ltx`, `game_story_ids.ltx`,
`game_spawn_story_ids.ltx`, `game_maps_single.ltx`, `patrol_paths.ltx`, `dynamic_spawn.ltx` read; the binary
`spawns/all.spawn` located (its contents are not text — characterized structurally, not byte-read). **No
`gamedata/` override** of any of these at writing (base `all.spawn` in `unpacked/spawns/`; no
`gamedata/spawns/` or config overrides found).

## `spawns/all.spawn` — the authored world (verified location; binary)

`unpacked/spawns/all.spawn` is the **binary world-spawn file**: it holds the `cse_alife_*` server record of
**every static world object** plus the level/graph cross-references — every smart_terrain, space restrictor,
anomaly zone, INVBOX stash (F117), story NPC, level-changer, and **patrol `way_` path**. It is built offline
by the SDK level-compiler / `xrAI` from the levels' spawn elements + the game graph; the engine loads it at
**new game** to seed the ALife simulator (F16/F41). It is **not** text LTX and cannot be edited as text —
modders use the SDK or an ACDC-style decompiler/rebuilder. Once a game is saved, ALife state diverges from
`all.spawn` (the save is authoritative thereafter). This is the **instance** layer: the placements whose
*templates* live in F118–F122 and whose *logic* (custom_data) is F121.

## `alife.ltx` — the simulator tuning (verified, owned here)

The `[alife]` block = the knobs the ALife online/offline machine (F16) reads: `schedule_min`/`schedule_max`/
`process_time`/`objects_per_update`/`update_monster_factor` (the offline update budget), **`time_factor`**
(6 in-game seconds per real second) + `normal_time_factor`, **`switch_distance`** (750 m online-switch
radius) + the `auto_switch_*` performance-distance system (an `xr_patch` optimization), `start_time`/
`start_date` (09:00, 25.10.2018), `autosave_interval`, and **`start_game_callback = _G.start_game_callback`**
— the gameplay-boot hook (F65). Tune ALife pacing/draw-distance here.

## `game_graphs.ltx` — the graph location/level tables (verified)

The text companion to the binary game graph: `[location_0]` is the **master ordered list of ~200 graph-
point names** (every smart_terrain, `*_sim_N` simulation point, and monster lair across all levels, indexed
by graph-point id — zaton/jupiter/pripyat/… then the rest). `[location_1]`/`[location_2]` are access-type
categories (limited/denied access, "not for the player"); `[location_3]` is the per-vertex **level
membership** list. These name the graph points the ALife/sim systems (F86 sim_board, F85 squads, level
transitions) reference by id. `[graph_points_draw_color_palette]` is debug-draw only.

## Story-ID registries (verified)

- `game_story_ids.ltx [story_ids]` — the **numeric story-ID → unique object-name** registry (per-level id
  ranges; ~74 lines). Scripts resolve a specific authored object (quest NPC, door, restrictor) via its
  story id (`story_objects` / `alife_object_by_story_id`) independent of its runtime object id. The corpus's
  way of saying "this exact placed object."
- `game_spawn_story_ids.ltx [spawn_story_ids]` — the parallel spawn-time story-ID table; **effectively
  empty in base** (just the header), populated by content that needs spawn-story refs.

## `game_maps_single.ltx` — level & world-map registry (verified)

`[level_maps_single]` lists every level (jupiter, k00_marsh, l01_escape, … l13_generators) — the PDA
local-map list: the only engine reader is `CUIMapWnd::Init` (`ui/UIMapWnd.cpp`); scripts read it plus the
per-level `[<level>] weathers` key (`level_weathers`, `ui_pda_radio_tab`, debug menus). It is **not** what the
loader uses: the engine's level ids/names come from `game.ltx [levels]` → `[levelNN]` (`game_levels.ltx`) and
the game graph, and `CApplication::Level_Scan` (`xrEngine/x_ray.cpp`) lists level folders on disk. `[def_map]`/`[global_map]` give the PDA world-map layout
(`bound_rect`/`texture`, F39), and `[___level_music_tracks]` holds per-level music windows (→ F116).
`game_maps_mp.ltx` is the MP variant.

## Vestigial stubs (verified)

- **`patrol_paths.ltx` is effectively empty** (1 line; the repo ships an empty
  `gamedata/configs/patrol_paths.ltx` too). Patrol paths in Anomaly are stored as `way_` spawn
  elements **inside `all.spawn`**, not here — so the `path_walk`/`path_look` names the walker/camper schemes
  reference (F74) resolve against the binary spawn, not this LTX. The file is a vestigial stub. (A few
  per-smart `*_patrol.ltx` under `scripts/` are custom_data, F121, not patrol geometry.)
  **But the engine path for LTX-defined patrols is real and now works:** `CPatrolPoint::load_from_config`
  reads `<point>:name` / `:position` / `:level_vertex_id` / `:game_vertex_id` (each `R_ASSERT4`'d present)
  plus optional `:flags` from a patrol section. It was broken until 2026-07 — every key was built with
  `FormatString(...).c_str()` on a **temporary** `xr_string`, so the `LPCSTR` dangled and lookups read
  freed memory; the fix keeps each key in a named `xr_string`. If you find old notes saying LTX patrol
  paths "don't work", that was this bug.
- **`dynamic_spawn.ltx` is near-empty** — the live dynamic-item-spawn config is `items/settings/
  dynamic_item_spawn.ltx` (F120/F106); this is a stub.

## The unifying picture — how the whole config tier composes

This flow closes the loop on Phase G9: **templates (F118 creatures · F120 items/weapons/outfits · F119
zones) × placements (`all.spawn`) × logic (F121 custom_data) × constants (F122 defines/presets)** = the
authored world, which the **ALife simulator** (F16, tuned by `alife.ltx`) runs offline over the **game
graph** (`game_graphs.ltx`), with **smart_terrain/sim_board** (F72/F86) populating it and **story ids**
(`game_story_ids.ltx`) letting scripts address specific objects. The PDA map (F39) draws it via
`game_maps_single.ltx`.

## Isolation ledger (the seams)

- **ALife (F16/F41):** `all.spawn` seeds the simulator at new game; `alife.ltx` tunes its schedule/switch/
  time; the save diverges thereafter.
- **Graph / simulation (F86/F85):** `game_graphs.ltx` names the graph points sim_board/squads reference;
  level transitions cross-reference them.
- **Story addressing:** `game_story_ids.ltx` → `alife_object_by_story_id`/`story_objects` across the quest/
  dialog/scheme corpus.
- **Levels / map (F39):** `game_maps_single.ltx` level registry + world-map layout + per-level music (F116).
- **Boot (F65):** `alife.ltx start_game_callback = _G.start_game_callback`.
- **Templates ↔ placements:** every `cse_alife_*` record in `all.spawn` instantiates a section defined in
  F118–F122 and may carry a `custom_data` logic file (F121).
- **LTX/DLTX mechanism:** the text files parse per `modlayer/config-ltx.md`; `all.spawn` is binary (SDK/
  ACDC, not DLTX).

## Failure & edge cases

- **`all.spawn` is binary and new-game-only** — editing it requires the SDK/ACDC, and changes affect only
  **new games** (saves carry their own ALife state). This is why "I edited the spawn but my save didn't
  change" is expected.
- **`patrol_paths.ltx` is a stub in base content** — compiled paths live in `all.spawn` as `way_`
  elements. Its sections now add paths on top of the spawn; one with a `level = <name>` key also
  replaces a compiled path of the same name, one without is skipped on a name clash
  (`features/spawn-overlays.md`).
- **Story ids are the stable handle** — runtime object ids change between sessions; `game_story_ids.ltx` is
  how scripts reliably find a placed object.
- **`game_graphs.ltx` names ≠ live objects** — it's the graph-point catalog; the actual server objects come
  from `all.spawn`. A mismatch (renamed smart) breaks sim references.
- **DLTX reaches the spawn through overlays** — `all.spawn` itself is still binary, but
  `spawn_overlays.ltx` (and `mod_spawn_overlays_*.ltx`) edit links, templates and paths at load time
  (`features/spawn-overlays.md`).

## Deeper

- Binary world spawn: `unpacked/spawns/all.spawn` (SDK/ACDC-built; not text)
- ALife tuning: `unpacked/configs/alife.ltx`
- Graph/levels/story: `unpacked/configs/game_graphs.ltx`, `game_maps_single.ltx`, `game_story_ids.ltx`,
  `game_spawn_story_ids.ltx`
- Stubs: `unpacked/configs/patrol_paths.ltx`, `dynamic_spawn.ltx`
- Runtime: `flows/alife-online-offline.md` (F16), `flows/alife-services.md` (F41), `flows/sim-board.md`
  (F86), `flows/scripted-squad.md` (F85), `flows/smart-terrain-core.md` (F72), `flows/pda-map-news.md`
  (F39), `flows/gameplay-boot.md` (F65)
- Map: `modlayer/gameplay/configs-data.md` (M23); LTX mechanism: `modlayer/config-ltx.md`

## Maintenance

- Text files read; `all.spawn` characterized structurally (binary — read with the SDK/ACDC when a task
  needs specific spawn contents). If a level's smart/graph names change, re-check `game_graphs.ltx` ↔
  `all.spawn` consistency.
- Re-check `gamedata/spawns/` + `gamedata/configs/{alife,game_*,patrol_paths,dynamic_spawn}.ltx` for
  overrides per session (none at writing).
- Keep the load-bearing notes: **`all.spawn` = binary, new-game-only placement layer**, **`patrol_paths.ltx`/
  `dynamic_spawn.ltx` are stubs** (paths live in `all.spawn`), and **story ids are the stable object
  handle**.
