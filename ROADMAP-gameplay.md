---
domain: meta
tags: [meta, roadmap, gameplay, modlayer, unpacked, coverage]
sources: [unpacked/scripts/, unpacked/configs/, gamedata/scripts/, gamedata/configs/]
related: [ROADMAP.md, INDEX.md, METHODOLOGY.md, CONVENTIONS.md, modlayer/_overview.md, modlayer/scripting.md]
updated: 2026-06-12
---

# KB Coverage Roadmap — Gameplay corpus (`unpacked/`)

The plan for documenting the **gameplay layer**: the ~456 base-game Lua scripts in `unpacked/scripts/`
plus the ~hundreds of `unpacked/configs/` data files that drive them. This is the second half of the
game — the engine (`src/`) provides the machine; this corpus is the **gameplay built on it** (schemes,
jobs, squads, tasks, dialogs, trade, survival, warfare, UI).

This is a **sibling of `ROADMAP.md`** (engine/renderer, F1–F64), not a separate KB: same
`claudedocs/kb/`, same `GLOSSARY.md`/`CONVENTIONS.md`/`METHODOLOGY.md`, same per-flow recipe and DoD.
The new work lives in `modlayer/gameplay/` (L1 maps) + `flows/` (L2/L3), F-numbered **F65+** so it
continues the engine numbering without collision. **Cross-references to engine flows are normal
`related:` links** — the gameplay corpus calls into the engine (CScriptGameObject, callbacks) and the
engine fires back into it.

## How to run a cold session on this file

1. Read `INDEX.md` (load order + audit table) and `GLOSSARY.md`. Read `METHODOLOGY.md` for the
   `unpacked/` (canonical, read-only) vs `gamedata/` (modded delta) dual-cite rule.
2. Open this file; pick the **lowest-numbered ⬜** item (they are ordered by dependency — the spine
   first). If it's a Phase G0 map, scaffold the map; otherwise it's a flow.
3. Execute the **per-flow recipe** (see `ROADMAP.md` → "Per-flow execution protocol"): one read-only
   Explore agent with the tight contract → spot-verify the spine yourself → write the file
   (`verified: code`, honest `(agent-traced)`/`(INFERRED)` tags) → wire into INDEX + this file + the
   owning L1 map → integrity check (0 line refs, 0 code fences, KB gitignored).
4. Flip the item ⬜→✅ here and add its `INDEX.md` rows. Bump `updated`.

**Entry hints below are existence-verified file pointers** (scripts confirmed present in
`unpacked/scripts/` on 2026-06-10), **NOT verified behavioral claims** — verify the spine on pickup.

## Why this is bounded (despite ~456 files)

Same insight as the engine roadmap: file count = breadth, not distinct mechanisms. The corpus is a
**small set of frameworks** (the scheme engine, state_mgr, the gulag/smart_terrain job board, the
condlist library, the binder layer) plus **many leaf schemes/configs that are instances of those
frameworks**. Document each framework's spine once; the leaves then read as "another scheme on the
documented pattern." So full coverage ≈ **~23 L1 maps + ~59 flows (F65–F123)**, not 456 files.

## The `unpacked/` ⇄ `gamedata/` rule (critical)

`unpacked/` is the **read-only** base corpus (never edit it). `gamedata/scripts/` overrides a subset by
same-name shadowing — the `gamedata/` copy wins at runtime. Every gameplay flow: document the
**canonical mechanism** citing `unpacked/`, then note the **modded delta** if a `gamedata/` override
exists (grep `gamedata/scripts/` for the same filename on pickup). See `METHODOLOGY.md`; the known trap
is assuming `unpacked/` is live when a `gamedata/` override exists (e.g. `bind_monster.script`).

## Existing bridge flows to link, not duplicate

These engine-side/bridge flows already touch the corpus — gameplay flows should **cross-link and
deepen**, not re-document: F1 `lua-callback-lifecycle`, F2 `dltx-config-load`, F3
`dxml-xml-interception`, F12 `ai-squad-logic`, F17 `options-runtime`, F18 `dialog-trade`, F20
`animation-callbacks`, F33 `smart-cover`, F36 `special-actors`, F37 `script-npc-actions`, F38 `tasks`,
F39 `pda-map-news`, F40 `relations-info`, F41 `alife-services`, F42 `cscript-game-object`.

---

## State snapshot (2026-06-11)

- **Phase G1 (gameplay spine) ✅ COMPLETE (2026-06-10).** ✅ **M1** hub + ✅ **F65–F73** (gameplay-boot,
  scheme-engine-core, condlist-library, logic-ltx-switching, db-registry, state_mgr core, state_mgr
  controllers, smart_terrain core, gulag job board) — all `verified: code`. The framework spine is done.
- **Phase G2 (NPC schemes) ✅ COMPLETE (2026-06-10).** ✅ **F74–F82** — walker/patrol/remark (the
  canonical template), camper/cover, combat, animpoint, sleeper/wounded/death, meet/abuse/bribe,
  gather/corpse, sensors/misc, companion. The `xr_*` scheme catalog is documented (catalog leaves now read
  as deltas on F74). Phases G3–G9 (F83–F123) ⬜.
- **Phase G3 (squads/server-entities) ✅ COMPLETE (2026-06-10).** ✅ **F83**+**F84** (`se_*` server-entity
  tier, map **M6**) + ✅ **F85** (scripted squad) + ✅ **F86** (sim_board) + ✅ **F87** (offline combat) +
  map **M5**.
- **Phase G4 (binders & object schemes) ✅ COMPLETE (2026-06-10).** ✅ **F88** (binder architecture) +
  ✅ **M7** (binders map) + ✅ **F89** (world-object binders) + ✅ **F90** (anomaly-field binders) + ✅ **F91**
  (passive `sr_*`) + ✅ **F92** (active `sr_*`) + ✅ **M10** (restrictors map) + ✅ **F93** (`ph_*`
  physics-object schemes) + ✅ **M11** (ph map).
- **Phase G5 (monsters & vehicles) ✅ COMPLETE (2026-06-11).** ✅ **F94** (`mob_*` monster schemes) +
  ✅ **M9** (monster-schemes map) + ✅ **F95** (helicopter AI) + ✅ **F96** (cars — **merged into M12**, too
  thin for its own flow: bind_car=F88-clone→ph_car F93, se_car=F84, driving=F29) + ✅ **M12** (vehicles map).
