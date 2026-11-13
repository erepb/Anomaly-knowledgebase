---
domain: features/spawn-overlays
tags: [engine, alife, spawns, graph, ltx, dltx, level-pack, save-load, cpp]
sources: [src/xrGame/spawn_overlays.cpp, src/xrGame/spawn_overlays.h, src/xrGame/spawn_packs.cpp, src/xrGame/spawn_packs.h, src/xrGame/spawn_templates.cpp, src/xrGame/spawn_templates.h, src/xrGame/game_graph_builder.cpp, src/xrGame/game_graph_builder.h, src/xrGame/alife_spawn_registry.cpp, src/xrGame/patrol_path_storage.cpp, src/xrGame/patrol_point.cpp, src/xrGame/level_graph.cpp, src/xrEngine/xrLevel.h, src/xrGame/game_graph_script.cpp, src/xrGame/alife_simulator_script.cpp, gamedata/configs/spawn_overlays.ltx, gamedata/configs/mod_spawn_overlays_meadow.ltx]
related: [flows/spawns-alife-graphs-configs.md, flows/alife-online-offline.md, flows/alife-services.md, flows/save-load.md, modlayer/config-ltx.md, GLOSSARY.md#spawn-overlay]
updated: 2026-11-13
---

# Spawn overlays & level packs

Modular `all.spawn` editing at load time. Before this, `all.spawn` was a single binary that only the SDK /
ACDC could change and only one mod could own. Now the engine assembles the spawn on every
`CALifeSpawnRegistry::load` (new game **and** every save load) from the shipped base plus text overlays,
SDK fragments and whole extra spawn files ("level packs"). Nothing is written back to disk. The
**authoritative user-facing manual** is the comment header of `gamedata/configs/spawn_overlays.ltx`
("What to use for what", rules of thumb, application order, per-section syntax). Read it before
changing behaviour; this file is the code map.

## Where it hooks

`alife_spawn_registry.cpp` (`CALifeSpawnRegistry::load`) builds one `spawn_overlays::CSpawnOverlays`
per load, between the chunks of `all.spawn`:

1. chunk 4 (game graph) → `CSpawnOverlays::apply_graph` returns a re-serialized graph (or nullptr when
   unchanged) that replaces `m_game_graph` *before* `ai().game_graph()` is set.
2. chunk 1 (templates) → `apply_objects`.
3. chunk 3 (patrol paths) → `ai().patrol_path_storage()` then `add_paths`.
4. `spawn_guid` folds every contributing pack into the header GUID; the save-compat assert compares
   against that combined GUID.

## Pieces

- **`CSpawnOverlays`** (`spawn_overlays.cpp`) — parses `spawn_overlays.ltx` + its DLTX mods
  (`mod_spawn_overlays_*.ltx`), owns a `CSpawnPacks`. Sections: `[settings]` (`link_tolerance`,
  `sync_level_offsets`), `[graph_links]`/`[graph_unlinks]` (+ `@suffix` variants), `[spawn_replace]`,
  `[spawn_remove]`, `[spawn_patch@<name>]`, `[path_remove]`, `[level_cut]`. Endpoints are written
  `level:x,y,z` and resolved to the nearest game vertex within `link_tolerance`. **Ids are never keys** —
  objects by name (+ position when shared), levels by name.
- **`CSpawnPacks`** (`spawn_packs.cpp`) — a pack is any `$game_spawn$\<name>.spawn` other than the
  base. Only what the base lacks is taken: levels absent from the base header (`append_levels`, game id
  from `game.ltx [levelNN]` or `id = auto`), records whose name+level+position (1 m,
  `SAME_OBJECT_TOLERANCE`) are new (`add_objects`), paths whose names are new (`add_paths`). A base level
  whose installed `level.ai` GUID matches the pack's build but not the base's is **substituted**
  (rebuilt from the pack). `report_drift` logs "renamed? / moved?" hints for pack records that look like
  a base template the pack's build changed.
- **`STemplates` / helpers** (`spawn_templates.cpp`) — name index + taken story/spawn-story/spawn ids,
  built once before edits so removed ids are never reused. `prepare_template` refuses creatures,
  non-ALife and nested objects; `relocate` places by nearest graph vertex (exact node snapped later by
  `CSE_ALifeDynamicObject::synchronize_location`); `resolve_destination` re-resolves a level changer's
  destination ids from level name + point.
- **`CGameGraphBuilder`** (`game_graph_builder.cpp`) — editable copy of the game graph (vertices, edges,
  level points, per-level cross tables) that re-serializes into the spawn-chunk layout; used by
  `apply_graph` for appends, links/unlinks, offsets and cuts.
- **Fragments** — `spawns\<level>\*.spawn` (SDK `level.spawn` format) add templates;
  `spawns\<level>\*.game` (SDK `level.game`) add patrol paths. `patrol_paths.ltx` sections with a
  `level = <name>` key replace a compiled path of the same name (`patrol_path_storage.cpp`,
  `patrol_point.cpp`).
- **`[level_cut]`** — removes a level from play without renumbering: every edge to/from it, its
  templates, the level changers leading into it, its paths.

## Save compatibility

Graph edits, offsets and path additions apply to existing saves. Object adds/removes/patches are
new-game content (a save already holds its objects). Adding/removing a level changes the spawn GUID →
older saves refuse. **Legacy translation:** a save made with a pack's own `all.spawn` as base is
detected (`legacy_save`), and `CALifeSpawnRegistry::remap_legacy` rewrites saved objects' graph/spawn
ids through the pack's id tables (`take_legacy`).

## Large AI maps (v13)

`level.ai` version 13 (`XRAI_LARGE_VERSION`, 26-bit links, 32-bit packed XZ; `NodeCompressed13` in
`xrLevel.h`) loads alongside Anomaly's v10. `CLevelGraph` ctor converts v10 nodes into the v13 in-memory
layout (`m_nodes_allocated`), so the rest of the engine sees one format. Cover quadtree storage grows
for large maps (`quadtree.h`). Packs check the installed `level.ai` GUID/version
(`CSpawnPacks::installed_level`).

## Lua exports (added alongside)

`game_graph():nearest_vertex(level_name, pos)`, `game_graph():level_neighbours(level_id)`
(`game_graph_script.cpp`), `alife():create_on_level(section, level_name, pos[, lvid])`
(`alife_simulator_script.cpp`) — spawn by level name/position without hardcoded vertex ids.
Signatures in `lua_help_ex.script`.

## Shipped content

`mod_spawn_overlays_meadow.ltx` — links Meadow to Cordon / Dark Valley via `[graph_links]`.
`dynamic_items_by_level.script` resolves dynamic item places by level instead of recorded game vertex
ids (so they survive graph renumbering).

## Deeper

- Manual: `gamedata/configs/spawn_overlays.ltx` header
- Load order: `alife_spawn_registry.cpp:CALifeSpawnRegistry::load`, `CALifeSpawnRegistry::remap_legacy`
- Packs: `spawn_packs.h:CSpawnPacks::SPack` (per-pack counters logged by `finish`)
- Graph assembly: `game_graph_builder.h:CGameGraphBuilder`
- Log grep: `[spawn_overlays]` (`*` applied, `!` skipped, `-` level not installed)

## Maintenance

- `git grep -n "spawn_overlays::" src/xrGame` should show only `alife_spawn_registry.cpp` outside the
  `spawn_*` files.
- If `spawn_overlays.ltx` gains/loses a section, update the section list above.
- Confirm `XRAI_LARGE_VERSION` still exists in `xrLevel.h`.