- **Phase G6 (tasks/dialogs/economy) ✅ COMPLETE (2026-06-11).** ✅ **F97** (task system core) + ✅ **M13**
  (tasks-lua map) + ✅ **F98** (dynamic task types) + ✅ **F99** (dialog system) + ✅ **M14** (dialogs-info
  map) + ✅ **F100** (characters/info/encyclopedia — M14 domain complete; also corrected F40's
  info-callback claim) + ✅ **F101** (trade & economy) + ✅ **M15** (trade-economy map) + ✅ **F102**
  (item Lua behaviors) + ✅ **M16** (items-lua map) + ✅ **F103** (actor survival & status) + ✅ **M17**
  (actor-survival map) + ✅ **F104** (actor menu & inventory UI — M17 domain complete) + ✅ **F105**
  (warfare metagame) + ✅ **M18** (warfare map) + ✅ **F106** (game modes & difficulty) + ✅ **M19**
  (gamemodes-services map) + ✅ **F107** (game services — M19 domain complete) + ✅ **F108**
  (faction relations (Lua) — standalone flow, no new map; engine tier is F40).
- **Phase G8 (UI/addon/utils/PDA/sound) in progress (2026-06-21).** ✅ **F109** (gameplay UI screens) +
  ✅ **M20** (ui-scripts map) + ✅ **F110** (PDA & its tabs) + ✅ **F111** (UI debug tools) + ✅ **F112**
  (news & notifications) — **M20 ui_* flow coverage complete** — + ✅ **F113** (AXR behaviour add-ons) +
  ✅ **F114** (AXR managers — task/companion/trade) — **M21 axr-addon domain complete** — + ✅ **F115**
  (utility libraries — `utils_*`) + ✅ **M22** (utils-db map scaffolded) + ✅ **F116** (sound & dynamic
  music) + ✅ **F117** (treasures & secrets). **Phase G8 COMPLETE.**
- **Phase G9 (config / data tier) ✅ COMPLETE (2026-06-21).** ✅ **M23** (configs-data map) + ✅ **F118**
  (creatures & actor) + ✅ **F119** (zones & anomaly params) + ✅ **F120** (items/weapons/outfits) + ✅
  **F121** (per-smart logic) + ✅ **F122** (plugins/tweaks/presets) + ✅ **F123** (spawns/ALife/graphs — the
  final item). All six data-tier flows written; M23 domain complete.
- **✅✅ GAMEPLAY CORPUS COMPLETE (M1–M23 + F65–F123).** Every gameplay flow + sub-map is written and
  verified. Combined with the engine (F1–F45) and renderer (F46–F64) roadmaps, the KB now covers the whole
  game, both halves. No roadmap items remain (OOS F45 MP netcode aside). The KB is in **maintenance mode** —
  fix/deepen files on contact per `CONVENTIONS.md`.
- **Map strategy adjusted:** the M2–M23 sub-maps are scaffolded **just-in-time** as each domain's first
  flow lands (not 23 upfront stubs); the M1 hub routes everything meanwhile. Phase G0 below is the
  catalog of sub-maps, created on demand. **Created so far: M1, M5, M6, M7, M9, M10, M11, M12, M13, M14,
  M15, M16, M17, M18, M19, M20, M21, M22, M23 — all gameplay sub-maps now exist.**
- L0/L1 for the engine + the shared meta files already exist; this adds the **gameplay L1 tier**
  (`modlayer/gameplay/`) and its flows.
- Suggested cold-start order = **ascending F-number** (dependency-ordered: spine → schemes → squads →
  binders → monsters/vehicles → tasks/dialogs/economy → actor/metagame → UI/utils → config/data tier).
  **Next up: nothing — the roadmap is complete.** F123 was the final item; the gameplay corpus
  (M1–M23 + F65–F123) and the whole KB (engine F1–F45 + renderer F46–F64 + gameplay F65–F123) are done.
  Future sessions: maintenance mode — fix/deepen on contact per `CONVENTIONS.md`; add a new F-row here only
  if a genuinely new system is discovered. (Note: `valid_item_sections.ltx` from the original F120 hint does not exist in this
  corpus — dropped.)

Priority: **P1** = highest modding value / framework spine · **P2** = medium · **P3** = niche/optional.

---

## Phase G0 — scaffold the gameplay L1 maps

Create each as a short L1 map (verified by *listing + headers*, not deep reads): module list, what the
domain is, where to start, and a `related:` block that delegates mechanics to the flows below. Mark
not-yet-written deep claims `(INFERRED)`; promote them as flows land. Do these first so flows have a
home to attach to and INDEX can route.

| ID | Map file | Pri | Covers |
|----|----------|-----|--------|
| M1 | `modlayer/gameplay/_overview.md` | P1 | ✅ **DONE** — the corpus hub: boot → scheme architecture → registries → schemes/binders/SE/configs; routes to all gameplay flows |
| M2 | `modlayer/gameplay/scheme-engine.md` | P1 | `xr_logic`, `xr_motivator`, `xr_gulag`, condlists, logic-LTX, scheme registration |
| M3 | `modlayer/gameplay/state-manager.md` | P1 | `state_mgr*`, `state_lib*` animation/movement/mental FSM |
| M4 | `modlayer/gameplay/smart-terrain-jobs.md` | P1 | `smart_terrain`, `gulag_general`, `xr_gulag`, job board |
| M5 | `modlayer/gameplay/squads-simulation.md` | P2 | ✅ **DONE** — `sim_squad_*`, `sim_board`, `simulation_objects`, `sim_offline_combat`; routes F85–F87 |
| M6 | `modlayer/gameplay/server-entities-lua.md` | P2 | ✅ **DONE** — `se_*` Lua server classes, registration, save/load; routes F83/F84 |
| M7 | `modlayer/gameplay/binders.md` | P1 | ✅ **DONE** — `bind_*` object→Lua binders, `class_registrator`; routes F88–F93 |
| M8 | `modlayer/gameplay/npc-schemes.md` | P1 | the `xr_*` stalker behavior-scheme catalog |
| M9 | `modlayer/gameplay/monster-schemes.md` | P2 | ✅ **DONE** — `mob_*` monster schemes; routes F94 |
| M10 | `modlayer/gameplay/restrictors-sr.md` | P2 | ✅ **DONE** — `sr_*` space-restrictor schemes (passive F91 + active F92); routes them |
| M11 | `modlayer/gameplay/physics-schemes-ph.md` | P2 | ✅ **DONE** — `ph_*` physics-object schemes + `phantom_manager`; routes F93 |
| M12 | `modlayer/gameplay/vehicles-heli.md` | P3 | ✅ **DONE** — `heli_*`, car bind/se/ph; routes F95 + holds merged F96 |
| M13 | `modlayer/gameplay/tasks-lua.md` | P1 | ✅ **DONE** (renamed from tasks.md to avoid basename clash with flows/tasks.md) — `task_manager`, `task_*`, `tasks_*` dynamic types; routes F97 (+F98/F114 ⬜) |
| M14 | `modlayer/gameplay/dialogs-info.md` | P1 | ✅ **DONE** — `dialogs*`, `dialog_manager`, character desc/attachment; routes F99 + F100 — domain complete |
| M15 | `modlayer/gameplay/trade-economy.md` | P1 | ✅ **DONE** — actor↔NPC shop, engine stock spawn, NPC↔NPC gulag-job economy; routes F101 (+F102/F114 ⬜) |
| M16 | `modlayer/gameplay/items-lua.md` | P2 | ✅ **DONE** — `item_*` plugins + itms_manager hub + _ITM registry + items/settings data tier; routes F102 |
| M17 | `modlayer/gameplay/actor-survival.md` | P1 | ✅ **DONE** — Lua survival layer (sleep/thirst/psy meters, HUD indicator strip, actor_effects sensory layer) + actor menu/inventory UI; routes F103 + F104 — domain complete |
| M18 | `modlayer/gameplay/warfare.md` | P2 | ✅ **DONE** — faction-war metagame (hub/smart/squad/faction/level/commander/options/UI pieces); routes F105 |
| M19 | `modlayer/gameplay/gamemodes-services.md` | P2 | ✅ **DONE** — gamemode_* + game_difficulties + game_setup pieces; routes F106 (+F107 `game_*` services ⬜) |
| M20 | `modlayer/gameplay/ui-scripts.md` | P2 | ✅ **DONE** — `ui_*` dialogs (CUIScriptWnd) + HUD overlays (custom-static); routes F104 + F109 + F110 + F111 + F112 — ui_* flow coverage complete |
| M21 | `modlayer/gameplay/axr-addon.md` | P2 | ✅ **DONE** (domain complete F114) — `axr_*` addon layer: boot orchestrator (axr_main F65) + behaviour schemes (F113 ✅) + task/companion/trade managers (F114 ✅); routes F113 + F114 |
| M22 | `modlayer/gameplay/utils-db.md` | P1 | ✅ **DONE** (scaffolded F115) — utility & registry foundation: `utils_*` helper libs (F115 ✅) + `db` registry (F69 ✅) + script bootstrap `_g`/`class_registrator`; routes F115 + F69 |
| M23 | `modlayer/gameplay/configs-data.md` | P2 | ✅ **DONE** (scaffolded F118) — the data tier: creatures/actor (F118 ✅) + zones (F119 ⬜) + items/weapons (F120 ⬜) + per-smart logic (F121 ⬜) + plugins/tweaks (F122 ⬜) + spawns/ALife (F123 ⬜); LTX/DLTX mechanism = modlayer/config-ltx.md; routes F118 |

---

## Phase G1 — gameplay spine (do first; everything depends on it)

| ID | Flow | Pri | Key files (`unpacked/scripts/` unless noted) | Engine xref |
|----|------|-----|----------------------------------------------|-------------|
| F65 | Gameplay boot & load order | ✅ P1 | **DONE** → `flows/gameplay-boot.md` |
| F66 | Scheme engine core | ✅ P1 | **DONE** → `flows/scheme-engine-core.md` |
| F67 | Condlist library (conditions/effects) | ✅ P1 | **DONE** → `flows/condlist-library.md` |
| F68 | Logic-LTX format & scheme switching | ✅ P1 | **DONE** → `flows/logic-ltx-switching.md` |
| F69 | db registry & global state | ✅ P1 | **DONE** → `flows/db-registry.md` |
| F70 | state_mgr core | ✅ P1 | **DONE** → `flows/state-manager-core.md` |
| F71 | state_mgr animation/movement layers | ✅ P2 | **DONE** → `flows/state-mgr-controllers.md` |
| F72 | smart_terrain core | ✅ P1 | **DONE** → `flows/smart-terrain-core.md` |
| F73 | Gulag job board | ✅ P1 | **DONE** → `flows/gulag-jobs.md` |

## Phase G2 — NPC behavior schemes (`xr_*` catalog)

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F74 | Walker / patrol / remark | ✅ P1 | **DONE** → `flows/scheme-walker-patrol.md` (the canonical scheme template) |
| F75 | Camper / cover | ✅ P2 | **DONE** → `flows/scheme-camper-cover.md` |
| F76 | Combat schemes | ✅ P1 | **DONE** → `flows/scheme-combat.md` |
| F77 | Animpoint | ✅ P2 | **DONE** → `flows/scheme-animpoint.md` |
| F78 | Sleeper / wounded / death | ✅ P2 | **DONE** → `flows/scheme-sleeper-wounded-death.md` |
| F79 | Meet / abuse / bribe | ✅ P2 | **DONE** → `flows/scheme-meet-abuse-bribe.md` |
| F80 | Gather items / corpse loot | ✅ P2 | **DONE** → `flows/scheme-gather-corpse.md` |
| F81 | Sensors & misc schemes | ✅ P2 | **DONE** → `flows/scheme-sensors-misc.md` (most are services/binders, not schemes) |
| F82 | Companion system | ✅ P1 | **DONE** → `flows/companion-system.md` |

## Phase G3 — squads, server entities, simulation

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F83 | Server entity: stalker (Lua) | ✅ P2 | **DONE** → `flows/se-stalker.md` (se_stalker + se_trader; F16/F15/F72 override seams; `can_choose_alife_tasks(false)`) | F16, F15 |
| F84 | Server entities: others (Lua) | ✅ P2 | **DONE** → `flows/se-others.md` (whole `se_*` family; se_monster=F83 clone, se_item=~19-class catalog, se_actor=sim target, se_restrictor teleport routes, se_smart_cover loopholes, se_heli) | F16 |
| F85 | Scripted squad behavior | ✅ P2 | **DONE** → `flows/scripted-squad.md` (`sim_squad_scripted` update→target-FSM→assign + `simulation_objects.evaluate_prior` scoring; **gamedata override** routes assign via sim_board) | F41 |
| F86 | sim_board assignment | ✅ P2 | **DONE** → `flows/sim-board.md` (`SIMBOARD` singleton: smart registry + `assign_squad_to_smart` population bookkeeping + `get_squad_target` top-5 pick + `fill_start_position` world population) | F41 |
| F87 | Offline combat | ✅ P3 | **DONE** → `flows/offline-combat.md` (OCS: periodic coordinator, full/smart modes, `validate_enemy`→`simulate_battle` power/rank-based damage, relations+news outcome; crowded-levels-only; task/story squads excluded) | F41 |

## Phase G4 — binders & object schemes

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F88 | Binder architecture | ✅ P1 | **DONE** → `flows/binder-architecture.md` (object_binder lifecycle reinit/net_spawn/update/save; attach via bind_object from init/actor_init/bind; actor_binder + generic_object_binder[**gamedata override**] + motivator_binder[F66]; class_registrator clsid→client+server) | F5, F42 |
| F89 | World-object binders | ✅ P2 | **DONE** → `flows/world-object-binders.md` (two sub-patterns: scheme-running `generic_physics_binder`[+box loot] vs manager/registry campfire/camp/faction/light; all F88 deltas) | F28 |
| F90 | Anomaly-field binders | ✅ P2 | **DONE** → `flows/anomaly-field-binders.md` (anomaly_zone_binder = **artefact-spawn economy**+layers [resolves F27's "where artefacts spawn"]; anomaly_field_binder + dynamic-anomalies relocation system + activation veto) | F27 |
| F91 | Restrictors — passive `sr_*` | ✅ P2 | **DONE** → `flows/restrictors-passive-sr.md` (F66 scheme template on space restrictors; `sr_idle`=template; silence/no_weapon/timer/postprocess/light/particle/psy_antenna/deimos passive effects via F31 effectors) | F31 |
| F92 | Restrictors — active `sr_*` | ✅ P2 | **DONE** → `flows/restrictors-active-sr.md` (event-firing restrictors: teleport actor / spawn crows+monster / camera cutscene; + `sr_camp` `CCampManager` storytelling via bind_camp) | F36 |
| F93 | Physics-object schemes `ph_*` | ✅ P2 | **DONE** → `flows/physics-object-schemes.md` (all 13 files; door/button/code/idle/hit/on_hit/on_death[file=ph_death]/oscillate/sound READ-IN-FULL + car/minigun spine [CCar, job-mode SIMBOARD targeting]; `set_fastcall` channel; **ph_appforce unregistered=dead**; ph_sound rewired to Zone FM; phantom_manager=binder+spawn service not a scheme) | F28, F29, F36 |

## Phase G5 — monsters & vehicles

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F94 | Monster schemes `mob_*` | ✅ P2 | **DONE** → `flows/monster-schemes.md` (all 11 files READ-IN-FULL; home leash [230 refs]/walker patrol/remark/jump on the F37 action queue; generic death/trade; **mob_camp unregistered=dead, mob_combat dormant** [no caller]; mob_remark `snd=` + mob_death unknown-killer latent bugs; trader=CAI_Trader callbacks; run by gamedata bind_monster override + bind_trader) | F34, F36, F37 |
| F95 | Helicopter AI | ✅ P3 | **DONE** → `flows/helicopter-ai.md` (3-layer stack: binder health/`heli_enemy_flag` → always-on heli_combat [flyby⇄round⇄search, night/surge off, actor-not-enemy-by-default] → heli_move story scheme vs heli_alife faction-sim [squad-style FSM on simulation_objects + smart population + radio help API]; Lagrange flight planner; combat per-type bodies + warfare copy not read) | F29, F85, F86 |
| F96 | Cars / drivable vehicles | ✅ P3 | **DONE (merged into M12)** → `modlayer/gameplay/vehicles-heli.md` (bind_car = generic_physics_binder clone running stype_item → `ph_car` F93; se_car = F84 passthrough; driving = engine F29 — too thin for its own flow) | F29, F93 |

## Phase G6 — tasks, dialogs, economy

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F97 | Task system core (Lua) | ✅ P1 | **DONE** → `flows/task-system-lua.md` (CRandomTask/CGeneralTask read-in-full: tm_*.ltx wildcard config, engine complete/fail functor seam, live functor-recomputed text/target, condlist outcomes, repeat cooldown, marshal persistence; **verified bug: reward_item reads reward_money line**; ~50 story functor bodies not read) | F38 |
| F98 | Dynamic task types | ✅ P2 | **DONE** → `flows/dynamic-task-types.md` (the plugin pattern: namespace injection into task_functor/xr_conditions/xr_effects + tm_*.ltx consumers + `target`-as-argument + stage stepping + sim plugins w/ own save_state; clear_map read-in-full as exemplar + bounty spine + 22-file survey; ~19 bodies not read = leaves) | F38, F67, F86 |
| F99 | Dialog system | ✅ P1 | **DONE** → `flows/dialog-system-lua.md` (3 tiers: character_dialogs attachment → XML graphs → `<init_func>` script-built trees via `add_dialog`; dialog_manager priority-picked chatter from 854-section LTX, runtime-only state; dialogs.script ~250-functor census; **verified bugs: nil-global `PRT_subtable` in precondition_no_more, inverted+dormant anomalies smart gate, once=pin-at-255**) | F18 |
| F100 | Characters / info / encyclopedia | ✅ P2 | **DONE** → `flows/characters-info-encyclopedia.md` (npc_profile→specific_character template pick [no_random/class/delta/used-id rules]; info XML registration + info_portions.script switches; encyclopedia = Lua plugin over plugins/encyclopedia*.ltx, vanilla XML dormant; **drove the F40 correction: actor DOES fire eInventoryInfo→actor_on_info_callback on new infos**) | F40 |
| F101 | Trade & economy | ✅ P1 | **DONE** → `flows/trade-economy.md` (trade profiles: condlist re-pick on talk + eco-hour restock via engine CPurchaseList; buy/sell lists = blacklist-by-empty-value + per-item multipliers via preset inheritance; Lua UI intercepts engine StartTrade and runs price+transaction itself, cost×mult×discount×cond^exponent; axr NPC↔NPC gulag-job economy with non-conservative money; **verified bug: `for i=count,1` sell loops missing -1 step — only count==1 sells**) | F18, F8 |
| F102 | Item Lua behaviors | ✅ P2 | **DONE** → `flows/item-behaviors-lua.md` (self-registering callback plugins over the `_g.Parse_ITM` ini-driven type registry; itms_manager hub: use veto/specials/drag-combos/ItemProcessor; item_device is a real BINDER [`bind = item_device.bind`]; **gamedata fork: item_weapon.script** — demonized ammo_aggregation_full + per-weapon overheat/smoke; **verified bug family: `se_load_var(item_id,…)` nil-global in itms_manager/axr_companions×2/death_manager — strapped-weapon guards dead**; item_mine `-- UNUSED` but callbacks live) | F21, F8 |

## Phase G7 — actor systems & metagame

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F103 | Actor survival & status | ✅ P1 | **DONE** → `flows/actor-survival-status.md` (engine owns health/power/satiety/rad/boosters [F30]; Lua adds sleep/thirst [twin difficulty-toggled 0–10000 meters, `eat_sleepiness`/`eat_thirstiness` keys, forced sleep, thirst 5% collapse death] + arszi_psy [**confiscates type-4 hits in actor_on_before_hit — engine psy bar is display-only via SetPsyBar**; protection stack outfit+helm+belt arts+psi booster; zombification = dead code behind file-local false]; actor_status = engine-backed getters + booster scan + safe-cover SetEvent feed for surge + `UIIndicators` HUD with `add_indicator` ext point; actor_effects = `plugins/actor_effects.ltx` timed-action scheduler [ms = lua-line via loadstring] + play_continuous_effect for tools + broken-weapon click/fire-veto at cond ≤0.014; **verified quirks: shared blur ppe id 39568 cross-suppression between sleep/thirst, `get_satienty` typo is the real API, scan_safe_zone rewrite dead [old runs], force gates always-true vs 10001**; no gamedata override anywhere) | F30 |
| F104 | Actor menu & inventory UI | ✅ P2 | **DONE** → `flows/actor-menu-inventory-ui.md` (engine CUIActorMenu never opens: actor_menu_inventory functor seam → `ActorMenu_on_before_init_mode` flags.ret veto → singleton Lua `UIInventory` [4 modes]; actor_menu = engine-R_ASSERTed mode mirror → info portions + `set_msg`/`set_fade_msg` HUD API; declarative `self.properties` context menu w/ **`useN_functor`/`useN_action_functor` LTX custom rows = the modding surface**; quick slots = console `slot_N` state; LMode corpse `known_info` info transfer; RMode pure delegation to inventory_upgrades; **monkey-patch surface verified: gamedata callbacks_gameobject wraps TMode_Sell/Buy→synthesizes actor_on_item_sell/buy, injects CInventory_BeltWidth**; quirks: On_CC_Trasfer typo load-bearing, `add_stats_table` unusable [shared empty-guard w/ prepare_stats_table — corrected agent claim], helm-cast wrong-var printe + no nil bail, dead xr_meet_* flags) | F8, F19 |
| F105 | Warfare metagame | ✅ P2 | **DONE** → `flows/warfare-metagame.md` (root `_g.IsWarfare()` [new-game option OR persisted enable_warfare_mode from ui_mm_faction_select]; **registration UNCONDITIONAL + reactive: all callbacks registered every game, per-callback IsWarfare early-exits, resetter TimeEvent unregisters 60s after load** [corrected agent]; **stock squad AI suppressed per-squad via native `squad.__lock`, NOT method nulling — `override_functions` is dead code [no caller], stranding the heli_alife→heli_alife_warfare update swap** [corrected agent]; se_smart_terrain show/hide replaced for fog-of-war spots; capture: owned smart flips only when owner has 0 defenders present, count-vs-count only for unowned; 3-cadence tick actor 10s / faction 1-5min economy lerp + resurgence / smart 1-5min defense-invasion-patrol-mutants; player commander: money buys squads/heli, influence from kills staked on smarts, possess-leader body swap; m_data.warfare persistence; **verified bugs: calculate_target_density `level` vs `lvl` linked-level dup, update_faction `patrol_squads` undefined-global nil store**) | F41, F86, F95 |
| F106 | Game modes | ✅ P2 | **DONE** → `flows/game-modes.md` (NO AGENT, all 6 files READ-IN-FULL; ui_mm_faction_select seeds enable_*_mode flags + ironman{uuid,…} from axr_main.config character_creation; game_difficulties = gdiff_N/econ_N factor families from plugins/difficulty.ltx, per-key overrides in diff_game/diff_eco, push-applied via console cmds [g_game_difficulty/g_hit_pwr_modif/g_dispersion_*/wpn_degradation] + engine setters + feature toggles, tamper info portions; game_setup = MISNAMED: dynamic world-item conservation loop [spawn-on-take, deferred uses/cond via F102 alife_process_item] + new-game purges + 3 one-time map fixups + cordon army-squad removal; agony/timer = on_before_save_input vetoes; **ironman death = iterates ALL .scoc sidecars on disk, marshal-rewrites death_count per uuid, deletes saves at limit** + Second Wind life grants; azazel = death cancelled via flags.ret_value, body-swap into sim_default_ stalker w/ corpse mimic + identity copy + task cancel; **verified bug: arty/bkpk degradation share ONE toggle — on_game_load double-call means bkpk value wins registration [arty=true+bkpk=false → arty degradation silently off]**; quirks: 5× dead azazel dist_mult locals, squad_mates never filled, bleeding=1 heals [ChangeBleeding=wound incarnation, C++ verified]) | — |
| F107 | Game services | ✅ P2 | **DONE** → `flows/game-services.md` (NO AGENT, all 7 game_* READ-IN-FULL + plugins/achievements.ltx + C++ functor driver; game_statistics = rank/reputation authority [counters→rank/rept, band-crossing news tips from game_relations.ltx, NPC m_data tallies, _G.ARTY_FROM_CONT exploit guard, MARSHAL saves unlock-bools+counts NOT reward values]; game_achievements = 24 triplets **polled ONLY by the PDA Ranking tab — CUIRankingWnd/CUIAchievements C++ functor driver, no Lua poll**, rewards DISTRIBUTED via has_achievement() at ~15 sites; game_autosave_new = 60s save-tempsave timer; fast/backpack travel = PDA map-spot (F39) to whitelisted smarts/deployed backpacks w/ guard battery; **DEAD: game_autosave fully commented out, game_registrator vestigial MP clsid, bookworm_food commented out of section → unreachable achievement**; quirks: articles/killed_stalkers unused, rep skip in arena, DEV_DEBUG cheater flag, comment/info drift; no gamedata override) | F15 |
| F108 | Faction relations (Lua) | ✅ P2 | **DONE** → `flows/faction-relations-lua.md` (NO AGENT, game_relations.script 1044 + game_relations.ltx 313 READ-IN-FULL; standalone flow, no new map — engine tier mapped by F40, routed from M1 hub. DoctorX Dynamic Faction Relations: 5-component goodwill sum [engine-computed], ~35-script API [is_factions_enemies/squad/gulag goodwill], community-matrix seed + drx_df_* per-pair persistence, dynamic-relations kill engine [calculate_relation_change weighted+jittered, count-limit damped, warfare drops limits unless fog_of_war]; game_relations.ltx SHARED with engine F40 + game_statistics F107 bands + monsters F34/F94. **VERIFIED BUGS: factions_table_all order ≠ shipped communities_relations order → reset_all_relations new-game reseed reads mismatched cells; reset_goodwill `==f1 or "actor_"..f1` always-truthy precedence; friend_num dead branch; set_squad_goodwill_to_npc undefined-global → always neutral**) | F40 |

## Phase G8 — UI, addon layer, utils, PDA, sound

| ID | Flow | Pri | Key files | Engine xref |
|----|------|-----|-----------|-------------|
| F109 | Gameplay UI screens | ✅ P2 | **DONE** → `flows/gameplay-ui-screens.md` + map **M20** (NO AGENT, 6 files READ-IN-FULL [ui_inventory deferred to F104]; two engine seams CUIScriptWnd + HUD custom-static; ui_main_menu = SP menu + save-veto + main_menu_on_* callback hub + F2 new-game test; ui_enemy_health bar; ui_dyn_msg_box reusable toolkit [msg_box/multi_choice/context_menu/context_props]; ui_freeplay_dialog Pripyat prompt. **DEAD/DORMANT: ui_extra fully unreferenced, Lua ui_dosimeter has no caller [engine UIDosimeter.cpp renders it], ui_registrator vestigial MP clsid**) | F19 |
| F110 | PDA & its tabs | ✅ P2 | **DONE** → `flows/pda-tabs.md` (spine pda/pda_actor/pda_flagger READ-IN-FULL + engine bridge; 5 tab modules agent-traced + spot-verified. pda.script = engine set_active_subdialog tab dispatch + Top-100 ranking board [coc_rankings_*, same UIRankingWnd as F107] + get_stat panel + F39 map-spot bridge + contacts-on-talk + map spots; **eptContacts replaced by warfare tab when _G.WARFARE**. Tabs: contacts/dead-NPC-PDA[register_pda from death_manager, **1% kill_the_strelok egg**]/Zone-FM radio+player[hardcoded 2 ch]/relations matrix[relation_registry+F108]/warfare board[**sets warfare_actor_faction_domination ≥50% = iron_curtain gate F107**]. pda_actor/pda_flagger = warfare commander map-menu plugins. Quirks: is_widescreen dead branch, encyclopedia tab wc false-empty=F100 module) | F39 |
| F111 | UI debug tools | ✅ P3 | **DONE** → `flows/ui-debug-tools.md` (spine READ-IN-FULL: ui_debug_launcher open/inject/editor-launch + xrs_debug_tools main_loop/activate_feature/toggle-keys + debug_cmd_list API + ui_debug_main dormant + _g DEV_DEBUG + 3 gamedata files; editor interiors structure-surveyed only. **DEV_DEBUG (in _g.script) gates the whole subsystem** — inert in shipping build. 3 generations: ui_debug_launcher LIVE [Tronex "Anomalous Debugger" F7=kCUSTOM17, inject API, CUIScriptWnd spawners/editors] / ui_debug_main DORMANT [opener commented out] / xrs_debug_tools HUD INSPECTOR [Alundaio, main_loop @1s → 3 custom-static panels via _G[..]["_show"] dynamic dispatch: squad/planner/heli/monster/zone; toggle 1/2/3]. debug_cmd_list = command registry. Gamedata delta: zz_ui_debug_inputs [input-history monkeypatch], imgui_lua_debug [Lua remote debugger menu], options_modded_exes_debug [F17 options page]) | — |
| F112 | News & notifications | ✅ P3 | **DONE** → `flows/ui-news-tips.md` (news_manager + dynamic_news_helper READ-IN-FULL, dynamic_news_manager spine read + composers surveyed. TWO tiers meet only at engine give_game_news F39: Tier1 news_manager = universal synchronous dispatcher ~50 callers [send_tip/send_sound/send_task/send_treasure/relocate_item/relocate_money + news_tips_icons.ltx]; Tier2 Dynamic News singleton [ARS→Alundaio→MLR→Tronex] = self-scheduling generator: ~27.5s startup delay, per-faction queue+channel_status, 6 TimeEvent ticks [TickNews drain 5-20s→helper.send_tip + TickQuick reaction; TickSpecial/Task/Random/Companion generators] + 3 death/loot callbacks, **NewsToggle gate = paused/invalid-map/surge-erases-queue/PDA-charge + comm-centre on-off + WelcomeToNetwork once**; helper = level_points landmarks+GetPointDescription compass, GetTimeString 6-tier phrasing, faction_expansions pickers, OWN send_tip [sound-by-type + drains PDA battery slot 8]. Quirks: shw_stash hardcoded false, surge-death reports = *_Fake fabricated, PickRandomKey dead+broken, GetTimeString tier-5 hrs_R bug. No gamedata override) | F39 |
| F113 | AXR behaviour add-ons | ✅ P2 | **DONE** → `flows/axr-behaviour-addons.md` (axr_beh + axr_beh_patches + axr_keybind READ-IN-FULL; 6 combat/utility schemes structure-surveyed; gamedata axr_beh_patches diffed = identical. Every add-on is a GOAP scheme on the F37 interface except axr_keybind. **axr_beh** [188111, active_section] = companion escort/patrol/cover brain [drives axr_companions F114, source of npcx_beh_* flags F110]; set_desired_target vocab commander/actor/cover/waypoint/task_target_*; handlers beh_wait/cover/move/path; use_camp branch COMMENTED OUT. **axr_beh_patches** [GhenTuong] monkeypatches set_scheme/execute/finalize for [beh@section] init/main/finalize hooks. Combat schemes [share validate/lmove + try_go_cover/strafe/backward]: fight_from_cover/stalker_panic/npc_vs_heli[F95]/radio_in_heli[F112]/npc_vs_box/turn_on_campfire[gulag jobs F72]. **axr_keybind** = scriptable key→functor [kCUSTOM*/DIK], rebindable via axr_main.config [global_keybinds]) | F37 |
| F114 | AXR managers | ✅ P2 | **DONE** → `flows/axr-managers.md` (axr_task_manager callbacks+honcho-tables+CFG_CACHE+get_first_available_task READ-IN-FULL, drx_sl_* generators+dialog builders surveyed; axr_companions MANAGER layer READ-IN-FULL [roster/command-system/special-squads/inventory/wheel/timers — deepens F82's agent-traced half]; axr_trade_manager routed to F101. **axr_task_manager** = DRX-storyline task GENERATOR on engine task core F97 — bounties_by_id/hostages_by_id registries, drx_sl_main_honcho_table+drx_sl_honchos_table, get_first_available_task = per-NPC picker [3hr stored-task cache + 75% gate + section-prefix pool + prereq/sim_communities filter]; dialog surface generate_available/ongoing/finished_tasks. **axr_companions** manager = roster [companion_squads/non_task_companions vs max_actor_squad_size 2→4 patriarch] + 5-axis command system writing npcx_beh_* [drives axr_beh F113] + special/story escort squads [setup_companion_task spawns + hostages_by_id] + inventory assignment [30kg cap] + command wheel kCUSTOM18 + healing/unstuck timers. **axr_trade_manager** = NPC↔NPC gulag economy, owned by F101. Quirks: bounties never fail [stage 3→1], 30kg cap hardcoded, unstuck = teleport hack, se_load_var strapped guard dead [F102 family], epic_hack = misnamed heli helper. No gamedata override) | F97, F101 |
| F115 | Utility libraries | ✅ P1 | **DONE** → `flows/utils-libraries.md` (catalog read: full function inventory of all 6 utils_* [~500 funcs] + load-bearing regions READ-IN-FULL [utils_obj cover/movement validate/lmove/try_go_cover, utils_data cfg_file class + parse_* family, utils_item get_item_cost/trade_status]; utils_ui/xml/stpk structure-surveyed. 6 flat helper libs, the cross-cutting layer ~everything calls into: **utils_obj** objects/squads/nav + the cover/movement PRIMITIVES combat schemes reuse [db.used_level_vertex_ids reservation]; **utils_item** item/weapon/outfit/addon + CANONICAL get_item_cost [on_get_item_cost callback seam, consumed by trade F101] + upgrade tree; **utils_data** math/vector/CTime + parse_condlist/parse_* [the scheme/condlist engine F66 primitives] + cfg_file [Lua read/WRITE INI, vs read-only engine ini_file — persists axr_main.config/plugin LTX] + findfunction; **utils_xml** UI XML load + resolution scaling + XmlParser DOM; **utils_ui** the LUA INVENTORY-CELL ENGINE UICellItem/UICellContainer [backbone of F104]; **utils_stpk** server cse packet read/WRITE [get_/set_*_data + parse_/fill_*_properties_packet mirroring C++ STATE_Read/Write, fragile]. No gamedata override of any 6) | F42 |
| F116 | Sound & dynamic music | ✅ P3 | **DONE** → `flows/sound-music.md` (xrs_dyn_music + aaa_sound_object_patch READ-IN-FULL; xr_sound get_sound_object region read; sound_theme/sound_manager/sound_ambient/mob_sound/ph_sound/xr_zones_sound surveyed; level_weathers + game_levels_music.ltx characterized; gamedata diffed. NO single sound manager. TWO music systems no shared code: engine ambient from game_levels_music.ltx [time-of-day playlists, snd_volume_music] vs **xrs_dyn_music** Lua combat music that DUCKS it — main_loop AddUniqueCall ~1s, gated IsDynamicMusic(); start on IsStalker enemy [ONLY stalkers, NOT monsters], 4 HARDCODED themes ×3 parts gapless-chained, crossfade via snd_volume_music, surge coupling, silence zones partly hardcoded. **xr_sound** = scheme-facing player + get_sound_object random-no-repeat selection + get_safe_sound_object path cache. **sound_theme** = config theme registry [npc/actor/object/looped_sound] read by F112 spoken news. **aaa_sound_object_patch** [aaa_ loads first] = _G.sound_object override w/ soundCache GC [gamedata copy cosmetic-only]. Scheme sounds: sound_manager [sm campfire storyteller CStory], sound_ambient [level ambient/wind/rain-helmet], ph_sound, xr_zones_sound, mob_sound. level_weathers = Lua weather cycle driver [tangential, feeds CEnvironment]. gamedata: options_modded_exes_sound OVERRIDE +sound-delay subpage [NEW]. Quirks: combat music ignores monsters, themes hardcoded, 2 cooperating sound caches) | — |
| F117 | Treasures & secrets | ✅ P3 | **DONE** → `flows/treasures-secrets.md` (treasure_manager READ-IN-FULL + tasks_stash surveyed + both config tiers characterized. Tronex "based on CoC treasure manager". PROCEDURAL: spawns loot in ALL INVBOX, content decided new-game + frozen in save, lazy-spawn on open. Pool from items\settings\treasure_manager.ltx → item_list[kind][tier] (economy-eco filtered); set_random_stash fills random weight budget by shuffle kind→tier→roll chance×multi, inc anti-stall, rag_and_bone +200. RARE stash gates toolkit reward on toolkit_reward_count>=task_num. try_spawn_treasure materializes csv → weapons get random addon/ammo/0-30%cond via utils_stpk F115. caches[id] state machine false/csv/true; actor_on_item_take_from_box → stashes_found stat + send_treasure F112 + item_radio.clear_stash. on_game_load registers INVBOX + fills HALF. dual persistence MARSHAL+net-packet. PREDEFINED secrets.ltx (misc/ + per-level) = authored item=count,prob legacy path (se_item/task_functor/dialogs), coexists with procedural. tasks_stash = DRX quest stash effects. item_radio RF stash hints. Quirks: loot new-game-locked, only HALF boxes filled, debug-only "Save as Rare Stash" map tool. No gamedata override) | F8 |

## Phase G9 — config / data tier

| ID | Flow | Pri | Key files (`unpacked/configs/`) | Engine xref |
|----|------|-----|----------------------------------|-------------|
| F118 | Creatures & actor params | ✅ P2 | **DONE** → `flows/creatures-actor-configs.md` (configs/creatures/ data tier, catalog read. Include structure: monsters.ltx→all m_*, stalkers.ltx→m_stalker/person/immunities/damages, spawn_sections.ltx→27 per-level. INHERITANCE [child]:parent: monster_base [in defines.ltx] → m_<species>_<variant> → spawn_section variant → spawned. m_*.ltx anatomy: ALife identity/stats/ef_*_type/bones/influences[fire-psy-rad]/physics-boxes + INLINE immunity/damage/protection/vision sub-sections [monster immunities per-species inline, NOT immunities.ltx]. immunities.ltx = named STALKER resistance presets [no/low/medium/high]. damages.ltx = bone-damage profiles [body_damage_*, per-bone armor+multiplier]. game_relations.ltx = rank/rep/rating tables + matrix → OWNED BY F40/F108, routed not dup. actor.ltx = player creature def + movement coefs/mass/attachables/NV/survival. spawn_sections_*=per-level spawn catalog→F123. helicopter.ltx→F95. Quirks: 2 immunity mechanisms, monster_base in defines.ltx, retune global=m_* / per-level=spawn_sections, DLTX no full overrides. No gamedata override) | F34, F30 |
| F119 | Zones & anomaly params | ✅ P2 | **DONE** → `flows/zones-anomaly-configs.md` (configs/zones/ data tier, catalog read. zones.ltx = #include manifest → zone_base inheritance root [carries script_binding = bind_anomaly_field.bind, the F90 artefact-binder seam] → class = ZS_* section→CustomZone-subclass mapping [ZS_MINCE/GALAN gravi, ZS_MBALD ~13, ZS_RADIO fields+monolith, ZS_BFUZZ grabbers, ZS_TORRD/Z_TORRID thermal, ZS_NGRAV, Z_CFIRE] → thin weak/avg/strong inheritance tiers. Param groups: strength/hit-type[strike/radiation/telepatic/burn → F30]/blowout-FSM/particles/light/impulse/teleport-corpse/wind/postprocess-preset. TWO families: zone_field_* passive DoT vs zone_mine_* blowout; oddballs monolith[psy emitter, artefact econ off] + campfire[own binder F89]. **config artefact fields ≠ live spawning — Lua anomaly_zone_binder F90 owns it via zone_base script_binding**. ef_anomaly/weapon_type = AI eval-fn family [F118]. **sr_teleport_sections.ltx = level-changer table for sr_teleport active restrictor F92 [enable/spot/text/hint/path/reject_path/autosave], NOT an anomaly**; per-script *_teleport*.ltx = F121. DLTX no full overrides, none at writing) | F27 |
| F120 | Items / weapons / outfits configs | ✅ P2 | **DONE** → `flows/items-weapons-outfits-configs.md` (configs/items/ data tier, biggest cluster ~134 weapon files. system.ltx → item_base.ltx manifest → 4 family roots. **WILDCARD AUTO-INCLUDE = the modding contract**: weapons/base.ltx #include "w_*.ltx"+"m_*.ltx", outfits/base_includes.ltx "o_*.ltx", items/items/base.ltx "items_*.ltx" — drop a new prefixed file, never edit shipped ones [NOTICE FOR MODDERS]. WEAPONS w_* [w_ak74 exemplar]: root :identity_immunities,weapon_probability,default_weapon_params,*_sounds + parallel *_hud:hud_base + variant subsections [cosmetic _alt/_custom/_modern/_old/_rpk + attach-state _1p29/_kobra/_ps01]; clusters identity/ballistics/mag-ammo/attach-slots[status 0/1/2]/condition-jam/recoil/AI-hit-chance/upgrades/HUD-anim-map → F22 + Lua item_weapon fork F102. OUTFITS o_* [o_seva]: outfit_base[class EQU_STLK, use1_functor=gameplay_disguise.menu_patch] + protection block + SEPARATE immunities_sect [2 mechanisms] + bones_koeff→damages F118 → F23; outfit_restrictions_1..5 gating + **_temp dummy must stay LAST [engine parse bug]**. ITEMS items_* = consumable/misc categories → F21/F24/F25/F102. AMMO weapon_ammo.ltx k_* ballistic coeffs → F22. ADDONS+NV scope presets → F22. UPGRADES item_upgrades.ltx → up_gr_* trees F26. Shared base parents in defines.ltx → F122. CROSS-TIER route-not-dup: trade/ → F101, settings/ → F117/F102/F89/F106/F123. No gamedata override [no dir]) | F22, F23 |
| F121 | Per-smart logic configs | ✅ P2 | **DONE** → `flows/per-smart-logic-configs.md` (configs/scripts/ data tier ~2016 files = the custom_data / logic-LTX content. **NOT #included — loaded by NAME** as object/smart custom_data pointer; top-level reusable templates + per-level subdirs [jupiter 206/zaton 177/pripyat 130/…]. TWO file roles: object logic [single [logic] active= FSM + on_* condlist transitions, F68 — door/sr_sleep/sr_teleport_logic] vs gulag/smart logic [many [logic@job] w/ prior/suitable{condlist}/monster_job = F73 rankable JOB CATALOG — walker/camper/surge-hide+out_restr/animpoint-camp day-night-rain/mob_home mutant-swarm; IN-FILE [child]:parent inheritance factors shared job fields]. smart/<smart>.ltx = smart respawn def [[smart_terrain]+respawn@+spawn_squads, F72/F86] DISTINCT from *_smart_logic.ltx. camp.ltx SPECIAL = campfire storyteller registry [stories/guitar/harmonica → sound_manager sm F116, not FSM]. scripts/tasks/ → F97/F98. DLTX deltas apply: 1 gamedata override labx8/mod_lx8_door_lock_fix.ltx = @[ph_button@unlocked] delta. QUIRKS: reference-by-name [typo=no logic]; smart def≠job catalog; suitable+prior decide assignment) | F66, F73 |
| F122 | Plugins / tweaks / presets | ✅ P3 | **DONE** → `flows/plugins-tweaks-presets-configs.md` (the constants/shared-foundation tier. **defines.ltx = inheritance ROOT** [#included FIRST, bottom of every [child]:parent]: hud_base + immunity presets + item chain global_item_actions→identity_immunities→booster[consumable eat_*/boost_*] + **weapon parents weapon_probability+default_weapon_params [F120 routes here]** + **creature root monster_base [F118 routes here]** — confirms "lives a dir up". presets/ = video[→F17/renderer]+economy_diff+gameplay_diff[→F106 namespaced option-key tables]+warfare[→F105] via includes.ltx. **axr_options.ltx = EMPTY stub** [real options = F17 options_modded_exes_*]. plugins/[26] + ai_tweaks/[18] = per-system tuning data ROUTED to owners [achievements F107/actor_effects F103/difficulty F106/disguise F102/dynamic_anomalies F90/encyclopedia F100/guaranteed_drops F102/news_tips F112/xr_sound F116/xrs_debug F111; xr_combat_ignore F76/simulation_objects F85-F86 linked-level/axr_companions F114/etc]. Insight: tweak numbers here, behavior owned by system flow [router + 2 owned: defines+presets]. No gamedata override) | F17 |
| F123 | Spawns / ALife / graphs | ✅ P3 | **DONE (FINAL ITEM)** → `flows/spawns-alife-graphs-configs.md` (the world-instance + ALife tier. **spawns/all.spawn = BINARY authored-world file** [unpacked/spawns/]: every cse_alife_* static object placement [smart_terrains/restrictors/anomalies/INVBOX/story NPCs/level-changers] + way_ patrol paths + graph xrefs; SDK/ACDC-built, loaded at NEW GAME to seed ALife F16/F41, save diverges after, not text. The INSTANCE layer for templates F118-F122 + logic F121. alife.ltx [alife] = simulator tuning [schedule/objects_per_update/time_factor 6/switch_distance 750m/auto_switch/start_game_callback=_G F65] → F16. game_graphs.ltx = ~200 graph-point NAME catalog + level membership [→ sim_board F86/squads F85/transitions]. game_story_ids.ltx [story_ids] = numeric story-ID→object-name = STABLE script handle [alife_object_by_story_id]. game_maps_single.ltx = level registry + PDA world-map layout + per-level music [F39/F116]. **patrol_paths.ltx EMPTY [paths live as way_ in all.spawn] + dynamic_spawn.ltx stub**. UNIFYING: templates×placements[all.spawn]×logic[F121]×constants[F122] = the world. QUIRKS: all.spawn binary+new-game-only [edit doesn't change saves]; patrol_paths/dynamic_spawn stubs; story ids stable handle. No gamedata override) | F16, F41 |

---

## Per-flow execution protocol

Identical to `ROADMAP.md` → "Per-flow execution protocol (the validated recipe)": one read-only
Explore agent (tight contract, no line numbers, no pasted code) → spot-verify the spine yourself →
write `flows/<name>.md` (`verified: code`, honest provenance) → wire into `INDEX.md` + this file + the
owning `modlayer/gameplay/` map (promote any `(INFERRED)` it resolves) → integrity check. For small,
already-understood schemes, do it inline without the agent.

## Definition of done

**Tier 4 — gameplay corpus coverage (M1–M23 + F65–F123).** Each L1 map is a verified `map` (structure
verified, mechanics delegated to flows); each flow is `verified: code` with its spine spot-verified and
agent-traced/INFERRED parts marked, wired into `INDEX.md` + this file + its map. Integrity green (0 code
fences, no `:\d` line refs outside CONVENTIONS, all `sources:`/`related:` resolve). `unpacked/` cited as
canonical with `gamedata/` deltas noted. When met, the KB covers **engine (F1–F45) + renderer (F46–F64)
+ gameplay (F65–F123)** — the whole game, both halves.

## Maintenance

- This file + `INDEX.md`'s audit table are the two sources of truth for gameplay status; keep in sync.
- If a flow splits or merges on contact with the code (likely for the big scheme/`sr_*`/`ph_*`
  batches), edit the table — keep the count honest, don't let it drift into fiction.
- New scheme families or addon systems discovered mid-work get a new F-row here before being documented.
- Always re-check for a `gamedata/scripts/` override of any `unpacked/` file you cite (the override is
  what runs).
