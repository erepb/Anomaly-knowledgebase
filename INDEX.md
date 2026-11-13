---
domain: meta
tags: [meta, engine, modlayer]
sources: [claudedocs/kb/, src/, gamedata/, README.md]
related: [GLOSSARY.md, CONVENTIONS.md]
updated: 2026-11-13
synced: 2026-11-13
synced_branch: all-in-one-vs2022-wpo-mt
synced_commit: 3470feeb2
---

# Knowledge Base — Index

> # 🔄 KB SYNCED TO: **2026-11-13** — `all-in-one-vs2022-wpo-mt` @ `3470feeb2`
>
> **Everything in the repo up to that commit is reflected here. Anything newer is not.**
>
> **To sync again:** `git log --no-merges 3470feeb2..HEAD` (cross-check `README.md` "Changelog"),
> reconcile the affected KB files, then update the three `synced*` fields above, add an entry to the
> [Sync log](#sync-log), and bump `updated:`. Full procedure: `CONVENTIONS.md` → "Repo sync protocol".
>
> Sync history: 2026-11-13 @ `3470feeb2` ← 2026-08-17 @ `becde89ee` ← 2026-06-26 @ `d7c39ae2` (KB creation: 2026-06-04/05 audit).

> ✅ **AUDIT COMPLETE (2026-06-04; deep second pass 2026-06-05) — no `partial` files; `engine/xrGame.md` is a verified `map` by design.**
> **Flow buildout (2026-06-05): all 20 flows now documented (F1–F20) — Phases 1–4 of `ROADMAP.md`
> complete.** Phase 1 (player core): object-lifecycle, actor, weapon-state-machine, inventory,
> damage-hit. Phase 2 (AI): ai-planner, ai-perception, ai-squad-logic. Phase 3 (engine pipelines):
> engine-boot-frame, render-frame. Phase 4 (persistence/systemic): save-load, alife-online-offline,
> options-runtime, dialog-trade, hud-ui-load, animation-callbacks. These are the cross-cutting
> *mechanism* spines. **Concrete xrGame gameplay subsystems (weapons/outfits/detectors/zones/vehicles/
> monster-AI/tasks/relations/…) are enumerated as F21–F45 in `ROADMAP.md` Phase 5+ — buildout in
> progress.** Done so far: **F22 weapon hierarchy & addons** (`flows/weapon-hierarchy-addons.md`),
> **F23 outfits & helmets** (`flows/outfit-helmet.md`), **F24 detectors & artefacts**
> (`flows/detector-artefact.md`), **F26 upgrade system** (`flows/upgrade-system.md`), **F27 anomaly
> zones** (`flows/anomaly-zones.md`), **F30 actor condition & survival** (`flows/actor-condition.md`),
> **F40 relations & info-portions** (`flows/relations-info.md`), **F38 tasks** (`flows/tasks.md`); P2
> so far: **F21 item base** (`flows/item-base.md`), **F31 camera/pp effectors**
> (`flows/camera-pp-effectors.md`), **F42 CScriptGameObject** (`flows/cscript-game-object.md`), **F25
> consumables/explosives** (`flows/consumables-explosives.md`), **F39 PDA/map/news**
> (`flows/pda-map-news.md`), **F41 ALife services** (`flows/alife-services.md`), **F28 physics/ragdoll**
> (`flows/physics-objects-ragdoll.md`), **F29 vehicles/holder** (`flows/vehicles-holder.md`), **F34
> monster AI control** (`flows/monster-ai-control.md`), **F32 stalker execution**
> (`flows/stalker-execution.md`), **F33 smart-cover** (`flows/smart-cover.md`), **F37 script-driven NPC
> actions** (`flows/script-npc-actions.md`), **F35 agent/squad combat** (`flows/agent-squad-combat.md`); P3:
> **F36 special-actors** (`flows/special-actors.md`), **F43 level-runtime** (`flows/level-runtime.md`),
> **F44 wallmarks/particles** (`flows/wallmarks-particles.md`).
> **All P1 + P2 + P3 subsystem flows are now done (F21–F44)** — only OOS **F45** (multiplayer netcode,
> irrelevant to Anomaly SP) remains, on demand. The KB now covers the mechanism level AND essentially
> every single-player gameplay subsystem; `engine/xrGame.md` has no residual `(INFERRED)` markers.
> **Renderer (`src/Layers/`) subsystem buildout (F46–F64, see `ROADMAP.md` Phase 11+): 15 done — entire
> P1+P2 set + P3 particle/environment.** F46 resource/shader (`flows/renderer-resources-shaders.md`),
> F47 blender compile (`flows/renderer-blender-compile.md`), F48 R_Backend/RCache
> (`flows/renderer-backend.md`), F49 scene-graph/dsgraph (`flows/renderer-scene-graph.md`), F50
> visibility (`flows/renderer-visibility.md`), F51 visual types (`flows/renderer-visual-types.md`), F52
> skeleton (`flows/renderer-skeleton.md`), F54 dynamic lights (`flows/renderer-lighting.md`), F55 sun
> cascades (`flows/renderer-sun-shadows.md`), F56 post/SSFX (`flows/renderer-post-process.md`), F57
> wallmark engine (`flows/renderer-wallmark-engine.md`), F58 particles (`flows/renderer-particles.md`),
> F59 environment (`flows/renderer-environment.md`), F60 HW device (`flows/renderer-hw-device.md`), F61
> render factory & bridges (`flows/renderer-factory-bridges.md`), F53 detail/grass
> (`flows/renderer-detail-grass.md`), F62 R1 reference path (`flows/renderer-r1-path.md`), F63 R2/R3
> generations (`flows/renderer-r2-r3-paths.md`), F64 DX10 3D fluid (`flows/renderer-3dfluid.md`) — **ALL
> 19 renderer flows (F46–F64), `src/Layers/` fully documented**: the active R4 pipeline end-to-end
> (device → resources → cull → scene-graph → visuals/skeleton/detail/particles → submit → lighting/sun →
> wallmarks → post → environment) + the engine↔Layers seam + every alternate/dormant path (R1, R2/R3,
> DX10 fluid), atop the mechanism tier (`engine/renderer.md` map + F14 `flows/render-frame.md`).
> **Nothing left in the renderer.**
>
> 🟡 **GAMEPLAY CORPUS (`unpacked/`) — BUILDOUT IN PROGRESS (plan `ROADMAP-gameplay.md`).** The ~456
> base-game Lua scripts + the `unpacked/configs/` data tier (the game's *gameplay* half: schemes, jobs,
> squads, tasks, dialogs, trade, survival, warfare, UI) are planned as **F65–F123 + 23 L1 maps** in a
> dedicated sibling roadmap **`ROADMAP-gameplay.md`** — same KB, new gameplay tier under
> `modlayer/gameplay/`. **Done so far: Phase G1 spine (F65–F73) + Phase G2 schemes (F74–F82) + Phase G3
> (F83–F87: `se_*` server entities + squad simulation + offline combat) + Phase G4 ✅ COMPLETE (F88 binder
> architecture, F89 world-object binders, F90 anomaly-field binders, F91+F92 `sr_*` restrictors, F93 `ph_*`
> physics-object schemes) + Phase G5 ✅ COMPLETE (F94 `mob_*` monster schemes, F95 helicopter AI, F96 cars
> merged into M12) + Phase G6 ✅ COMPLETE (F97 task system core, F98 dynamic task types, F99 dialog
> system, F100 characters/info/encyclopedia, F101 trade & economy, F102 item Lua behaviors)
> + Phase G7 in progress (F103 actor survival & status ✅, F104 actor menu & inventory UI ✅ —
> M17 domain complete, F105 warfare metagame ✅, F106 game modes ✅, F107 game services ✅ —
> M19 domain complete, F108 faction relations (Lua) ✅) + Phase G8 in progress (F109 gameplay UI
> screens ✅ — M20 scaffolded, F110 PDA & its tabs ✅, F111 UI debug tools ✅, F112 UI news/tips ✅ —
> M20 ui_* flow coverage complete, F113 AXR behaviour add-ons ✅, F114 AXR managers ✅ — M21 domain
> complete, F115 utility libraries ✅ — M22 scaffolded, F116 sound & dynamic music ✅, F117 treasures &
> secrets ✅ — **Phase G8 complete**; Phase G9 in progress: F118 creatures & actor configs ✅ — M23
> scaffolded, F119 zones & anomaly params ✅, F120 items/weapons/outfits configs ✅, F121 per-smart logic
> configs ✅, F122 plugins/tweaks/presets ✅, F123 spawns/ALife/graphs ✅ — **Phase G9 COMPLETE; the entire
> gameplay corpus M1–M23 + F65–F123 is DONE**)
> + maps M1/M5/M6/M7/M9/M10/M11/M12/M13/M14/M15/M16/M17/M18/M19/M20/M21/M22/M23.**
> **✅ GAMEPLAY CORPUS COMPLETE.** The KB now covers **engine (F1–F45) + renderer (F46–F64) + gameplay
> (F65–F123)** — the whole game, both halves. No roadmap items remain (OOS F45 MP netcode aside). Cold
> sessions: the KB is in maintenance mode — fix/deepen files on contact per `CONVENTIONS.md`. Engine↔gameplay references are
> normal `related:` links; existing bridge flows (F12/F33/F36/F37/F38/F40/F41/F1/F3/F18/F20/F42/F39) get
> linked, not duplicated.
>
> Each new
> flow was agent-researched then spine-spot-verified against code (see per-file audit rows). Earlier
> KB content mixed code-verified facts with prose from the README / general X-Ray knowledge. Every
> file has been re-checked against source: domain + flow files are `verified` against code, and the one
> map file `engine/xrGame.md` is now a verified `map` (structure verified; every major mechanism
> delegated to a verified flow; residual breadth items marked inline *(INFERRED)*) — **no `partial`
> files remain.** Authoritative per-file status is the Audit status table below. The 2026-06-05 pass
> then drove almost all remaining inline *(INFERRED)* markers to
> code by reading the actual implementations (FS dedup, robin_hood, materials, ALife offline brain,
> full weapon-fire→bullet-sim chain, scheduler being serial-not-MT, xrCPU_Pipe/ReShade/compressor,
> blender pattern, hot-reload, kernel-script load gating, xrGame dependency edges). What still carries
> an *(INFERRED)* marker is now only genuinely-external or disproportionate-to-read material, listed at
> the bottom. Authoritative per-file status is the Audit status table below. **Rule going forward: no
> statement that isn't derived from reading code (cite `file:symbol`).**

Entry point for the `xray-monolith` repo KB. This is a local-only, self-maintaining map of the
codebase so a session can get architecture + domain context without re-grepping. Read order and the
full file map are below.

## What this repo is

Hybrid codebase: a **C++ X-Ray monolith engine** (`src/`) plus a **Lua + config mod layer**
(`gamedata/`) producing patched "modded exes" for S.T.A.L.K.E.R. Anomaly 1.5.3. Base-game content is
not included — only the mod's engine patches and `gamedata/` additions. See `GLOSSARY.md#project-shape`.

## Branch caveat

The KB is gitignored, so it lives in the working tree and **does not switch with `git checkout`** —
one KB serves all branches. The engine/modlayer files describe the mainline working branch. Any file
documenting branch-specific work carries a `branch:` field in its frontmatter and a banner (e.g.
`features/spom.md` → `spom` branch). Don't trust a branch-specific file's in-tree code pointers
unless you're actually on that branch (`git branch --show-current`).

## How to use the KB

0. Glance at the **sync banner** at the top. If `git rev-parse --short HEAD` differs from
   `synced_commit`, the tree has moved since the last sync — the KB is still the right starting point,
   but verify claims in the area you touch (`git log --no-merges <synced_commit>..HEAD -- <path>` shows
   whether *your* area changed), and fix what you find per the self-maintenance rules.
1. Always read `GLOSSARY.md` first (concepts: DLTX, DXML, unlocalizer, R1–R4, callbacks, …).
2. Read `CONVENTIONS.md` if you will edit the KB; read `METHODOLOGY.md` to understand the L0–L3
   depth layers and to add flows/digests.
3. For "where does X live" use the **domain files** (L1, below). For "how does X run, in what order"
   use the **flows** (L2/L3, `flows/`).
4. Jump to the relevant file, then follow its `## Deeper` pointers into the actual code. The tree is
   the `related:` link graph, not the folder layout.
5. If the KB is wrong or thin for your task, fix the relevant file before finishing (see
   `CONVENTIONS.md` self-maintenance protocol).

## Load order

`GLOSSARY.md` → (`CONVENTIONS.md` if editing) → relevant domain file(s) → code.

## File map

### Top level (always relevant)
- `GLOSSARY.md` — repo-specific concept dictionary. **Read first.**
- `CONVENTIONS.md` — KB format, tags, self-maintenance protocol.
- `METHODOLOGY.md` — how the codebase is analyzed/divided: granularity layers (L0–L3), domain
  decomposition procedure, flow/digest formats, and the rollout backlog of flows to document.
- `ROADMAP.md` — the **coverage plan & tracker**: the full bounded set of ~20 flows (F1–F20), what's
  done vs pending, phasing, and the per-flow execution recipe. **Read after a compaction to know what
  to do next.** This + the audit table below are the two sources of truth for status.

### flows/ — L2 flows & L3 function digests (how things run)
- `flows/lua-callback-lifecycle.md` — engine Lua boot → module bootstrap → callback bus →
  dispatch, incl. Pre/Normal/Post tiers + before/after ordering (`on_xml_read` is an ordinary bus
  callback since 2026.10.11). Flagship/template. tags: lua, callbacks, flow, lifecycle
- `flows/dltx-config-load.md` — LTX parse + DLTX mod-file discovery/merge (operators, depth
  precedence, fatals). tags: config, dltx, ltx, flow
- `flows/dxml-xml-interception.md` — engine XML → `COnXmlRead` → `xml_obj` mutate → reload; modxml
  registration + character hooks; API guide is `docs/DXML.md`. tags: dxml, lua, flow
- `flows/weapon-fire-callbacks.md` — FireStart→state_Fire→OnShot→FireBullet→bullet→hit; per-object
  weapon callbacks + misfire/jam. tags: weapons, actor, callbacks, flow
- `flows/object-lifecycle.md` — **backbone**: CGameObject spawn→per-frame (UpdateCL RT vs
  shedule_Update deferred + Crow)→destroy; scheduler/ALife/Lua seams. tags: lifecycle, scheduler,
  alife, callbacks, flow
- `flows/actor.md` — player `CActor`: spawn, control on the deferred channel (input→movement→
  physics) vs HUD/camera on real-time, condition, hit (eHit), death. tags: actor, input, camera,
  condition, damage, flow
- `flows/weapon-state-machine.md` — weapon states (show/idle/fire/reload/zoom/hide); HUD-animation
  timing drives transitions via UpdateCL→OnAnimationEnd→SwitchState. tags: weapons, hud, animation,
  state-machine, flow
- `flows/inventory.md` — items take/place(slot/belt/ruck)/activate/drop; deferred active-slot
  machine + GE_OWNERSHIP_* event seam. tags: inventory, items, callbacks, flow
- `flows/damage-hit.md` — shared hit resolution: SHit/GE_HIT → bone scale → ConditionHit (health/
  wounds) → death/eDeath; the chain actor & weapon-fire feed into. tags: damage, hit, condition,
  death, flow
- `flows/ai-planner.md` — NPC think loop: shedule_Update→Think→brain GOAP (evaluate→goal→A* solve→
  execute action); GetScriptControl seam to Lua xr_logic. tags: ai, goap, planner, lua, flow
- `flows/ai-perception.md` — sense→memory: Feel vision/sound/hit → CMemoryManager sub-managers →
  enemy/danger selection the planner reads. tags: ai, perception, memory, flow
- `flows/ai-squad-logic.md` — Lua orchestration: smart_terrain job → xr_logic scheme → binder →
  engine script-control. Base Lua (unpacked/) + repo bind_monster override. tags: ai, smart-terrain,
  xr-logic, lua, flow
- `flows/gameplay-boot.md` — **(F65)** gameplay boot/load order: engine→`_g.start_game_callback`→
  `axr_main.on_game_start` + core-manager boot; `class_registrator` class binding; `LoadScheme` scheme
  registry. tags: gameplay, unpacked, boot, callbacks, flow
- `flows/scheme-engine-core.md` — **(F66)** the NPC logic framework: `xr_motivator` per-frame tick +
  `xr_logic` activate/switch/issue_event; condlist-triggered section switching. tags: gameplay, unpacked,
  xr_logic, schemes, flow
- `flows/db-registry.md` — **(F69)** `db.script` global registry: per-object `storage[id]` blackboard +
  actor ref + category lookups; binders write, everything reads. tags: gameplay, unpacked, registry, flow
- `flows/condlist-library.md` — **(F67)** `xr_conditions`/`xr_effects`: bare global predicates/effects
  `(actor,npc,p)` namespaced per-script; evaluated by `xr_logic.pick_section_from_condlist`. tags:
  gameplay, unpacked, condlist, flow
- `flows/logic-ltx-switching.md` — **(F68)** logic-LTX authoring + parse: `parse_condlist`/
  `extract_conditions` token types + `cfg_get_switch_conditions` → `st.logic` (the switch list F66 walks).
  tags: gameplay, unpacked, logic-ltx, parsing, flow
- `flows/state-manager-core.md` — **(F70)** per-NPC animation/posture FSM: `set_state` records target →
  `state_manager:update` GOAP-converges sub-controllers to the `state_lib` entry. tags: gameplay, unpacked,
  state-mgr, animation, flow
- `flows/state-mgr-controllers.md` — **(F71)** the state_mgr sub-controllers (weapon/movement/mental/
  bodystate/direction/animstate/animation) + the animation IN→IDLE→OUT marker machine. tags: gameplay,
  unpacked, state-mgr, animation, flow
- `flows/smart-terrain-core.md` — **(F72)** smart_terrain class: spawn→job-assign→`setup_logic`→xr_logic
  bridge; `setup_gulag_and_logic_on_spawn` gulag-vs-`initialize_obj` fork. tags: gameplay, unpacked,
  smart-terrain, jobs, flow
- `flows/gulag-jobs.md` — **(F73)** gulag job board: `load_job` builds a smart's job arrays + in-memory
  logic ltx; `job_info_by_job_type_id` catalog; prior/precondition ranking. tags: gameplay, unpacked,
  gulag, jobs, flow
- `flows/scheme-walker-patrol.md` — **(F74)** the canonical scheme template (set_scheme/add_to_binder/
  evaluator/action driving move_mgr+state_mgr) via walker; patrol/remark deltas. tags: gameplay, unpacked,
  schemes, walker, flow
- `flows/scheme-combat.md` — **(F76)** combat schemes: `set_combat_type` condlist gate over the engine
  combat planner; combat_ignore enemy filter; danger/hit; zombied/camper/monolith styles. tags: gameplay,
  unpacked, combat, flow
- `flows/companion-system.md` — **(F82)** companions: recruit = `npcx_is_companion` info + configure_schemes;
  `xr_companion` follow/wait scheme; commands = `npcx_beh_*` info portions. tags: gameplay, unpacked,
  companion, flow
- `flows/scheme-camper-cover.md` — **(F75)** camper (hold/scan/sniper + combat handoff) & cover
  (`best_cover` vertex → pose); deltas on the walker template. tags: gameplay, unpacked, schemes, camper, flow
- `flows/scheme-animpoint.md` — **(F77)** animpoint: two-action reach→loop at a smartcover; predicate-
  filtered actions + campfire director/listener roles. tags: gameplay, unpacked, schemes, animpoint, flow
- `flows/scheme-sleeper-wounded-death.md` — **(F78)** sleeper/wounded/help-wounded/eat-medkit/death:
  wounded is the combat-gating hub (`wounded_fight`); death records killer. tags: gameplay, unpacked,
  schemes, wounded, flow
- `flows/scheme-meet-abuse-bribe.md` — **(F79)** meet (distance-tier greet/dialog, `process_npc_usability`),
  abuse (shove counter), bribe (`at_peace` faction ceasefire feeding combat_ignore). tags: gameplay,
  unpacked, schemes, meet, flow
- `flows/scheme-gather-corpse.md` — **(F80)** loot schemes: detect→claim→walk→search→transfer→un-claim;
  corpse `HasValuableLoot`, gather `Items[]`, box = `axr_npc_vs_box`. tags: gameplay, unpacked, loot, flow
- `flows/scheme-sensors-misc.md` — **(F81)** services not schemes: xr_sound/xr_hear (hooked from motivator),
  xr_zones (binder), xr_detector (actor-only), xr_weapon_jam (gamedata override), xr_reach_task. tags:
  gameplay, unpacked, sensors, flow
- `flows/se-stalker.md` — **(F83)** Lua server entity for a stalker NPC: `se_stalker`(+`se_trader`)
  subclasses `cse_alife_human_stalker`, overriding the ALife register/save/online-switch/death hooks to
  attach smart_terrain jobs + db registry + callbacks; `can_choose_alife_tasks(false)` hands control to
  Lua. tags: gameplay, unpacked, alife, server, save-load, flow
- `flows/se-others.md` — **(F84)** the rest of the `se_*` family: uniform F83-delta passthroughs +
  exceptions (se_monster=NPC clone, se_item=~19-class item catalog, se_actor=simulation target,
  se_restrictor=teleport-route map spots, se_smart_cover=loophole persistence, se_heli, se_artefact
  proximity gate); class→clsid registration map. tags: gameplay, unpacked, alife, server, save-load, flow
- `flows/scripted-squad.md` — **(F85)** the squad server class `sim_squad_scripted`: update→target-select
  (`get_script_target`→specific/generic)→move/reach/stay FSM→assign members to smart; `simulation_objects.
  evaluate_prior` target scoring; squad is also a sim target. **gamedata override** routes smart-assign
  through sim_board. tags: gameplay, unpacked, alife, squad, simulation, flow
- `flows/sim-board.md` — **(F86)** the `SIMBOARD` singleton: smart_terrain registry + `assign_squad_to_smart`
  (squad↔smart move + population bookkeeping) + `get_squad_target` (top-5 evaluate_prior pick, 50/50
  random/best) + `fill_start_position` (initial world population from simulation.ltx). tags: gameplay,
  unpacked, alife, squad, simulation, flow
- `flows/offline-combat.md` — **(F87)** Offline Combat Simulator: periodic `coordinator` (full vs
  on-smarts-only modes) → `validate_enemy` (offline/enemy/close/non-task gate) → `simulate_battle`
  (rank-based power, randomized damage, kill NPC/squad) → relations + news; crowded-levels-only,
  story/task squads excluded. tags: gameplay, unpacked, alife, squad, simulation, flow
- `flows/binder-architecture.md` — **(F88)** how Lua attaches to an online object: `object_binder`
  subclasses (actor_binder/motivator_binder/generic_object_binder) attached via `obj:bind_object` from a
  `init`/`actor_init`/`bind` hook; the engine-called lifecycle reinit→net_spawn→update→save→net_destroy;
  `class_registrator` clsid→client+server. **bind_monster=gamedata override.** tags: gameplay, unpacked,
  lifecycle, callbacks, flow
- `flows/world-object-binders.md` — **(F89)** the `bind_*` world-object binders: two sub-patterns —
  scheme-running (`generic_physics_binder` runs xr_logic schemes + box loot; the binder `ph_*` schemes
  attach to) vs manager/registry (campfire/camp/faction/light wrap an engine sub-object or manager). All
  F88 deltas. tags: gameplay, unpacked, lifecycle, physics, flow
- `flows/anomaly-field-binders.md` — **(F90)** `bind_anomaly_zone` = the **artefact-spawn economy** (layers,
  chance×surge, `arty_collection`, artefact_ways) — the Lua path that spawns artefacts (resolves F27);
  `bind_anomaly_field` = per-field binder (`fields_by_names`, set_enable) + the **dynamic-anomalies**
  relocation system + the activation veto. tags: gameplay, unpacked, lifecycle, zone, artefact, flow
- `flows/restrictors-passive-sr.md` — **(F91)** passive `sr_*` restrictor schemes: the F66 scheme template
  (`sr_idle`) on a space restrictor + the passive-effect catalog (silence/no_weapon/timer/postprocess/light/
  particle/psy_antenna/deimos), driving the F31 pp/cam-effector + sound APIs. tags: gameplay, unpacked,
  schemes, zone, flow
- `flows/restrictors-active-sr.md` — **(F92)** active `sr_*` restrictors that fire events on actor entry:
  `sr_teleport` (weighted-random actor teleport), `sr_crow_spawner`/`sr_monster` (alife_create spawns),
  `sr_cutscene` (camera `.anm` takeover), `sr_camp` (`CCampManager` storytelling via bind_camp). tags:
  gameplay, unpacked, schemes, zone, flow
- `flows/physics-object-schemes.md` — **(F93)** `ph_*` physics-prop schemes on stype_item (run by F89's
  binder): door/button/code/idle/hit/on_hit/on_death/oscillate/sound + scripted car & mounted minigun
  (engine CCar) + the `set_fastcall` high-rate channel; `phantom_manager` spawn service. tags: gameplay,
  unpacked, schemes, physics, flow
- `flows/monster-schemes.md` — **(F94)** `mob_*` monster schemes (stype_mobile, run by the gamedata
  bind_monster override) + trader schemes (stype_trader): home leash/walker patrol/remark/jump + generic
  death/trade listeners; capture-the-action-queue vs leash-the-FSM control model; dead (mob_camp) and
  dormant (mob_combat) paths. tags: gameplay, unpacked, schemes, ai, flow
- `flows/helicopter-ai.md` — **(F95)** the heli stack: heli_binder (health/flame/death, heli_enemy_flag)
  → always-on heli_combat manager (flyby/round/search; night+surge off) → heli_move story scheme
  (stype_heli) vs heli_alife faction-sim layer (squad-style target FSM on simulation_objects); Lagrange
  flight planner. tags: gameplay, unpacked, ai, alife, flow
- `flows/task-system-lua.md` — **(F97)** the quest tier over F38's CGameTask shell: tm_*.ltx config →
  CGeneralTask with live functor-recomputed title/descr/target + condlist outcomes → engine
  complete/fail functor poll → task_callback effects/rewards/repeat-cooldown; marshal persistence.
  tags: gameplay, unpacked, task, quest, flow
- `flows/dynamic-task-types.md` — **(F98)** the ~22 `tasks_*` quest-type plugins: namespace injection
  into task_functor/xr_conditions/xr_effects (resolved by F97+F67), `target` field as functor argument,
  stage stepping, script-callback simulations with own m_data persistence. tags: gameplay, unpacked,
  task, quest, flow
- `flows/dialog-system-lua.md` — **(F99)** the Lua/data dialog tier over F18: attachment
  (character_dialogs include), XML phrase graphs naming `dialogs.*` functors, script-built trees
  (`<init_func>` → `dialog_manager.add_dialog`), and the dynamic-chatter priority engine over
  `dialog_manager.ltx`. tags: gameplay, unpacked, dialog, npc, flow
- `flows/characters-info-encyclopedia.md` — **(F100)** NPC identity pick (npc_profile →
  specific_character template selection w/ rank/rep deltas + used-id exclusion), info-portion
  registration + actor info-switch Lua, and the PDA encyclopedia plugin (interaction-driven
  unlocks, mode-gated articles). tags: gameplay, unpacked, character, info-portion, pda, flow
- `flows/trade-economy.md` — **(F101)** the Lua trade tier: per-NPC trade profiles (condlist re-pick
  on talk, eco-hour restock via engine CPurchaseList), blacklist+multiplier buy/sell lists from
  presets, the Lua UI takeover of engine StartTrade (price = cost×mult×discount×cond^exponent, pure
  Lua transaction), and the axr NPC↔NPC gulag-job economy. tags: gameplay, unpacked, trade, economy,
  money, flow
- `flows/item-behaviors-lua.md` — **(F102)** the `item_*` plugin family + itms_manager hub over the
  `_g.Parse_ITM` ini-driven type registry: use veto/specials/drag-combos/ItemProcessor, repair/
  disassembly/cooking/recipes, the item_device binder, deployables, and the item_weapon gamedata
  fork. tags: gameplay, unpacked, items, crafting, flow
- `flows/actor-survival-status.md` — **(F103)** the Lua survival layer over the engine condition
  tick: sleep/thirst difficulty-toggled meters, arszi_psy psy-health (hijacks telepatic hits;
  engine bar display-only), actor_status HUD indicator strip + safe-cover feed, and the
  actor_effects LTX-driven item-use animation scheduler. tags: gameplay, unpacked, actor, survival,
  sleep, thirst, psy, hud, flow
- `flows/actor-menu-inventory-ui.md` — **(F104)** the Lua inventory window that replaced the
  engine ActorMenu: functor veto seam, actor_menu mode mirror + HUD message API, bags/context
  menu (`useN_*` LTX custom rows), drag-drop, loot/repair modes, utils_ui cell framework, the
  monkey-patch surface. tags: gameplay, unpacked, ui, inventory, actor-menu, flow
- `flows/warfare-metagame.md` — **(F105)** the optional faction-war mode: `IsWarfare()` root,
  per-squad `__lock` suppression of stock AI, smart capture/defense/invasion/patrol tick,
  faction economy + resurgence, player commander (money/influence, possess-leader), fog-of-war
  map spots. tags: gameplay, unpacked, warfare, faction-war, simulation, flow
- `flows/game-modes.md` — **(F106)** opt-in rule modifiers + difficulty: new-game flag seeding,
  the `gdiff_`/`econ_` factor families, game_setup's dynamic world-item loop + map fixups,
  agony/timer save vetoes, ironman cross-save death accounting, azazel body-swap death.
  tags: gameplay, unpacked, gamemodes, difficulty, ironman, azazel, flow
- `flows/game-services.md` — **(F107)** the `game_*` services: game_statistics rank/reputation
  engine (counters→rank/rept, band-crossing tips, NPC `m_data` tallies, visited tracking, MARSHAL
  persistence), game_achievements detector (pull-based, polled only by the PDA Ranking tab;
  rewards distributed via `has_achievement()` consumers; dead `bookworm_food`), autosave, PDA-map
  fast/backpack travel, vestigial game_registrator. tags: gameplay, unpacked, achievements,
  statistics, rank, autosave, fast-travel, flow
- `flows/faction-relations-lua.md` — **(F108)** the Lua faction-relations layer (DoctorX Dynamic
  Faction Relations): the 5-component goodwill model + the getter/setter API ~35 scripts call
  (`is_factions_enemies`/squad goodwill/gulag relation), community-matrix seeding & per-pair
  persistence, and the dynamic-relations event engine (kills drift faction standings, warfare
  override). `game_relations.ltx` is the shared data tier (bands, matrices, action points).
  tags: gameplay, unpacked, relations, goodwill, community, faction, flow
- `flows/gameplay-ui-screens.md` — **(F109)** the Lua UI screens & HUD widgets outside the
  inventory: the SP `ui_main_menu` (save-veto + `main_menu_on_*` callback seams), the HUD
  custom-static widgets (`ui_enemy_health`; dormant Lua `ui_dosimeter`; dead `ui_extra`), the
  `ui_dyn_msg_box` reusable toolkit (msg-box/multi-choice/context-menu/context-props), the Pripyat
  `ui_freeplay_dialog`, and vestigial `ui_registrator`. tags: gameplay, unpacked, ui, hud,
  main-menu, flow
- `flows/pda-tabs.md` — **(F110)** the PDA hub (`pda.script` = engine `set_active_subdialog` tab
  dispatch + Top-100 ranking board + stats panel + map-spot bridge + contacts capture) and the
  script tabs: contacts, dead-NPC PDA reader, Zone-FM radio + music player, faction-relations
  matrix + companion roster, warfare dominance board; plus the warfare commander map plugins
  (`pda_actor`/`pda_flagger`). tags: gameplay, unpacked, ui, pda, ranking, radio, flow
- `flows/ui-debug-tools.md` — **(F111)** the `DEV_DEBUG`-gated dev tooling: the `ui_debug_launcher`
  ("Anomalous Debugger", F7) hosting spawners/target-actions/editors with an `inject` extensibility
  API, the `xrs_debug_tools` frame-driven HUD inspector (squad/planner/heli/monster/zone panels via
  custom-statics), the `debug_cmd_list` command registry, the dormant legacy `ui_debug_main`, and the
  gamedata delta (`zz_ui_debug_inputs` input-history, `imgui_lua_debug` remote debugger,
  `options_modded_exes_debug` options page). tags: gameplay, unpacked, gamedata, ui, debug, hud, flow
- `flows/ui-news-tips.md` — **(F112)** the PDA notification layer: `news_manager` (the universal
  tip/showcase dispatcher ~50 scripts call — `send_tip`/`send_task`/`relocate_item`/`relocate_money`
  over engine `give_game_news`) and the self-scheduling **Dynamic News** generator
  (`dynamic_news_manager` + `dynamic_news_helper`): per-faction channel queues, six TimeEvent ticks,
  death/loot callbacks, the `NewsToggle` gate (paused/underground/surge-erases-queue/PDA-charge +
  comm-centre transitions), and the helper's landmark/time-phrasing/faction pickers. tags: gameplay,
  unpacked, ui, news, pda, dynamic-news, flow
- `flows/axr-behaviour-addons.md` — **(F113)** the `axr_*` behaviour layer: Alundaio's scripted GOAP
  schemes (the `axr_beh` companion escort/patrol/cover brain + `axr_beh_patches` customization hook;
  the auto-active combat/utility schemes `axr_fight_from_cover`/`axr_stalker_panic`/`axr_npc_vs_heli`/
  `axr_radio_in_heli`/`axr_npc_vs_box`/`axr_turn_on_campfire`) + the `axr_keybind` scriptable key
  dispatcher. All ride the F37 scheme interface. tags: gameplay, unpacked, axr, scheme, goap, ai, flow
- `flows/axr-managers.md` — **(F114)** the three high-level `axr_*` managers: `axr_task_manager` (the
  DRX-storyline task generator on the F97 core — honcho task menus, the `bounties_by_id`/`hostages_by_id`
  registries, the 3-hr/75%-gated per-NPC task picker over `CFG_CACHE`); the `axr_companions` **manager
  layer** (roster join/dismiss, the five-axis command system that writes the `npcx_beh_*` flags,
  special/story squads + escort spawning, inventory assignment, command wheel, healing/unstuck timers —
  deepens F82's scheme-side coverage); `axr_trade_manager` routed to F101. tags: gameplay, unpacked, axr,
  manager, task, companion, trade, flow
- `flows/utils-libraries.md` — **(F115)** the six `utils_*` helper libraries (the cross-cutting layer,
  ~500 functions): `utils_obj` (object/squad/navigation + the `validate`/`lmove`/`try_go_cover` movement
  primitives combat schemes reuse), `utils_item` (item/weapon/outfit + `get_item_cost`/`get_item_trade_status`
  + the `on_get_item_cost` callback), `utils_data` (math/vector/CTime/path + the `parse_condlist`/`parse_*`
  config primitives + the `cfg_file` read/write INI class), `utils_xml` (UI XML load + resolution scaling +
  `XmlParser`), `utils_ui` (the Lua inventory-cell engine `UICellItem`/`UICellContainer` behind F104),
  `utils_stpk` (server-object packet read/write — `get_/set_*_data` + `parse_/fill_*_properties_packet`).
  tags: gameplay, unpacked, utils, helper, library, flow
- `flows/sound-music.md` — **(F116)** the audio layer: two music systems (engine ambient from
  `game_levels_music.ltx` vs `xrs_dyn_music` Lua combat music that ducks it via `snd_volume_music`; combat
  themes hardcoded, only stalker enemies trigger them); the sound playback core (`xr_sound` scheme-facing
  player + `get_sound_object`/`get_safe_sound_object` theme selection + path cache, `sound_theme` registry,
  `aaa_sound_object_patch` global `sound_object` lifetime guard); scheme-driven sounds (`sound_manager`
  campfire stories, `sound_ambient` level ambient/wind/rain, `ph_sound`, `xr_zones_sound`, `mob_sound`);
  `level_weathers` weather sibling (characterized). tags: gameplay, unpacked, sound, music, audio, flow
- `flows/treasures-secrets.md` — **(F117)** stashes & loot caches: the procedural `treasure_manager`
  (fills every `INVBOX` on new game to a kind×tier value budget from `treasure_manager.ltx`, economy-type
  filtered, lazy-spawns on open, rare stashes gate toolkit rewards on task count, `caches[id]` state
  machine, half the boxes filled); predefined authored secrets (`secrets.ltx` + per-level includes,
  `item=count,prob`); quest stashes (`tasks_stash` DRX effects); the `item_radio` RF stash-hint coupling;
  treasure map spots + `send_treasure` news. tags: gameplay, unpacked, treasure, stash, secret, loot, flow
- `flows/creatures-actor-configs.md` — **(F118)** the `configs/creatures/` data tier: the include +
  `[child]:parent` inheritance chain (`monster_base` in `defines.ltx` → `m_<species>_<variant>` →
  per-level `spawn_sections_*` → spawned object); per-monster `m_*.ltx` anatomy (ALife stats, AI
  `class`/`cform`, `ef_*_type`, bones, fire/psy/rad influences, physics boxes, **inline**
  immunity/damage/protection/vision sub-sections); `actor.ltx` (movement coefs, mass, attachables, NV);
  shared profiles `immunities.ltx` (named stalker resistance presets) + `damages.ltx` (bone-damage
  profiles); `game_relations.ltx` → F40/F108; `helicopter.ltx` → F95. tags: gameplay, config, ltx,
  creatures, monster, actor, data, flow
- `flows/zones-anomaly-configs.md` — **(F119)** the `configs/zones/` data tier: `zones.ltx` include
  manifest → `zone_base` inheritance root (carries `script_binding = bind_anomaly_field.bind`, the F90
  artefact-binder seam) → `class = ZS_*` section→engine-class mapping (CustomZone family, F27) → thin
  `weak/average/strong` tiers; parameter groups (strength/hit-type/blowout-FSM/particles/light/impulse/
  teleport/wind/postprocess); field (passive DoT) vs mine (blowout) families + monolith/campfire oddballs;
  config artefact fields ≠ live spawning (F90 owns it); `sr_teleport_sections.ltx` = the level-changer
  table for the `sr_teleport` active restrictor (F92), NOT an anomaly. tags: gameplay, config, ltx, zones,
  anomaly, teleport, data, flow
- `flows/items-weapons-outfits-configs.md` — **(F120)** the `configs/items/` data tier (the biggest
  cluster, ~134 weapon files): `item_base.ltx` manifest → family roots; the **wildcard auto-include**
  modding contract (`#include "w_*.ltx"`/`"o_*.ltx"`/`"items_*.ltx"` — drop a new prefixed file, never edit
  shipped ones); weapons `w_*` (root + `_hud` + cosmetic/attach-state variant subsections; ballistics/ammo/
  addon/condition/recoil/upgrade clusters → F22), outfits `o_*` (`outfit_base`, protection block + immunities
  sect + `use1_functor` disguise hook → F23/F102), items `items_*` (consumable/misc categories → F21/F24/
  F25), ammo `k_*` coeffs, addons/NV scope presets (F22), upgrade trees (F26); shared base parents →
  `defines.ltx` (F122); trade/ → F101, settings/ → F117/F102/F123. tags: gameplay, config, ltx, items,
  weapons, outfits, ammo, data, flow
- `flows/per-smart-logic-configs.md` — **(F121)** the `configs/scripts/` data tier (~2016 files): the
  **custom_data / logic-LTX** content — **loaded by name** (not `#include`d), org'd as top-level reusable
  templates + one subdir per level. Two file roles: object logic (single `[logic] active=` FSM, e.g. `door`/
  `sr_sleep`/`sr_teleport_logic` → F68 format + F67 condlists) vs gulag/smart logic (many `[logic@job]` with
  `prior`/`suitable`/`monster_job` = the F73 job catalog, e.g. walker/camper/surge-hide/animpoint-camp/
  mob_home jobs). `smart/<smart>.ltx` = smart_terrain respawn def (`[smart_terrain]`+`respawn@`+`spawn_squads`
  → F72/F86), distinct from `*_smart_logic.ltx`. `camp.ltx` = campfire storyteller registry (F116, not an
  FSM). DLTX `@`-deltas apply (1 gamedata override). tags: gameplay, config, ltx, logic, custom-data, gulag,
  data, flow
- `flows/plugins-tweaks-presets-configs.md` — **(F122)** the tuning-knob & shared-foundation tier:
  **`defines.ltx`** (the inheritance ROOT `#include`d first — `hud_base`, `identity_immunities`/`booster`
  consumable base, `weapon_probability`+`default_weapon_params` [F120 weapon parents], `monster_base` [F118
  creature root]); **`presets/`** (video/economy/gameplay-diff/warfare option-key bundles → F17/F106/F105);
  **`axr_options.ltx` = empty stub** (real options = F17 `options_modded_exes_*`); **`plugins/`** (26 data
  files) + **`ai_tweaks/`** (18 AI tuning tables) routed to owning system flows. The constants tier vs the
  content tiers. tags: gameplay, config, ltx, plugins, tweaks, presets, defines, data, flow
- `flows/spawns-alife-graphs-configs.md` — **(F123)** the world-instance & ALife tier: **`spawns/all.spawn`**
  (the BINARY authored-world file — every `cse_alife_*` static object placement + `way_` patrol paths; SDK/
  ACDC-built, new-game-only, save diverges after → F16/F41); **`alife.ltx`** (simulator tuning:
  schedule/objects_per_update/time_factor/switch_distance/auto_switch/start_game_callback → F16);
  **`game_graphs.ltx`** (graph-point name catalog + level membership → F86/F85); **`game_story_ids.ltx`**
  (numeric story-ID → object name, the stable script handle); **`game_maps_single.ltx`** (level registry +
  PDA world-map layout + per-level music → F39/F116); `patrol_paths.ltx`/`dynamic_spawn.ltx` = vestigial
  stubs (paths live in all.spawn). **Closes the config tier: templates F118–F122 × placements (all.spawn) ×
  logic F121 = the world.** tags: gameplay, config, ltx, spawns, alife, graph, story-id, data, flow
- `flows/engine-boot-frame.md` — WinMain→engine/device init→message_loop/on_idle frame (seqFrame/
  seqRender + MT GameThread scheduler). The outermost loop. tags: engine, boot, frame, device, flow
- `flows/render-frame.md` — R4/DX11 deferred frame: cull→G-buffer→shadows→light accum→combine/post
  (SSFX)→forward→present. tags: renderer, r4, dx11, deferred, flow
- `flows/save-load.md` — `.sav` serialize/restore: ALife registry (Spawn+Update packets) + online
  objects flushed via ClientSave + Lua binder save/load. tags: alife, save, load, serialization, flow
- `flows/alife-online-offline.md` — ALife online↔offline switch (crash-prone): proximity + hysteresis
  band, add_online/remove_online ordering guards. tags: alife, online, offline, switch, crash, flow
- `flows/options-runtime.md` — option lifecycle: builder DSL → tree → DXML force-load injection →
  UIOptions render → On_Accept → console var / axr_options.ltx. tags: options, ui, dxml, console, flow
- `flows/dialog-trade.md` — NPC dialog phrase graph (Lua precondition/action + info gates) + trade
  item/money transfer with condition/relation/script-discount price. tags: dialog, trade, npc, flow
- `flows/hud-ui-load.md` — UI from XML: CXml::Load → DXML `_G.COnXmlRead` → CUIXmlInit recursive
  widget tree (AttachChild) → per-frame Update/Draw. tags: ui, hud, xml, cuiwindow, flow
- `flows/ui-scaling.md` — the 1024x768 virtual canvas: anisotropic X/Y stretch at draw time, `kx`
  aspect correction, the dynamic font-atlas ladder (`texture_h<N>`; text still doesn't scale
  continuously), frame-line cap scaling + crisp `border_*` bands, `_16`/`_21` XML variants, cursor
  space, clip stacks. tags: ui, hud, renderer, config, lua, flow
- `flows/animation-callbacks.md` — animation timeline → gameplay: motion marks, animation end
  (drives state machine), footsteps; main vs worker thread. tags: animation, motion-mark, footstep, flow
- `flows/weapon-hierarchy-addons.md` — weapon type tree (CWeapon→Magazined→pistol/shotgun/WGrenade…)
  + scope/silencer/GL attach/detach, InitAddons koeffs, ammo boxes & magazine. tags: weapons, addons,
  ammo, inheritance, flow
- `flows/outfit-helmet.md` — body armor + helmet: equip/visual swap, protection (×condition) vs
  immunity-as-wear, HitThroughArmor, outfit→helmet damage seam, passive restore. tags: outfit, armor,
  helmet, protection, condition, flow
- `flows/detector-artefact.md` — detector finds artefacts via Feel::Touch (rank+section gate) → per-tier
  UI; artefact belt restore/immunity (shared actor tick) + spawn-zone activation. tags: detector,
  artefact, feel-touch, belt, activation, flow
- `flows/upgrade-system.md` — item upgrades: manager tree (Root/Group/Upgrade/Property) → upgrade_install
  → install_upgrade_impl applying ltx deltas (process_if_exists additive vs set). tags: upgrade, inventory,
  ltx, money, flow
- `flows/anomaly-zones.md` — zones detect occupants via Feel::Touch → idle/awaking/blowout/accumulate
  cycle → Affect/CreateHit (m_eHitTypeBlowout, distance-attenuated); campfire/gravi/radioactive. tags:
  zone, anomaly, feel-touch, blowout, hit, flow
- `flows/actor-condition.md` — survival tick: delta-accumulator UpdateCondition (health/power/satiety/
  radiation/bleeding/psy), satiety-driven regen, boosters, last-chance health. tags: actor, condition,
  survival, satiety, booster, flow
- `flows/relations-info.md` — info-portions (knowledge flags; no on_info callback — poll has_info) +
  relations (GetAttitude 5-component sum → friend/neutral/enemy thresholds). tags: relations, goodwill,
  reputation, info-portion, flow
- `flows/tasks.md` — player quests: CGameTask (monolithic, no objectives) → UpdateState polled gates
  (timeout/fail/complete via HasInfo + functors); eTaskStateChange; ALife smart-terrain task is separate.
  tags: task, quest, info-portion, pda, flow
- `flows/item-base.md` — item class tree (CInventoryItem mixin → CInventoryItemObject diamond →
  CHudItemObject) + OnH_* ownership transitions (drop=physics on/pickup=off) + bone attach + condition.
  tags: item, inventory, physics, attach, condition, flow
- `flows/camera-pp-effectors.md` — two effector stacks: camera (shake/bob/recoil, sequential mutation)
  + post-process (tint/blur, additive); level.add_cam_effector/add_pp_effector. tags: camera, effector,
  postprocess, flow
- `flows/cscript-game-object.md` — Lua `game_object` wrapper: split registration, smart_cast delegation,
  per-object set_callback seam (m_callbacks). tags: lua, luabind, game_object, callback, flow
- `flows/consumables-explosives.md` — eatable use (influence+booster, decrement uses) + grenade
  throw→cook→Explode (blast wave GE_HIT + fragments via BulletManager). tags: consumable, explosive,
  grenade, flow
- `flows/pda-map-news.md` — map-location registry (spots/pointers track objects), PDA Feel::Touch
  contacts, news queue (give_game_news). tags: pda, map, news, contacts, flow
- `flows/alife-services.md` — ALife sim services: game time (factor×elapsed), surge = C++ spawn-only
  STUB / Lua surge_manager does emission, location/interaction stubs. tags: alife, time, surge, flow
- `flows/physics-objects-ragdoll.md` — game-side physics shells (box/skeleton) + death ragdoll
  (animated→physics on Die: CreateShell/esDead, friction evolves); HangingLamp. tags: physics, ragdoll,
  death, flow
- `flows/vehicles-holder.md` — CHolderCustom mount seam (input/camera/physics handoff), CCar physics
  vehicle, CHelicopter is NOT a holder (AI aircraft). tags: vehicle, car, holder, mount, flow
- `flows/monster-ai-control.md` — monster AI: FSM (StateMan) + pluggable control elements (capture
  anim/move/dir/path + ability jump/charge), NOT GOAP. tags: ai, monster, control, fsm, flow
- `flows/stalker-execution.md` — GOAP action → execution: movement (path/detail), sight (aim gates
  fire), animation parts, NPC world-weapon fire (skill dispersion/burst). tags: ai, stalker, movement,
  sight, fire, flow
- `flows/smart-cover.md` — designer smart covers + loopholes (fov + enter/idle/fire animations) vs
  auto cover-grid; loophole GOAP picks action. tags: ai, smart-cover, loophole, cover, flow
- `flows/script-npc-actions.md` — Lua takes NPC control (GetScriptControl?ProcessScripts:Think fork) →
  action queue drives movement/sight/anim; C++ side of xr_logic. tags: ai, script-control, xr-logic, flow
- `flows/agent-squad-combat.md` — group combat coordinator: pools members' enemy sightings + assigns
  enemies/cover/corpses/grenades (mask-indexed); danger is per-NPC. tags: ai, agent-manager, squad,
  combat, flow
- `flows/special-actors.md` — trader (dialog-only NPC), crow (ambient bird+ragdoll), phantom (psy
  apparition state machine) — none use GOAP/monster-FSM. tags: ai, trader, crow, phantom, flow
- `flows/level-runtime.md` — CLevel per-frame: object collection tick, network spawn/events, map/task
  update, input routing, load; resolves "rest of Level.cpp". tags: level, object-collection, frame, flow
- `flows/wallmarks-particles.md` — impact decals + gameplay particles via material-pair lookup
  (GetMaterialPair → wallmark+particle+sound); explosion wallmarks; Lua APIs. tags: wallmark, particles,
  material, flow
- `flows/renderer-resources-shaders.md` — CResourceManager find-or-create dedup tree: blender→6
  ShaderElements→Shader; HLSL load/compile via getShaderPath+shader_compile; texture/state/constant
  dedup; creationGuard. tags: renderer, shaders, blender, dx11, flow
- `flows/renderer-scene-graph.md` — CDSGraphManager dsgraph: traverse→capture→insert (SSA/flag-route)→
  sort-key render queue→state-batched flush; per-view GMBase/GMLight/GMCascade (main + shadow queues).
  tags: renderer, r4, dx11, deferred, flow
- `flows/renderer-post-process.md` — R4+SSFX post chain in phase_combine: ordered SSAO→combine→sky→SSR/
  water→forward→volumetric→bloom→…→AA→final (each o.ssfx_*/ps_*-gated). tags: renderer, r4, dx11,
  shaders, flow
- `flows/renderer-blender-compile.md` — blender→passes: CLSID factory (blender_create) + name lookup;
  CBlender_Compile recorder (r_Pass/r_End/r_dx10Texture/Sampler) turning Compile/iElement into deduped
  SPasses; legacy CLSIDs return 0 on R4. tags: renderer, shaders, blender, flow
- `flows/renderer-visibility.md` — cull input: HOM (CPU software occluder depth) + sector/portal
  traverse + GPU occlusion queries; disabled/timeout ⇒ visible; feeds the scene-graph. tags: renderer,
  r4, dx11, flow
- `flows/renderer-backend.md` — CBackend/global RCache: state-dedup set_* (skip-if-unchanged), deferred
  constants/RT flushed inside Render→DrawIndexed, R_DStreams dynamic VB/IB (discard/no-overwrite). tags:
  renderer, r4, dx11, flow
- `flows/renderer-visual-types.md` — dxRender_Visual tree (Fvisual/FProgressive/FHierrarhy/CSkeletonX/
  FTree/FLOD) + CModelPool two-tier instancing (load-once reference + per-object instance, shared
  geometry, pool recycle); Render(LOD). tags: renderer, r4, dx11, flow
- `flows/renderer-skeleton.md` — kinematics: CKinematics(:FHierrarhyVisual)/CKinematicsAnimated/
  CSkeletonX; CalculateBones→Bone_Calculate recursion + bone-callback=physics override; LL_PlayCycle/
  UpdateTracks blends; HW skinning upload. tags: renderer, animation, flow
- `flows/renderer-lighting.md` — dynamic point/spot lights: per-light GMLight dsgraph collects casters →
  SMAP atlas → phase_smap_spot depth → accum_point/spot into accumulator; LP_normal/LP_pending occlusion
  split; sun=F55. tags: renderer, r4, dx11, deferred, flow
- `flows/renderer-sun-shadows.md` — sun CSM: 3 cascades (per-cascade GMCascade dsgraph) →
  render_sun_cascade → phase_smap_direct depth → accum_direct_cascade into shared accumulator; pairs
  F54. tags: renderer, r4, dx11, deferred, flow
- `flows/renderer-wallmark-engine.md` — decal engine (engine side of F44): static wallmarks clipped to
  CDB tris + aged in CWallmarksEngine; skeleton wallmarks skinned via CKinematics::RenderWallmark /
  sorted Wmark path. tags: renderer, r4, dx11, flow
- `flows/renderer-hw-device.md` — CHW: D3D11 device/context/swapchain + base RTV/DSV; CreateDevice/Reset/
  UpdateViews; present in dxRenderDeviceRender::End; CHWCaps gates features. tags: renderer, dx11, build,
  flow
- `flows/renderer-factory-bridges.md` — engine↔Layers seam: global Render/RenderFactory/UIRender wiring;
  dxRenderFactory creates dx* impls; dxRenderDeviceRender device glue (present in End); dxUIRender/font/
  console/debug → RCache. tags: renderer, ui, flow
- `flows/renderer-particles.md` — particle system IS-A dxRender_Visual; PAPI(xrParticles) sim + engine
  anim/collision; camera/path billboards via dynamic VB; scene-graph B2F sort bypassed. tags: renderer,
  r4, dx11, flow
- `flows/renderer-environment.md` — sky/clouds/rain/thunder/flares + output gamma: game owns CEnvironment
  state, Layers dx*Render draws (called from phase_combine); gamma=output ramp not phase_lut. tags:
  renderer, r4, dx11, flow
- `flows/renderer-detail-grass.md` — CDetailManager: camera-centered slot cache + terrain-trace
  decompress + VS-instanced wind-swayed grass; renders in main + shadow passes (hw_Render(light*)). tags:
  renderer, r4, dx11, flow
- `flows/renderer-r1-path.md` — R1 reference/contrast: forward + baked light-maps + projector/blob
  shadows (NOT active build, which is R4 deferred); getShaderPath r1\ vs R4 r3\. tags: renderer, r1, dx9,
  build, flow
- `flows/renderer-r2-r3-paths.md` — R2(DX9)/R3(DX10) deferred generations: same shape as R4, deltas-vs-R4
  (R4 adds SSFX/DX11/HDR10); all report GENERATION_R2; compile-time-selected, OOS. tags: renderer, r2,
  r3, build, flow
- `flows/renderer-3dfluid.md` — DX10 GPU Navier-Stokes volumetric smoke (grid sim + ray-march); manager
  initialized in r4.cpp but per-volume RenderFluid commented out → dormant. tags: renderer, dx10, flow

### engine/ — C++ engine (`src/`)
- `engine/_overview.md` — module map, dependencies, build system, branch variants. *Start here for
  engine work.* tags: engine, cpp, build
- `engine/xrGame.md` — gameplay: actor, weapons, inventory, AI, in-game UI, Lua binding/exports.
  tags: engine, cpp, actor, weapons, ai, ui, luabind
- `engine/xrCore.md` — foundation: shared_str, smart_cast, FS, LTX/ini + DLTX parsing.
  tags: engine, cpp, dltx, ltx
- `engine/renderer.md` — `Layers/`: R1–R4 paths, DX9/10/11 backends, blenders, shader pipeline,
  SPOM. tags: engine, renderer, shaders, r4, dx11, spom
- `engine/physics-collision.md` — `xrPhysics` + `xrCDB` (CFORM). tags: engine, physics, collision
- `engine/engine-core.md` — `xrEngine`: device, main loop, scheduler, console, input.
  tags: engine, scheduler
- `engine/server-entities.md` — `xrServerEntities` + `xrNetServer`: ALife server objects, net,
  script engine host. tags: engine, alife, server, net, luabind
- `engine/sound.md` — `xrSound`, doppler. tags: engine, sound
- `engine/subsystems-misc.md` — `xrParticles`, `xrCPU_Pipe`, `xrXMLParser`, `compressor/`,
  `3rd party/`. tags: engine, cpp

### modlayer/ — mod overlay (`gamedata/`)
- `modlayer/_overview.md` — gamedata vs db/mods vs unpacked, load order, what this repo ships.
  *Start here for mod work.* tags: modlayer, config
- `modlayer/scripting.md` — `.script` model, globals, callbacks, axr_main, class registration, hot
  reload. tags: modlayer, lua, callbacks, hot-reload
- `modlayer/config-ltx.md` — LTX format, DLTX operators, `mod_system_*`, extra params.
  tags: modlayer, config, dltx, ltx
- `modlayer/dxml.md` — DXML XML manipulation; `modxml_*`, `on_xml_read`, `xml_obj`. tags: modlayer,
  lua, dxml
- `modlayer/unlocalizer.md` — local→global var promotion; `configs/unlocalizers/`. tags: modlayer,
  unlocalizer
- `modlayer/options-ui.md` — options framework + imgui; `ui_options` quirks. tags: modlayer, options,
  ui, imgui
- `modlayer/assets.md` — shaders, textures, materials, ui xml, text/localization, levels.
  tags: modlayer, assets, shaders, localization
- `modlayer/gameplay/_overview.md` — **(M1)** gameplay-corpus (`unpacked/`) L1 hub: boot→db→scheme
  engine→schemes→jobs/squads→binders→systems; routes to F65–F123 (`ROADMAP-gameplay.md`). tags:
  modlayer, gameplay, unpacked, map
- `modlayer/gameplay/server-entities-lua.md` — **(M6)** the `se_*` Lua server-entity family map: thin
  subclasses of `cse_alife_*` adding register/save/online-switch/death gameplay bookkeeping; routes
  F83/F84. tags: modlayer, gameplay, unpacked, alife, server, map
- `modlayer/gameplay/squads-simulation.md` — **(M5)** the squad-simulation map: `sim_squad_scripted`
  (squad class) + `simulation_objects` (target scorer) + `sim_board` + `sim_offline_combat`; the macro
  "where a group goes" layer above smart_terrain/gulag. routes F85–F87. tags: modlayer, gameplay,
  unpacked, alife, squad, simulation, map
- `modlayer/gameplay/binders.md` — **(M7)** the `bind_*` binder map: `object_binder` subclasses attached to
  online objects (actor/stalker/monster + world/anomaly/restrictor/physics object binders); the client-side
  counterpart to the `se_*` server tier. routes F88–F93. tags: modlayer, gameplay, unpacked, lifecycle, map
- `modlayer/gameplay/restrictors-sr.md` — **(M10)** the `sr_*` restrictor-scheme map: F66 scheme pattern on
  space restrictors; passive (effects/hazards, F91) vs active (camp/cutscene/teleport/monster/crow, F92).
  tags: modlayer, gameplay, unpacked, schemes, zone, map
- `modlayer/gameplay/monster-schemes.md` — **(M9)** the `mob_*` monster-scheme family map:
  scheme-vs-dead-vs-helper classification, stype_mobile/stype_trader routing; routes F94. tags:
  modlayer, gameplay, unpacked, schemes, ai, map
- `modlayer/gameplay/physics-schemes-ph.md` — **(M11)** the `ph_*` physics-scheme family map:
  scheme-vs-unregistered-vs-binder classification; routes F93. tags: modlayer, gameplay, unpacked,
  schemes, physics, map
- `modlayer/gameplay/vehicles-heli.md` — **(M12)** vehicles Lua-tier map: heli stack (F95) + the
  merged car binding (bind_car/se_car → F93/F84/F29). tags: modlayer, gameplay, unpacked, ai, map
- `modlayer/gameplay/tasks-lua.md` — **(M13)** Lua task family map: core (F97) + dynamic types (F98)
  + AXR extender (F114 ✅ `flows/axr-managers.md`) + tm_*.ltx config tier. tags: modlayer, gameplay, unpacked, task, quest, map
- `modlayer/gameplay/dialogs-info.md` — **(M14)** dialogs & character-info map: dialog_manager +
  dialogs.script + story packs + XML graphs/attachment + info-portion data tier; routes F99 + F100 —
  domain complete. tags: modlayer, gameplay, unpacked, dialog, info-portion, map
- `modlayer/gameplay/trade-economy.md` — **(M15)** trade & economy map: trade profiles + presets
  data tier, Lua trade UI, NPC↔NPC gulag-job economy, eco difficulty knobs; routes F101 (+F102/F114
  ⬜); trader schemes → F94, engine tier → F18. tags: modlayer, gameplay, unpacked, trade, economy,
  map
- `modlayer/gameplay/items-lua.md` — **(M16)** item-behavior family map: type registry + hub +
  economy/equipment/deployable plugin clusters + `configs/items/settings/` data tier; routes F102.
  tags: modlayer, gameplay, unpacked, items, crafting, map
- `modlayer/gameplay/actor-survival.md` — **(M17)** actor survival map: status hub + HUD strip,
  sleep/thirst meters, arszi_psy, actor_effects sensory layer, difficulty toggles, actor menu &
  inventory UI; routes F103 + F104 — domain complete. tags: modlayer, gameplay, unpacked, actor,
  survival, map
- `modlayer/gameplay/warfare.md` — **(M18)** warfare metagame map: hub/smart/squad/faction/level
  pieces, commander deck, options surface, PDA UI; routes F105. tags: modlayer, gameplay,
  unpacked, warfare, map
- `modlayer/gameplay/gamemodes-services.md` — **(M19)** game modes & services map: flag seeding,
  difficulty store, world setup, save policies, death rules + the `game_*` services
  (stats/achievements/autosave/travel); routes F106 + F107 — domain complete. tags: modlayer,
  gameplay, unpacked, gamemodes, services, map
- `modlayer/gameplay/ui-scripts.md` — **(M20)** the `ui_*` Lua layer: dialogs (`CUIScriptWnd`) +
  HUD overlays (custom-static) built from `configs/ui/*.xml`; routes F104 (inventory) + F109 (menu/
  HUD/widgets) + F110 PDA tabs + F111 debug tools + F112 news/tips — ui_* flow coverage complete;
  engine UI core F19, options F17.
  tags: modlayer, gameplay, unpacked, ui, hud, map
- `modlayer/gameplay/axr-addon.md` — **(M21)** the `axr_*` modded-exe addon layer: the `axr_main` boot
  orchestrator (F65), the behaviour schemes (F113), and the task/companion/trade managers (F114); domain
  complete. routes F113 + F114. tags: modlayer, gameplay, unpacked, axr, scheme, manager, map
- `modlayer/gameplay/utils-db.md` — **(M22)** the utility & registry foundation (bottom of the gameplay
  stack): the `utils_*` helper libraries (F115), the `db` global state registry (F69), and the script
  bootstrap (`_g`, `class_registrator`). routes F115 + F69. tags: modlayer, gameplay, unpacked, utils, db,
  registry, map
- `modlayer/gameplay/configs-data.md` — **(M23)** the config/data tier (Phase G9): creatures/actor
  (F118), zones (F119), items/weapons (F120), per-smart logic (F121), plugins/tweaks (F122), spawns/ALife
  (F123); the LTX/DLTX parse+inherit+patch mechanism is `modlayer/config-ltx.md`. routes F118. tags:
  modlayer, gameplay, config, ltx, data, map

### features/ — cross-cutting feature notes
- `features/spom.md` — Silhouette POM. **Branch-specific: lives on the `spom` branch, NOT on the
  current working branch.** Pointers to `claudedocs/` design docs + SPOM auto-memory. tags: spom,
  renderer, shaders
- `features/pda-tab-scroll.md` — dynamic PDA tabs from Lua (`pda_dynamic_tabs` registry) + strip
  overflow handling (slack squeeze, then scroll with spliced/skin arrows) + per-tab icons; per-tab
  geometry, convex hit clips, two-pass clipped draw. **Merged to mainline** (was `feat/tabs*`).
  tags: ui, pda, tabs, scripting, cpp, lua
- `features/spawn-overlays.md` — load-time `all.spawn` assembly: `spawn_overlays.ltx` (graph links/
  unlinks, spawn remove/patch/replace, path remove, level cut), SDK `level.spawn`/`level.game`
  fragments, level packs (extra `.spawn` files), legacy-save id translation, v13 large AI maps,
  `nearest_vertex`/`create_on_level`. tags: alife, spawns, graph, ltx, dltx, level-pack
- `features/script-attachments.md` — `script_attachment` (HUD/world/camera models with bones, physics,
  light, 3D UI, shaders), `CUI3dStatic` 3D icons + DX11 atlas, first-person hand poses / arm IK /
  motion-mark timing. tags: lua, animation, weapons, ui, physics, renderer
- `features/shader-bus.md` — named `bus_<id>` shader lanes written from Lua (`shader_bus.*`), engine/
  cvar/object lanes, console + ImGui monitor; guide `docs/shader_bus.md`. tags: renderer, shaders, lua

## Quick routing

| Task involves… | Start at |
|---|---|
| AI behavior in C++ | `engine/xrGame.md` |
| How an object spawns/updates/dies (the backbone) | `flows/object-lifecycle.md` |
| Player actor: input/movement/camera/condition/death | `flows/actor.md` |
| Weapon show/fire/reload/zoom states & HUD anim | `flows/weapon-state-machine.md` |
| Weapon class tree / scope-silencer-GL addons / ammo & magazine | `flows/weapon-hierarchy-addons.md` |
| Outfit/helmet armor, protection, why armor fails, restore bonuses | `flows/outfit-helmet.md` |
| Detector finds artefacts / detector tiers / artefact belt effects & activation | `flows/detector-artefact.md` |
| Item upgrades: grid/groups, install gating, how a stat delta applies | `flows/upgrade-system.md` |
| Anomaly zones: detection, blowout cycle, damage/impulse, campfire | `flows/anomaly-zones.md` |
| Survival: health/stamina/hunger/radiation/bleeding regen, boosters | `flows/actor-condition.md` |
| Info-portions / faction goodwill / friend-enemy attitude / reputation | `flows/relations-info.md` |
| Player quests/tasks: give, completion check, state, PDA waypoint | `flows/tasks.md` |
| Item class tree / dropped-item physics / bone attach / condition | `flows/item-base.md` |
| Camera shake/bob/recoil, screen post-process FX, level.add_*_effector | `flows/camera-pp-effectors.md` |
| Lua `game_object` binding, obj:set_callback, how a method delegates | `flows/cscript-game-object.md` |
| Eating/meds/boosters use; grenade throw/cook/explode, blast vs frags | `flows/consumables-explosives.md` |
| Map markers/quest spots, PDA contacts, news popups | `flows/pda-map-news.md` |
| Game time/time-factor, emissions/surge (Lua vs C++), ALife services | `flows/alife-services.md` |
| Dropped-object/destructible physics, death ragdoll, hanging lamp | `flows/physics-objects-ragdoll.md` |
| Cars/turrets mounting, drive control handoff, helicopter (AI) | `flows/vehicles-holder.md` |
| Monster AI behavior (FSM/controls, not GOAP), jump/attack moves | `flows/monster-ai-control.md` |
| How an NPC moves/aims/shoots once it decides (execution layer) | `flows/stalker-execution.md` |
| Smart covers / loopholes / why NPCs use cover, cover animations | `flows/smart-cover.md` |
| Script-controlled NPCs / xr_logic C++ side / action queue | `flows/script-npc-actions.md` |
| Squad combat: shared enemies, target/cover assignment among members | `flows/agent-squad-combat.md` |
| Trader / crow / phantom (special non-GOAP actors) | `flows/special-actors.md` |
| CLevel per-frame, object collection, spawn/destroy, input, level load | `flows/level-runtime.md` |
| Bullet holes/blood/scorch decals, impact particles, material effects | `flows/wallmarks-particles.md` |
| Item pickup/equip/slot-belt-ruck/drop | `flows/inventory.md` |
| Hit/damage/health/wounds/death of any creature | `flows/damage-hit.md` |
| NPC AI decisions / GOAP / script-control seam | `flows/ai-planner.md` |
| NPC senses / vision / memory / enemy selection | `flows/ai-perception.md` |
| smart_terrain jobs / xr_logic schemes / NPC binder | `flows/ai-squad-logic.md` |
| Rendering, shaders, a visual bug, SPOM | `engine/renderer.md` → `features/spom.md` |
| Passing values from Lua to a shader (`bus_*`, replacing `shader_param_N`) | `features/shader-bus.md` |
| Scripted weapon/HUD/world attachments, 3D UI icons, first-person IK / hand poses | `features/script-attachments.md` |
| Monster Lua combat seam (`monster_on_combat_action`, `set_monster_*`, `try_monster_*`) | `flows/monster-ai-control.md` |
| Veto/attenuate script, ambient, thunder, rain or music sounds from Lua | `flows/sound-music.md` |
| RenderDoc captures / LuaJIT CPU or allocation profiling | `flows/ui-debug-tools.md` |
| How a shader/material loads, texture/constant dedup, getShaderPath | `flows/renderer-resources-shaders.md` |
| What gets drawn each frame / render queue / culling order / shadow queues | `flows/renderer-scene-graph.md` |
| Post FX chain, SSAO/bloom/DOF/SMAA/SSR, SSFX toggles, phase_combine order | `flows/renderer-post-process.md` |
| How a blender builds passes / blender CLSID / r_Pass-r_End / add a shader technique | `flows/renderer-blender-compile.md` |
| Culling: HOM occluders, sectors/portals, occlusion queries, "why isn't X drawn" | `flows/renderer-visibility.md` |
| How a draw is submitted / RCache state caching / constant buffers / dynamic VB-IB | `flows/renderer-backend.md` |
| Model/visual types, .ogf load, LOD, model pool/instancing, why geometry is shared | `flows/renderer-visual-types.md` |
| Skeleton bones, animation blending, bone matrices, ragdoll-vs-animation, skinning | `flows/renderer-skeleton.md` |
| Dynamic lights, shadow maps, light accumulation, shadow atlas, "light not lighting X" | `flows/renderer-lighting.md` |
| Sun shadows, cascaded shadow maps (CSM), shadow shimmer/seams, directional lighting | `flows/renderer-sun-shadows.md` |
| Bullet holes/blood/scorch decals (engine side), wallmark aging, decal not following corpse | `flows/renderer-wallmark-engine.md` |
| D3D device/swapchain, present/vsync, resolution-fullscreen reset, GPU caps/MSAA support | `flows/renderer-hw-device.md` |
| How engine reaches renderer / render factory / UI-font-console-debug rendering / present | `flows/renderer-factory-bridges.md` |
| Particle effects rendering / billboards / particle sim seam / particle sort order | `flows/renderer-particles.md` |
| Sky/clouds/rain/lightning/lens-flares rendering, gamma/brightness, weather visuals | `flows/renderer-environment.md` |
| Grass/detail objects, grass density/distance, grass shadows, grass pop-in | `flows/renderer-detail-grass.md` |
| R1 renderer / static light-maps / what "R1" means / why a path has no deferred lighting | `flows/renderer-r1-path.md` |
| R2/R3 renderers / DX9-vs-DX10-vs-DX11 differences / render generation deltas | `flows/renderer-r2-r3-paths.md` |
| Volumetric smoke/fluid sim, 3D fluid, why volumetric fog isn't rendering | `flows/renderer-3dfluid.md` |
| A config value / `mod_system_*` / DLTX patch | `modlayer/config-ltx.md` |
| An XML/UI-layout change from Lua | `modlayer/dxml.md` |
| A new in-game option/toggle | `modlayer/options-ui.md` |
| Lua callback / new `.script` module | `modlayer/scripting.md` |
| Build/branch/variant questions | `engine/_overview.md` |
| Engine boot / main loop / per-frame ordering | `flows/engine-boot-frame.md` |
| R4/DX11 render passes / deferred pipeline | `flows/render-frame.md` |
| Save/load a game, what gets serialized | `flows/save-load.md` |
| ALife online/offline switch, transition crash | `flows/alife-online-offline.md` |
| How an option goes from definition to effect | `flows/options-runtime.md` |
| NPC dialog / phrase graph / trade & pricing | `flows/dialog-trade.md` |
| Building a UI/HUD window from XML | `flows/hud-ui-load.md` |
| UI stretched/squashed at 16x9 or ultrawide / UI text too small at 4K / where `kx` and `_16` XMLs come from / adding a font atlas rung (`texture_h<N>`) / crisp `border_*` frame-line bands | `flows/ui-scaling.md` |
| Adding a PDA tab from a mod / tab strip scrolls or clips wrong / tab icons + scroll-arrow art | `features/pda-tab-scroll.md` |
| Per-NPC AI tuning from Lua (vision, view distance, aim, fire queue, enemy-selection bias, regen) | `flows/ai-perception.md` + `flows/stalker-execution.md` |
| Vetoing an engine AI decision from Lua (combat action switch, cover re-pick) | `flows/stalker-execution.md` |
| Texture VRAM growth / mid-session texture eviction cvars | `flows/renderer-resources-shaders.md` |
| Adding a modded-exes console var + options row | `flows/options-runtime.md` |
| Reload-mark/anim-end/footstep callbacks | `flows/animation-callbacks.md` |
| ALife/squads/net sync | `engine/server-entities.md` |
| How the Lua gameplay layer boots / load order | `flows/gameplay-boot.md` |
| How NPC scripted logic (schemes) runs/switches | `flows/scheme-engine-core.md` |
| Where per-object Lua state lives (`db.storage`) | `flows/db-registry.md` |
| condlist `=func`/`%effect%` (xr_conditions/xr_effects) | `flows/condlist-library.md` |
| Authoring/parsing NPC logic LTX (`on_*`, condlists) | `flows/logic-ltx-switching.md` |
| How an NPC reaches a posture/animation state | `flows/state-manager-core.md` |
| state_mgr channels / animation marker machine | `flows/state-mgr-controllers.md` |
| smart_terrain jobs → NPC logic at spawn | `flows/smart-terrain-core.md` |
| How a smart's job list is built/ranked (gulag) | `flows/gulag-jobs.md` |
| How any xr_* NPC scheme is structured | `flows/scheme-walker-patrol.md` (template) |
| How scripted NPCs fight / ignore combat | `flows/scheme-combat.md` |
| Companions (recruit/command/follow) | `flows/companion-system.md` |
| Camper/cover (hold position, find cover) | `flows/scheme-camper-cover.md` |
| Animpoint (sit at smartcover/campfire) | `flows/scheme-animpoint.md` |
| Sleeper / wounded / death schemes | `flows/scheme-sleeper-wounded-death.md` |
| Meet/dialog-enable, abuse, faction bribe | `flows/scheme-meet-abuse-bribe.md` |
| NPC looting items/corpses/boxes | `flows/scheme-gather-corpse.md` |
| xr_sound/hear/zones/detector/weapon_jam/reach | `flows/scheme-sensors-misc.md` |
| Lua server entity for an NPC / `se_stalker` / ALife register-save-switch hooks | `flows/se-stalker.md` |
| Other `se_*` classes (monster/item-catalog/zones/actor/smart_cover/heli/car) / which clsid → which se_ class | `flows/se-others.md` |
| The `se_*` Lua server-class family (overview / map) | `modlayer/gameplay/server-entities-lua.md` |
| How a squad picks where to go / squad target selection / `sim_squad_scripted` / squad move-FSM | `flows/scripted-squad.md` |
| `SIMBOARD` / smart registry / assign squad to smart / world population / `get_squad_target` | `flows/sim-board.md` |
| Offline squads fighting / offline combat simulator / OCS / how offline kills happen | `flows/offline-combat.md` |
| Squad simulation overview (squads / sim_board / target scoring / offline combat) | `modlayer/gameplay/squads-simulation.md` |
| How Lua attaches to an object / binders / object_binder / bind_object / actor_binder / `class_registrator` | `flows/binder-architecture.md` |
| Campfire/camp/faction/box/physics-prop binders / `generic_physics_binder` / box loot on smash | `flows/world-object-binders.md` |
| Where artefacts spawn in anomaly zones / anomaly layers / dynamic (relocating) anomalies / bolt-only activation | `flows/anomaly-field-binders.md` |
| Silence/no-weapon/psy/deimos/postprocess/timer zones / passive `sr_*` restrictor effects | `flows/restrictors-passive-sr.md` |
| Teleport zones / crow+monster spawn zones / camera cutscenes / camp storytelling / active `sr_*` | `flows/restrictors-active-sr.md` |
| The `sr_*` restrictor-scheme family (overview / passive vs active) | `modlayer/gameplay/restrictors-sr.md` |
| Scripted doors/buttons/code pads / scripted cars & mounted miniguns / `ph_*` schemes / psy-phantom spawning | `flows/physics-object-schemes.md` |
| The `ph_*` physics-scheme family (overview) | `modlayer/gameplay/physics-schemes-ph.md` |
| Monster logic schemes / `mob_home` leash / monster patrol / scripted trader performance / `on_trade` triggers | `flows/monster-schemes.md` |
| The `mob_*` monster-scheme family (overview) | `modlayer/gameplay/monster-schemes.md` |
| Helicopter behavior / why a heli won't attack / heli combat & patrol / faction air support / `heli_enemy_flag` | `flows/helicopter-ai.md` |
| Vehicles Lua tier (heli stack + car binding `bind_car`/`se_car`) | `modlayer/gameplay/vehicles-heli.md` |
| How quests work / give_task / task condlists & functors / quest rewards / repeatable tasks / `tm_*.ltx` | `flows/task-system-lua.md` |
| Bounty/defense/fetch/clear-map quest types / `tasks_*` plugins / task stages / setup_X_task effects | `flows/dynamic-task-types.md` |
| The Lua task family (core + dynamic types + AXR bounties) | `modlayer/gameplay/tasks-lua.md` |
| Dialog content/gating: who offers which dialog, phrase functors, NPC small talk (`dialog_manager`), task-offer/companion/broker menus | `flows/dialog-system-lua.md` |
| The dialog & character-info family (overview: scripts + XML graphs + attachment) | `modlayer/gameplay/dialogs-info.md` |
| How an NPC gets its name/icon/faction/dialog list (profile→specific_character), info-portion XML registration, the PDA encyclopedia / article unlocks / recipe notes | `flows/characters-info-encyclopedia.md` |
| Trader stock & restock / trade profiles `trade_*.ltx` / why a trader won't buy an item / trade prices & discounts / NPCs selling loot to traders / NPC weapon upgrades at mechanics | `flows/trade-economy.md` |
| The trade & economy family (overview: profiles + Lua UI + NPC economy + eco knobs) | `modlayer/gameplay/trade-economy.md` |
| Item use/combine/drag behaviors / repair kits / disassembly & parts / cooking / recipes / batteries & devices / scopes & silencer toggle / ammo wheel / player stashes / `IsItem` types | `flows/item-behaviors-lua.md` |
| The item-behavior family (overview: registry + hub + plugin clusters) | `modlayer/gameplay/items-lua.md` |
| Sleep/thirst deprivation meters / psy health & zombification / HUD status indicators & boosters strip / safe cover for emissions / item-use animations & screen overlays / forced sleep | `flows/actor-survival-status.md` |
| Inventory window & context menus / right-click custom actions (`useN_functor`) / drag-drop & quick slots / loot & body search / mechanic repair-upgrade UI / HUD messages (`set_msg`) / why the engine ActorMenu never opens | `flows/actor-menu-inventory-ui.md` |
| The actor-survival family (overview: meters + psy + HUD + sensory layer + inventory UI) | `modlayer/gameplay/actor-survival.md` |
| Warfare mode / faction war / smart capture & ownership / faction respawn & invasions / buying squads & helis / influence points / warfare PDA tab & fog of war | `flows/warfare-metagame.md` |
| The warfare family (overview: hub + smart/squad layers + commander + options) | `modlayer/gameplay/warfare.md` |
| Difficulty settings & factors / ironman & azazel modes / save restrictions (agony/timer/campfire) / random world items on new game / why a save got deleted | `flows/game-modes.md` |
| Achievements (how/when they unlock, why one never fires) / player rank & reputation & news tips / NPCs ranking up / autosave timer / fast travel & backpack travel on the PDA map / visited-smart tracking | `flows/game-services.md` |
| Faction-vs-faction relations / `is_factions_enemies`/`is_factions_friends` / dynamic relations (kills shifting faction standings) / squad & gulag goodwill helpers / `communities_relations` matrix & `game_relations.ltx` sections / how to set goodwill between NPCs/factions | `flows/faction-relations-lua.md` |
| Main menu (buttons, save veto, new-game/F2 test, callback seams) / enemy health bar / dosimeter HUD / battery HUD / message boxes & right-click context menus (`context_props`) / the Pripyat "leave the Zone" prompt | `flows/gameplay-ui-screens.md` |
| PDA tabs (contacts list / dead-NPC PDA messages / radio & music player / faction relations matrix / companion roster / warfare board) / how a PDA tab is dispatched / the Top-100 character ranking board / anomaly & sleep map spots / how contacts get recorded | `flows/pda-tabs.md` |
| Debug launcher (F7 "Anomalous Debugger") / item & object spawners / weather-lighting-wpn_hud editors / the live debug HUD inspector (planner/squad/heli/monster panels) / debug command registry / `DEV_DEBUG` flag / Lua remote debugger / why debug tools don't appear in a normal build | `flows/ui-debug-tools.md` |
| PDA tip popups / item & money & treasure showcase icons / task-state notifications / the ambient "Dynamic News" radio chatter (stalker deaths, loot, companion talk, faction bulletins) / why dynamic news goes silent (surge/underground/dead PDA) / `news_manager` vs `dynamic_news_manager` | `flows/ui-news-tips.md` |
| Companion follow/escort/patrol behaviour (`axr_beh`, the `npcx_beh_*` flags) / NPCs fighting from cover or panicking or hiding from helicopters / stalkers lighting smart-terrain campfires / scriptable custom hotkeys (`axr_keybind`, `kCUSTOM*`) / how an `axr_*` behaviour scheme plugs into the GOAP planner | `flows/axr-behaviour-addons.md` |
| Faction honchos handing out randomized storyline/mechanic/change-faction tasks / bounty tasks (target tracking, why a bounty completes even if someone else gets the kill) / hostage/escort task registries / the DRX-storyline task generator on top of the engine task core | `flows/axr-managers.md` |
| Recruiting/dismissing companions / commanding squadmates (combat/move/stealth/loot/distance modes, the command wheel `kCUSTOM18`) / giving companions items & the carry-weight cap / spawning story escort squads / companion auto-heal & unstuck teleport / the `npcx_beh_*` command flags (manager side) | `flows/axr-managers.md` |
| The `axr_*` modded-exe addon layer (overview: boot orchestrator + behaviour schemes + task/companion/trade managers) | `modlayer/gameplay/axr-addon.md` |
| The `ui_*` UI-scripts family (overview: dialogs + HUD overlays, what routes where) | `modlayer/gameplay/ui-scripts.md` |
| Which `utils_*` helper has the function I need / cover & movement primitives (`validate`/`lmove`/`try_go_cover`) / item price & tradability (`get_item_cost`) / parsing condlists or LTX from Lua / the `cfg_file` read-write INI writer / resolution-scaling a custom UI / the Lua inventory-cell engine / reading or editing a server object's packet (`utils_stpk`) | `flows/utils-libraries.md` |
| The utility & registry foundation (overview: `utils_*` libs + `db` registry + script bootstrap) | `modlayer/gameplay/utils-db.md` |
| Dynamic combat music (why it starts/stops, why monsters don't trigger it, how to change the themes) / level background music / crossfading & ducking the music channel / scripted NPC sounds & campfire stories / level ambient + wind + rain-on-helmet sound / zone/physics-object sounds / the `sound_object` lifetime patch | `flows/sound-music.md` |
| How stash loot is generated/spawned / why a box is empty or why loot didn't change after editing configs / tiers & kinds & economy-type filtering of treasure / rare stashes & toolkit rewards / predefined vs procedural secrets / quest stashes / radio stash hints / treasure map spots | `flows/treasures-secrets.md` |
| Where a monster's HP/speed/AI-class/immunities/loot are configured / the `m_*.ltx` → `spawn_sections` inheritance chain / actor movement & mass params (`actor.ltx`) / bone-damage & resistance profiles (`damages.ltx`/`immunities.ltx`) / retuning a creature globally vs per-level | `flows/creatures-actor-configs.md` |
| Where an anomaly's strength/damage-type/blowout timing/particles are configured / the `zone_*.ltx` → engine `class = ZS_*` mapping & `weak/average/strong` tiers / why editing config artefact fields doesn't change artefact spawning (F90 binder owns it) / field (passive DoT) vs mine (blowout) anomaly families / how level-changers / level transitions are configured (`sr_teleport_sections.ltx`: spot/text/path/reject_path) | `flows/zones-anomaly-configs.md` |
| Where a weapon's damage/recoil/ammo/scope/jam params are / how to add a new gun-outfit-item without conflicts (the `w_*`/`o_*`/`items_*` glob auto-include) / weapon variant & scope-attach-state subsections / where outfit protection & immunities are configured / ammo `k_*` ballistic coeffs / per-item upgrade trees / why editing shipped `w_*` files is discouraged | `flows/items-weapons-outfits-configs.md` |
| Where a smart-terrain's NPC jobs / an object's logic (door/sleep/teleport/box) is configured / the `[logic@job]` gulag job catalog with `prior`/`suitable` / smart respawn & spawn_squads defs / why a logic file does nothing (custom_data pointer) / `smart/<smart>.ltx` vs `*_smart_logic.ltx` / how to edit campfire stories | `flows/per-smart-logic-configs.md` |
| Where the shared base/parent sections are (`defines.ltx`: hud_base/booster/default_weapon_params/monster_base) / video & difficulty & warfare presets / why `axr_options.ltx` is empty / where a plugin's or ai_tweak's numbers live (achievements/difficulty/combat_ignore/simulation_objects/guaranteed_drops/…) vs which flow owns the behavior / the constants-vs-content distinction | `flows/plugins-tweaks-presets-configs.md` |
| Adding levels / linking levels / removing, patching or adding spawned objects or patrol paths without the SDK / level packs / v13 AI maps / spawning by level name | `features/spawn-overlays.md` |
| Where the world's objects are placed (`all.spawn` binary) / how to edit spawns & why save doesn't change / ALife simulator tuning (`alife.ltx`: time_factor/switch_distance/schedule) / the game-graph point catalog / story-IDs (finding a specific placed object) / the level & world-map registry / why `patrol_paths.ltx` is empty (paths live in all.spawn) | `flows/spawns-alife-graphs-configs.md` |
| The config/data tier overview (creatures/zones/items/logic/plugins/spawns LTX, the DLTX patch model) | `modlayer/gameplay/configs-data.md` |
| The gamemodes & services family (overview: flags + difficulty + setup + modes + services) | `modlayer/gameplay/gamemodes-services.md` |
| The `bind_*` binder family (overview / world/anomaly/restrictor/physics object binders) | `modlayer/gameplay/binders.md` |
| Anything else in `unpacked/` (gameplay corpus) | `modlayer/gameplay/_overview.md` → `ROADMAP-gameplay.md` |

## Sync log

One entry per repo→KB sync, newest first. The banner at the top of this file is the authoritative
"synced to" marker; this log is what each sync actually changed. Where an entry here and a row in the
older "Audit status" table disagree about what was verified, **the newer entry wins**.

### 2026-11-13 — `becde89ee` → `3470feeb2` (branch `all-in-one-vs2022-wpo-mt`)

308 non-merge commits walked (`git log --no-merges becde89ee..HEAD`), cross-checked against the
`README.md` changelog entries 2026.09.12, 2026.10.11 and 2026.11.09 (prerelease). Each cluster was
reconciled against code or diffs read this session.

**New files:** `features/spawn-overlays.md` (spawn overlays, level packs, fragments, level cut, legacy
saves, v13 AI maps), `features/script-attachments.md` (attachments, 3D icons/atlas, arm IK, hand poses),
`features/shader-bus.md`. GLOSSARY: `spawn-overlay`, `shader-bus`, `script-attachment`,
`callback-tier`.

Reconciled (each `updated: 2026-11-13`):

- **Callbacks** — `flows/lua-callback-lifecycle.md` (`callback_set` ordering: before/after + Pre/Normal/
  Post tiers, transactional reorder; **`on_xml_read` is now an ordinary bus callback**, the
  `dxml_core` monkey-patch / `xmlCallbacks` / `xmlDispatch` are gone), `flows/dxml-xml-interception.md`
  (same), `modlayer/scripting.md` (new bus callbacks + Lua math/ini exports).
- **AI** — `flows/monster-ai-control.md` (Lua combat seam: `_G.CMonsterCombatAction` consumed by every
  species state manager, readers/setters/maneuvers, `set_monster_dest`), `flows/ai-perception.md`
  (`Vision::o_trace` ray fix, anomaly spatials `STYPE_FEELVISIONIGNORE`, hit-memory null guard),
  `flows/stalker-execution.md` (`set_movement_hold`, `npc_dont_drop_weapons_on_death`,
  `npc_relaxed_idle_mode`, smart-cover reload speed cap, throw-force hooks).
- **ALife** — `flows/alife-online-offline.md` (online squad member location sync, ixray squad
  teleport port, deferred spawn children, `CCustomMonster::net_Relevant`),
  `flows/spawns-alife-graphs-configs.md` (banner + stale "not patchable"/"stub" claims corrected).
- **Weapons/physics** — `flows/weapon-hierarchy-addons.md` (`CWeaponSSRS` rework, script weapon
  controls), `flows/physics-objects-ragdoll.md` (frozen shells, vector scale), `flows/anomaly-zones.md`
  (gravi-zone exit fix), `flows/camera-pp-effectors.md` (cam effector speed, PP queries, CamAttached FOV,
  damage stagger factor).
- **UI/PDA** — `flows/hud-ui-load.md` (global mouse-capture stack, hide lock, clip rects, `FindChild` +
  casts, `SetLineTo`, `ShowAt`, custom-statics lock, Shniaga modal pass), `flows/pda-map-news.md`
  (map-location callbacks, bulk spot read, `CUIMiniMapWidget`).
- **Sound/environment** — `flows/sound-music.md` (six veto/volume hooks, distance cvars),
  `flows/renderer-environment.md` (env mods from LTX, frame-rate independent wetness,
  `set_rain_wetness`, thunderbolt pause fix).
- **Renderer** — `flows/renderer-scene-graph.md` (R4 full-detail distance, `mapLOD` clear, legs fix,
  forward-stage attachment UI), `flows/renderer-resources-shaders.md` (texture `.ltx` descriptors,
  `.seq` dedup, constants typing, frame count in `ssfx_timedelta.y`).
- **Engine/MT/debug** — `engine/engine-core.md` (`thread_spawn_ex`, particle/bullet locks, CFORM
  `build_arrays`/`build_tree`, FS + luabind hardening), `flows/engine-boot-frame.md` (`seqFrameMT`
  before parallel GC, splash/alt-tab/loading-art fixes), `flows/ui-debug-tools.md` (RenderDoc, LuaJIT
  profilers, `dump_cvar_enable`, ImGui show/hide).
- **Config/misc** — `modlayer/config-ltx.md` (root `mod_*.ltx` list), `flows/dltx-config-load.md`
  (`reload_system_ini` clears the whole cache), `flows/tasks.md` (escort fixes),
  `flows/trade-economy.md` (antifreeze drop events, Sidorovich/Forester callbacks). `DXML.md` moved to
  `docs/DXML.md`: all KB references repointed.

Deliberately left open:

- `src/.clang-format` / `.editorconfig` / "formatter changes" (repo-wide formatting) are not documented.
- Not traced: the 1819-line `script_attachment_manager.cpp` body beyond its header and Lua surface;
  `CUIMiniMapWidget` internals beyond its API; `spawn_packs.cpp` substitution/drift heuristics beyond
  the header contract.
- `vis_data::skip_marker` is set by attachments but no renderer reader was found. Recheck next sync.
- Verdatim's additive-animation PR was reverted (`b86d733f4`); nothing documented for it.
- Pre-existing `related:` failures listed in `CONVENTIONS.md` are unchanged.

### 2026-08-17 — `d7c39ae2` → `becde89ee` (branch `all-in-one-vs2022-wpo-mt`)

~100 non-merge commits walked (`git log --no-merges d7c39ae2..HEAD`, cross-checked against
`README.md`'s changelog); every affected file reconciled **against code read that session**.

Reconciled (each `updated: 2026-08-17`):

- **UI/fonts/frame-lines** — `flows/ui-scaling.md` (dynamic font-atlas ladder `GetFontTextureName`
  replacing three duplicated ladders; `WidthScale` glyph widths; frame-line cap scaling + crisp
  `border_*` bands; `CUILines` vertical-align fix), `GLOSSARY.md` (`font-atlas-rung` rewritten,
  `crisp-band` added), `flows/hud-ui-load.md` (`AttachChildKeepOwner`, `OnMouse` script hook,
  `SetHitClip`, custom-draw backgrounds).
- **PDA** — `features/pda-tab-scroll.md` **rewritten** (feature merged; `endcap_width` /
  `frame_cap_scaled` / `scroll_texture` / runtime "healing" are gone — slack squeeze, per-tab overlap
  rebuild, `CUITabScrollArrows`, icons, `SlidePolyIntoBand` are current), `flows/pda-tabs.md` (dynamic
  tab registry), `flows/pda-map-news.md` (multi-spot merge + context replay + `AddHeader`/`AddSubmenu`/
  row TAG + faction spot-size DXML fix).
- **AI** — `flows/ai-perception.md` (per-NPC vision/view-distance/enemy-bias/hit-redirect levers,
  `make_enemy_visible`, actor-visibility moved into `CActor`, uninitialized `m_hit_redirect_*` hazard),
  `flows/stalker-execution.md` (aim params — note the now-live `ai_aim_predict_time`; fire-queue scale;
  fire gates; the two veto callbacks), `flows/damage-hit.md` (`bullet_check_visual`,
  `is_hit_anim_playing`, regen boost), `flows/monster-ai-control.md` (pseudogiant cvars, burer
  `remove_links` lesson), `flows/cscript-game-object.md` (where repo bindings get documented).
- **Renderer** — `flows/renderer-resources-shaders.md` (mid-session texture eviction + PPL re-load
  contract + unload skips), `flows/renderer-post-process.md` (HUD excluded from the emissive pre-pass),
  `engine/renderer.md`.
- **Engine/core** — `engine/engine-core.md` (console key-binding table + numpad/NumLock),
  `engine/xrCore.md` (`FS_Path` hardening), `engine/physics-collision.md` (id-not-pointer collision
  callback), `flows/alife-services.md` (three spawn/scheduler crash guards),
  `flows/spawns-alife-graphs-configs.md` (`CPatrolPoint::load_from_config` now works),
  `engine/_overview.md` (incremental LTCG, two project files per module, `-mt` working branch).
- **Mod layer** — `flows/dxml-xml-interception.md` + `modlayer/dxml.md` (parent invariant, mutation
  fixes, the reverted malformed-tag leniency), `modlayer/config-ltx.md` (`mod_system_anomaly_fixes.ltx`;
  `mod_system_signal_light.ltx` is gone), `modlayer/scripting.md` (veto/replacement patterns,
  `aaaa_script_fixes_mp` role, `lua_help_ex` is documentation-only),
  `flows/options-runtime.md` (four-file checklist for a new option row),
  `flows/inventory.md` (`GEG_PLAYER_ITEM2SLOT` gained a trailing `u8`),
  `flows/weapon-state-machine.md` (NPC reload edges), `flows/animation-callbacks.md` (motion-mark
  speed scaling), `flows/upgrade-system.md` (upgrade-granted scopes),
  `flows/vehicles-holder.md` (`CAR_NEW`: drone / visual camera / camera scopes).

Deliberately **not** covered: nothing in the range was left undocumented, but the newest features are
documented at their seams (config keys, callbacks, cvars) rather than as full L3 digests — deepen when a
task needs it.

Left open (pre-existing, not caused by this range): the four dangling `modlayer/gameplay/*` `related:`
targets recorded in `CONVENTIONS.md` → "Verifying the whole KB".

### Earlier

- **2026-06-26 @ `d7c39ae2`** — sync recorded only as a KB commit (`upd`); no per-file log was kept.
- **2026-06-04/05** — initial build + trust audit; see the "Audit status" table below and `ROADMAP.md`.

## Audit status (2026-06-04)

Honest per-file state after the trust audit. **verified** = claims traced to source this session;
**map** = a verified L1 map (structure/named symbols verified; every major mechanism delegated to a
verified flow; a few breadth items with no flow still marked inline *(INFERRED)*) — complete *as a
map*, not a full digest; **partial** = structure/named symbols verified, some behavioral claims still
marked inline *(INFERRED — unverified)*; **unaudited** = prose not yet re-checked against code — do
not trust.

| File | Status | Notes |
|---|---|---|
| flows/lua-callback-lifecycle.md | verified | script_engine.cpp, _g.script, axr_main, dxml_core; `dynamic_callbacks.lua` shadowing RESOLVED (debug-only load + superseded by _g → live bus always _g→axr_main) |
| flows/dltx-config-load.md | verified | read Xr_ini.cpp parse+merge; `EvaluateSection` not line-by-line (flagged) |
| flows/dxml-xml-interception.md | verified | read xrXMLParser→ScriptXMLInit→dxml_core COnXmlRead |
| flows/weapon-fire-callbacks.md | verified | full chain read: state_Fire→OnShot→FireTrace(buckShot)→FireBullet→AddBullet→CommitEvents→UpdateWorkload(process_bullet, parabolic ballistics, opt. MT seqParallel)→RayQuery→eHit; + misfire/jam |
| flows/object-lifecycle.md | verified | spine spot-verified against code (CGameObject hierarchy; net_Spawn/net_Destroy/shedule_Update/UpdateCL bodies; CObject::net_Spawn registration; g_sv_Spawn; _G.CGameObject_NetSpawn/NetDestroy + eNetSpawnAfter). Agent-assisted research; a few sub-steps marked agent-traced inline |
| flows/actor.md | verified | spine spot-verified: CActor inheritance (CEntityAlive/IInputReceiver/CInventoryOwner/Feel/CStepManager), all callback sites (eHit, eEnter/eExitLevelBorder, eOnWeaponRaised/Lowered, eAttach/eDetachVehicle + _G.CActor_Fire/on_jump/on_land/__BeforeHitCallback), function homes. Movement on deferred channel. Deep bodies (net_Spawn/shedule_Update/UpdateCondition/Die) agent-traced inline |
| flows/weapon-state-machine.md | verified | spine spot-verified: both state enums (CHUDState::EHudStates, CWeapon::EWeaponStates incl. tri-state substates), OnAnimationEnd→SwitchState routing, CHudItem::UpdateCL m_dwMotionEndTm animation-end driver + OnMotionMark. switch2_*/PlayAnim*/ReloadMagazine bodies agent-traced inline |
| flows/inventory.md | verified | spine spot-verified: CInventory::Take placement cascade (stored place → slot/belt/ruck fallback, ruck terminal) + Activate deferred active-slot machine (m_iActiveSlot/m_iNextActiveSlot, OnServer guard, SendDeactivateItem). Slot/Belt/Ruck/DropItem/Update/owner callbacks/GE_OWNERSHIP_* handlers agent-traced inline |
| flows/damage-hit.md | verified | spine spot-verified: SHit/GenHeader (Hit.h), CEntityAlive::Hit→CDamageManager::HitScale→ConditionHit chain, _G.CBeforeHitAfterCalcs Lua hook, callback(GameObject::eDeath) in CEntityAlive::Die, CActorCondition::ConditionHit chains to generic. UpdateCondition/bleeding/GE_DIE server path agent-traced inline |
| flows/ai-planner.md | verified | spine spot-verified: CAI_Stalker::Think→brain().update(), GetScriptControl→ProcessScripts fork, StalkerDecisionSpace::eWorldProperty*/eWorldOperator* enums (+ Lua export). A* solver internals (problem_solver_inline) + action/evaluator virtuals agent-traced/SKIMMED inline; base xr_logic Lua cited (unpacked/) |
| flows/ai-perception.md | verified | spine spot-verified: CMemoryManager::update + visual/sound/hit/enemy/danger manager homes, CSoundMemoryManager::feel_sound_new, CHitMemoryManager::add, CEnemyManager:CObjectManager, CCustomMonster::shedule_Update driver. Vision formula/danger scoring/enemy weights agent-traced/SKIMMED inline. **UPDATED 2026-08-17:** repo per-NPC levers read in code — CVisualMemoryManager m_vision_speed (factor on get_visible_value, after the ai_vision_speed_boost cvar and the Lua override) + m_view_distance_factor (on object_visible_distance), neither reset by reinit/reload; CEnemyManager::evaluate biases (m_visible_enemy_bias_actor/npc replacing the 900/300 subtractions, m_hit_redirect_max/falloff replacing the flat CQB step) — **m_hit_redirect_* are never initialized by the ctor and set_visible_enemy_bias resets them**; make_enemy_visible → CMemoryManager::make_object_visible_somewhen; actor visibility state moved from CUIMotionIcon into CActor (m_npc_visibility/GetVisibility/SetVisibilityFromObject + free SetActorVisibility/GetActorVisibility, Lua get_actor_visibility); all three sense managers' remove_links now null-check instead of VERIFY |
| flows/ai-squad-logic.md | verified | function homes verified by grep: xr_logic configure_schemes/activate_by_section/issue_event/try_switch_to_another_section/initialize_obj; se_smart_terrain register_npc/select_npc_job/setup_logic/setup_gulag_and_logic_on_spawn; generic_object_binder:update. **Corrected agent: gamedata/scripts/bind_monster.script IS a live override.** Scheme internals agent-traced; base Lua = unpacked/ (read-only) |
| flows/engine-boot-frame.md | verified | spine homes verified: WinMain/WinMain_impl (x_ray.cpp), CEngineAPI::Initialize/InitializeNotDedicated, CRenderDevice::message_loop/on_idle/FrameMove, seqFrame/seqRender/seqAppStart/seqFrameMT registrators (device.h). Init order + secondary-thread bodies agent-traced inline |
| flows/render-frame.md | verified | entry + phase homes verified: CRender::Render (r4_R_render.cpp), phase_scene_begin/phase_combine/phase_combine_volumetric (r4_rendertarget*), render_lights (r4.h), accum_direct/R_sun. **Pass order + RT set + SSFX stages agent-traced** (large SSFX-modified R4 — verify per-task); resolves renderer.md's inferred deferred passes |
| flows/save-load.md | verified | spine spot-verified: game_sv_Single::save_game/load_game, CALifeStorageManager::save/load (both overloads, header→time→spawns→objects→registry order, rtc_compress, ALIFE_VERSION), CALifeObjectRegistry::save/get_object/load (Spawn_Write+UPDATE_Write, NOT STATE_Write; parent-only top-level + child recursion), CScriptBinder::save/load (try/catch swallows→clear), CGameObject::net_Save/net_Load. Sub-registry bodies (header/time/spawns) agent-traced inline |
| flows/alife-online-offline.md | verified | spine spot-verified: update_switch/CSwitchPredicate, switch_object/synchronize_location/try_switch_online/offline/add_online/remove_online, object try_switch_* (online_distance/offline_distance hysteresis from "alife" section: switch_distance*(1∓factor)), can_switch_* flags + creature health>0 guard, VERIFY(children.empty()) crash canary. Process_spawn/Perform_destroy internals agent-traced. Resolves INFERRED online-radius claim |
| flows/options-runtime.md | verified | spine spot-verified: options_builder check/track/page/group/import_into, options_modded_exes root GROUP, ui_options_modded_exes.on_xml_read (inserts GROUP + patches vanilla rows; get_opt_table for control rows, tree-walk-by-id for group inserts), dxml_core force_load do-block, UIOptions:On_Accept (cmd→exec_console_cmd+cfg_save; non-curr→axr_options.ltx; on_option_change). GetCurrentValue/CacheValue agent-traced (base game) |
| flows/dialog-trade.md | verified | spine spot-verified: CPhraseDialog::SayPhrase (toggle speaker, Action then per-edge Precondition, goodwill sort, ReceivePhrase; R_ASSERT no-phrase), CDialogScriptHelper::Precondition/Action 2-speaker (CheckInfo+functors, TransferInfo; DIALOG_UPGRADE param-table variant), CActor::RunTalkDialog/StartTalk homes, CTrade::TransferItem (GE_TRADE_SELL/BUY+set_money+eTradeSellBuyItem), GetItemPrice (condition^exp × relation-interp action factor × deficit, then R_ASSERTed trade_manager.get_*_discount). UI windows agent-traced; no live gamedata overrides (base-game functors/configs). REFINED 2026-06-11 (during F101): the player shop transaction is the Lua trade UI (ui_inventory intercepts CUIGameSP::StartTrade via actor_menu_inventory functor) — TransferItem/GetItemPrice are the engine tier underneath, see flows/trade-economy.md |
| flows/hud-ui-load.md | verified | spine spot-verified: CXml::Load→XMLLuaCallback (_G.COnXmlRead → LoadFromString re-parse, the DXML seam for every XML), CUIXmlInit::InitWindow (NavigateToNode+pos/size+InitAutoStaticGroup), InitAutoStaticGroup (recursive auto_static/frameline/framewindow/text → Init*→AttachChild), CUIWindow::AttachChild/Update(focus+recurse+CollectGarbage)/Draw(recurse, csUi guard). CUIGameCustom/CHUDManager per-frame owner agent-traced |
| flows/ui-scaling.md | verified | NO AGENT — whole path personally read: ui_defs.h (UI_BASE_WIDTH/HEIGHT = 1024/768 compile-time #define, private dup in XR_IOConsole.cpp), ui_base.cpp (OnDeviceReset scale + m_2DFrustum, ClientToScreenScaled* [Width/Height DIVIDE — names are backwards], AlignPixel, get_current_kx == scale.y/scale.x, screenmode 3 buckets, get_xml_name _16/_21 fallback chain, PushScissor [base units] vs clip-frustum stack [device px], pp_start/pp_stop), UIStaticItem::RenderInternal (both overloads, S2DVert::rotate_pt takes kx), UIWindow::GetAbsoluteRect + uiabstract.h waCenter, dxUIRender::PushPoint + FVF::TL = D3DFVF_XYZRHW + stub_notransform_t.vs (backend contract = absolute device px), Device_destroy.cpp seqDeviceReset pump. KEY FACTS: stretch is ANISOTROPIC (independent X/Y factors, 1.33x horizontal over-stretch at 16x9, 1.79x at 3440x1440) — kx correction is opt-in per call site (~2 dozen consumers: cursor, minimap+compass, map spots, UIMap pan/zoom/rotate [UIMapWnd un-applies for hit-test], cell icons/addons, rotated sprites). THREE draw classes: sprites fully stretched; **TEXT NOT SCALED AT ALL** (dxFontRender::OnRender uses atlas px directly, g_current_font_scale identity outside main-menu PP; layout inverse-maps metrics via ClientToScreenScaledWidth/Height) — size comes from which atlas rung is picked. **UPDATED 2026-08-17 (feat/font + feat/hires merged):** rung selection is now the single `GetFontTextureName` in GameFont.cpp (CFontManager::GetFontTexName and x_ray.cpp `_GetFontTexName` are forwards; the CGameFont ctor lambda is gone) — arbitrary `texture_h<N>` rungs alongside the legacy four, ranked by |log(authored/height)| with an atlas-existence probe per candidate; glyph advances scale by CGameFont::WidthScale (matters for the fsDeviceIndependent console/stat fonts). Still discrete, and base fonts.ltx still has nothing above texture1600 for ui_font_*, so ~93% design size at 1080p / ~70% at 1440p / ~47% at 2160p persists with base content (fixable by content: atlases + mod_system_*.ltx). CUIFrameWindow frames still NOT scaled (device px first, then tile at native texture px), but CUIFrameLineWnd now has two data-driven opt-ins — SetCapScaled (used by CUIComboBox) and crisp `border_l/t/r/b` bands from textures_descr (cut-table span placement; shipped on list lines, scrollbars, option sliders, edit/spin/combobox backgrounds). CUILines::Draw vertical-align indent moved into base units (pre-conversion). Cursor lives in base units (CUICursor::UpdateCursorPosition, uses clientWidth/Height NOT dwWidth/Height). Lua mirror in utils_xml.script (correct_ratio/screen_ratio==kx/is_widescreen). VERIFIED QUIRKS: CUIXmlInit::ApplyAlignX/Y are NO-OPS (align="r"/"b"/"c" attribute does nothing; only alignment="c"→waCenter works); AlignPixel floors LT only, not RB (±1px widths); no user-facing UI scale and no DPI awareness anywhere (the only ui_scale in tree is unrelated script_attachment property); one _21 XML exists game-wide (scopes_21.xml) |
| flows/animation-callbacks.md | verified | spine spot-verified: CHudItem::UpdateCL (motion-mark crossing prev==NULL&&curr!=NULL→OnMotionMark; m_dwMotionCurrTm>EndTm→OnAnimationEnd), OnMotionMark (_G.CHudItem__OnMotionMark), OnAnimationEnd (actor eActorHudAnimationEnd + eBore→Idle), motion_marks::pick_mark, CStepManager::update (per-leg offset_time → actor _G.CActor__FootstepCallback ID==0 else C++ FootStepCallback), LL_PlayCycle/LL_UpdateTracks homes. CBlend::update/anim_script_callback agent-traced |
| flows/weapon-hierarchy-addons.md | verified | spine spot-verified: inheritance tree from class headers (CWeapon→CWeaponMagazined→AutoPistol/CustomPistol→Pistol/Shotgun/SVD/Binoculars; AutomaticShotgun; MagazinedWGrenade; Knife←CWeapon; StatMgun outside tree), addon two-axis model (EWeaponAddonStatus × m_flagsAddOnState bits) IsScopeAttached/etc., CanAttach/CanDetach/Attach/Detach/DetachScope, InitAddons (scope optics/UIScope + silencer FX swap + koeffs), Load/Apply/Reset Silencer/ScopeKoeffs (clamp 0.01–2.0), GetScopeName (modular vs scope_name), CWeaponAmmo::Load/Get + box_size + m_magazine/m_ammoTypes. Network addon-sync (net_*/GE_ADDON_CHANGE), install_upgrade_addon, WGrenade GL/CRocketLauncher, StatMgun bodies agent-traced |
| flows/outfit-helmet.md | verified | spine spot-verified: type tree (CCustomOutfit←CInventoryItemObject + Exo/Military/Scientific/Stalker pure-config; CHelmet sibling), GetDefHitTypeProtection/GetHitTypeProtection (×GetCondition ×bone koeff), HitThroughArmor (fire-wound AP-vs-bone ballistics + flat subtraction one=1.0/0.1; Hit wears condition via GetHitImmunity), CEntityCondition::HitOutfitEffect (outfit→helmet) + HitPowerEffect (power_loss avg/3), OnMoveToSlot/OnMoveToRuck/ApplySkinModel (visual+hud swap, helmet/backpack eject, NV), passive restore tick in CActor, get_artefact_count Lua override, CExoOutfit pure-config confirmed, CHelmet has own Load/OnMoveToSlot/HitThroughArmor. CHelmet bodies + full Load param list + install_upgrade_impl + SBoneProtections::reload + immunities load agent-traced |
| flows/detector-artefact.md | verified | spine spot-verified: detector tree (CCustomDetector←CCustomDevice; Simple/Advanced/Elite; Scientific←Elite), CCustomDetector::shedule_Update→feel_touch_update(m_fAfDetectRadius), CAfList::feel_touch_contact (m_TypesMap section + GetAfRank<=m_af_rank gate), CAfList:CDetectList<CArtefact>, UpdateWork→UpdateAf (3 overrides exist), DETECTOR_SLOT/ARTEFACT_SLOT, CArtefact members (restore speeds, m_ArtefactHitImmunities, m_af_rank/GetAfRank, m_bCanSpawnZone, m_activationObj/m_detectorObj), ActivateArtefact/Action + GEG_PLAYER_ACTIVATEARTEFACT, belt-restore applied in CActor tick (×artefact condition, verified during F23). CCustomDevice battery/anim bodies, per-tier UpdateAf math, SArtefactActivation/SArtefactDetectorsSupport internals, leaf gimmicks agent-traced |
| flows/upgrade-system.md | verified | spine spot-verified: Manager::upgrade_install (upgrade_verify→can_install→pre_install_upgrade→install_upgrade(section)→run_effects→add_upgrade; FATAL on empty section) + init_install (installed_upgrades), CInventoryItem::install_upgrade/verify_upgrade (test=dry-run)/install_upgrade_impl base (cost/weight/flags/immunities)/has_upgrade/add_upgrade (GE_INSTALL_UPGRADE)/pre_install_upgrade (UnloadMagazine), process_if_exists (value=value+read, additive) vs process_if_exists_set (value=read, replace), Upgrade::can_install/run_effects + Group::can_install + UpgradeStateResult enum (result_e_parents/group/precondition_money/quest) exist, manager via ai().alife().inventory_upgrade_manager(). Element construct/load bodies, functor contracts, weapon/outfit install_upgrade_impl bodies agent-traced |
| flows/anomaly-zones.md | verified | spine spot-verified: hierarchy (CCustomZone←CSpaceRestrictor+Feel::Touch; Radioactive/NoGravity; CVisualZone→Hairs/Ameba; CMosquitoBald→Torrid/ZoneCampfire; CBaseGraviZone→GraviZone; CFryupZone is CScriptObject not a zone), UpdateWorkload state dispatch (idle/awaking/blowout/accumulate + postprocess effector + secondary hit), UpdateCL fast-mode, Power=m_fMaxPower×RelativePower, AffectObjects (per-frame guard → Affect per object), _G.CCustomZone_BeforeActivateCallback Lua veto, feel_touch_* existence. Per-subclass Affect bodies, UpdateBlowout timed sub-events, CreateHit packet internals agent-traced. CORRECTED: zones do NOT spawn artefacts in this path (only commented ALife) |
| flows/actor-condition.md | verified | spine spot-verified: delta-accumulator pattern (ChangeHealth/Power/Radiation/PsyHealth += into m_fDelta*), CEntityCondition::UpdateCondition (3 OnCritical*HealthLoss callbacks + last-chance-health m_fKillHitTreshold/m_fLastChanceHealth + apply/reset/clamp), UpdateRadiation (decay V_Radiation + damage V_RadiationHealth×dose), BleedingSpeed (Σwound×k clamp 0–10), CActorCondition::UpdateCondition (satiety/boosters/alcohol + weight-based ConditionWalk/Stand + max-power leak), UpdateSatiety (signed satiety_health_koef around critical → regen/starve + power regen). Boosters (ApplyBooster/BoostParameters/EBoostParams), eatable UseBy, ConditionWalk/Stand, UpdateHealth/Power bodies, tutorial-threshold _G.on_actor_* callbacks agent-traced. REFINED 2026-06-11 (during F103): actor psy is Lua-owned in the shipped corpus — arszi_psy zeroes type-4 hits in actor_on_before_hit and drives the engine bar via SetPsyBar; caveat + cross-link added |
| flows/relations-info.md | verified | spine spot-verified: RELATION_REGISTRY::GetAttitude (linear 5-component sum: personal+reputation+rank+community_goodwill+community_to_community), GetRelationType (attitude_neutal/friend thresholds from game_relations; NO_GOODWILL→neutral), GetRelation (worst-of-both-directions), CInventoryOwner::TransferInfo/OnReceiveInfo (KNOWN_INFO_VECTOR push_back, GE_INFO_TRANSFER event)/OnDisableInfo/HasInfo, CInfoPortion is a stub. REFINED 2026-06-11 (during F100): base CInventoryOwner::OnReceiveInfo fires NO callback (NPCs: poll has_info), but CActor::OnReceiveInfo override (actor_communication.cpp, read) DOES fire eInventoryInfo on new infos → bind_stalker → actor_on_info_callback bus; OnDisableInfo fires nothing. character relation tables, relation_registry_actions/fights, SetGoodwill clamp, Lua surface, entity_alive Hit/Die wiring agent-traced. REFINED 2026-06-12 (during F107): noted the Lua game_statistics layer (F107) as the normal-play driver of actor character_rank/reputation (counters→rewards + band-crossing news tips, bands from game_relations.ltx); cross-link added. REFINED 2026-06-12 (during F108): noted the Lua faction-relations layer (F108, game_relations.script — DoctorX dynamic relations + community goodwill matrix manipulation) as the gameplay tier over this engine attitude math; cross-link added |
| flows/tasks.md | verified | spine spot-verified: CGameTask::UpdateState (ordered gates: timeout if m_ReceiveTime≠m_TimeToComplete → CheckInfo(failInfos) → CheckFunctions(fail) → CheckInfo(completeInfos) → CheckFunctions(complete)), CheckInfo (Actor()->HasInfo all, break-on-first-missing = the F40 seam), MONOLITHIC model verified (no m_Objectives/SGameTaskObjective in GameTask.h — only m_completeInfos+functors), eTaskStateChange callback site, SetTaskState + CGameTaskManager::GiveGameTaskToActor/UpdateTasks/SetTaskState/ActiveTask exist. Manager bodies, Lua surface (give_task/task enums/CGameTask methods), map-location wiring, save/load registry, CheckFunctions combine, CALifeSmartTerrainTask (separate NPC-movement task) agent-traced |
| flows/item-base.md | verified | spine spot-verified: inheritance tree from headers (CInventoryItem:CAttachableItem+CHitImmunity mixin; CPhysicItem:CPhysicsShellHolder+CPHShellSimpleCreator; CInventoryItemObject:CInventoryItem+CPhysicItem diamond; CHudItemObject:CInventoryItemObject+CHudItem; CEatableItem:CInventoryItem), CPhysicItem::OnH_B_Independent (setVisible/Enabled+activate_physic_shell on drop) / OnH_B_Chield (deactivate on pickup), CAttachableItem::enable (CAttachmentOwner bone attach), CInventoryItemObject diamond calls both bases. Load param list, ChangeCondition/Hit math, net_Spawn agent-traced |
| flows/camera-pp-effectors.md | verified | spine spot-verified: CEffectorFall::ProcessCam(SCamEffectorInfo&) signature/lifetime, level.add_cam_effector (4 overloads) + add_pp_effector + remove_pp_effector Lua bindings. Engine manager (CCameraManager m_EffectorsCam list + m_EffectorsPP vector, SPPInfo additive composite — xrEngine), per-effector bodies (Bobbing/ZoomInertion/Shot, PostprocessAnimator .ppe, CPPEffectorDistance, CZoneEffector) agent-traced |
| flows/cscript-game-object.md | verified | spine spot-verified: CScriptGameObject ctor (m_game_object + R_ASSERT2), script_register chaining (script_register_game_object2(1(trader(move(instance))))), SetCallback → object().callback(type).set(functor) per-object seam, two file families (registration _script* vs impl .cpp/N + domain partials). BIND_FUNCTION* smart_cast macro bodies, method catalog, CScriptCallbackEx storage agent-traced |
| flows/consumables-explosives.md | verified | spine spot-verified: inheritance (CGrenade:CMissile,CExplosive; CMissile:CHudItemObject; CExplosive:IDamageSource mixin; CBolt:CMissile,IDamageSource; CEatableItem:CInventoryItem), CEatableItem::UseBy/CExplosive::Explode/CMissile::Throw exist. UseBy body (ApplyInfluence/ApplyBooster/decrement uses), Explode (fragments via BulletManager.AddBullet + blast-wave spatial sphere → ExplodeWaveProcess → GE_HIT), Throw (spawn_fake_missile+GE_OWNERSHIP_REJECT), grenade cook timer, BottleItem break, EXPLOSIVE_CHANGE Lua callbacks agent-traced |
| flows/pda-map-news.md | verified | spine spot-verified: CMapManager::AddMapLocation (xr_new<CMapLocation>), CPda:CHudItemObject+Feel::Touch, CActor::AddGameNews (game_news_registry push). CMapManager Update/Remove culling, CMapLocation spot/CalcPosition, CRelationMapLocation (relation→spot), CPda feel_touch contacts→AddRelationLocation, GAME_NEWS_DATA, give_game_news Lua agent-traced. Map Lua API mostly commented out (C++/task-driven). **UPDATED 2026-08-17:** context-menu section re-verified against the tree — GatherSpotsUnderCursor/clean_hint_title/AddHeader multi-spot merge, m_property_owners + TagSubmenuOwners, the m_UIScratchBox add-properties replay, gamedata `pda.set_map_spot_context` + replaced `pda.property_box_clicked` forwarding GetTAG, CUIPropertiesBox AddSubmenu/AddItem(text,id)/RemoveItemByTAG (the vanilla ShowSubMenu mechanism is gone; UIDemoPlayControl positions its own box), DrawHint suppressed while the box is shown, and modxml_fix_faction_spot_size |
| flows/alife-services.md | verified | spine spot-verified: CALifeTimeManager::game_time (m_game_time + time_factor×(dwTimeGlobal−m_start_time)) + set_time_factor rebase; CALifeSurgeManager.cpp is SPAWN-ONLY (spawn_new_spawns/fill_spawned_objects/spawn_new_objects — no update/kill/emission → confirms C++ stub, Lua surge_manager.script owns emission). CALifeSimulator ownership, CALifeUpdateManager::update (verified in alife-online-offline), CLocationManager, communication/interaction stubs, Lua surge agent-traced |
| flows/physics-objects-ragdoll.md | verified | spine spot-verified: CPhysicObject:CPhysicsShellHolder+CPHSkeleton, CCharacterPhysicsSupport:CPHSkeleton+CPHDestroyable + in_Die/CreateShell exist (ragdoll entry; m_eState esDead). CPhysicsShellHolder create/activate bodies, death sequence (KillHit/death_anims/imotion/EndActivateFreeShell/friction evolution), HangingLamp, ClimableObject, config params agent-traced. Key fact: ragdoll only on death (kinematics→shell), living hits push via movement controller |
| flows/vehicles-holder.md | verified | spine spot-verified: CCar:…CHolderCustom… (mountable), CHelicopter:CEntity,CShootingObject,CRocketLauncher,CPHSkeleton,CPHDestroyable,CHitImmunity,CExplosive (NO CHolderCustom — NOT mountable, AI aircraft), CActor::attach_Vehicle(CHolderCustom*,bForce) + eAttachVehicle callback. CHolderCustom interface bodies, Car subcomponents (wheels/doors/weapon/cameras/engine), ActorInput key-forward (except kUSE), helicopter movement/script, HolderEntityObject agent-traced |
| flows/monster-ai-control.md | verified | spine spot-verified: CControl_Manager (m_object CBaseMonster*, m_control_elems[type], capture/capture_pure/update_frame), CBaseMonster has IStateManagerBase* StateMan + CControl_Manager* m_control_manager (control()) — FSM+controls, NOT GOAP. Control-element bases, pure controls (animation/movement/direction/path_builder), ability controls (jump/run_attack/melee/threaten), Think/update_fsm loop, CMonsterEnemyManager agent-traced. Key fact: monster=FSM+control-manager vs stalker=GOAP |
| flows/stalker-execution.md | verified | spine spot-verified: CAI_Stalker owns CStalkerAnimationManager* m_animation_manager + stalker_movement_manager_smart_cover* m_movement_manager (smart-cover variant) + animation()/sight()/movement(). Update order, movement (game→level→detail path), sight (ESightType), CStalkerAnimationManager part selection, fire (g_fireParams, GetWeaponAccuracy rank+state dispersion, select_queue_params burst, aim-gates-fire), NPC-weapon-is-world agent-traced. **UPDATED 2026-08-17:** per-NPC combat levers + veto hooks read in code — CAI_Stalker aim params consumed per frame by CSightManager::Exec_Look and CSightAction::predict_object_position (**the formerly dead ai_aim_predict_time cvar is now the fallback, default 0.40s, so target leading changed globally**), set_fire_queue_scale applied in CStalkerActionCombatBase::select_queue_params after the weapon×distance band, can_kill_enemy/can_kill_member/fire_make_sense readers (false with no active item), g_fireParams() table, new virtual CActionPlanner::allow_action_switch (default true) overridden by CStalkerCombatPlanner → _G.CAI_Stalker__AllowCombatActionSwitch, and the best_cover veto _G.CAI_Stalker__AllowCoverRepick before on_best_cover_changed |
| flows/smart-cover.md | verified | spine spot-verified: CCoverManager owns m_smart_covers + add_smart_cover (registration) AND the auto cover-grid (compute_static_cover); smart_cover::object/loophole exist. Loophole structure (fov/actions/transitions), animation_planner/selector GOAP, occupy/fire sequence, cover_evaluators, Lua descriptions agent-traced. Key fact: smart cover (authored/animated/loopholes) vs cover-grid point (generated/geometric) |
| flows/script-npc-actions.md | verified | spine spot-verified: CScriptEntity m_bScriptControl/GetScriptControl()/ProcessScripts() (the control-flow fork GetScriptControl?ProcessScripts:Think, consistent with ai-planner seam). CScriptEntityAction sub-actions (movement/watch/animation/sound/object), bfAssign* drive same execution managers as stalker-execution, AddAction (obj:command), planner fully yields. Key fact: script control is a fork NOT a GOAP world property. Sub-action/bfAssign bodies agent-traced |
| flows/agent-squad-combat.md | verified | spine spot-verified: CAgentManager owns m_corpse/m_enemy/m_explosive/m_location/m_member/m_memory sub-managers (+ planner; NO danger sub-manager → danger is per-NPC), CAgentEnemyManager::fill_enemies/assign_enemies/distribute_enemies exist. Per-group ownership (group_hierarchy_holder seniority hierarchy), member combat mask (64-bit squad_mask_type), enemy pooling+victory-probability assignment, corpse/explosive/location reactions, alife_combat_manager inert agent-traced. Key facts: per online group, danger per-NPC, macro squad = Lua sim_squad |
| flows/special-actors.md | verified | spine spot-verified: class decls — CAI_Trader:CEntityAlive+CInventoryOwner+CScriptEntity+CAI_PhraseDialogManager (Think empty, dialog/trade/anim-only), CAI_Crow:CEntity (fly-idle/death-fall states + ragdoll on death), CPhantom:CEntity (stBirth→stFly→stContact state machine, kinematic, telepatic PsyHit). Behavior bodies + Lua binders (bind_trader/crow, phantom_manager) agent-traced. None uses GOAP/monster-FSM |
| flows/level-runtime.md | verified | spine spot-verified: CLevel:IGame_Level+IPureClient, CLevel::OnFrame exists (updates wallmarks pre-render). Owned managers (BulletManager/MapManager/GameTaskManager/seniority/spawn), per-frame sequence (ClientReceive→ProcessGameEvents→Objects.Update→physics/script/sound), Level_input routing (Lua→UI→game→entity), Level_load, CObjectList lifecycle (cl_Process_Spawn→g_sv_Spawn→net_Spawn→Update→ProcessDestroyQueue) agent-traced. RESOLVES xrGame.md residual (INFERRED) rest-of-Level.cpp |
| flows/wallmarks-particles.md | verified | spine spot-verified: CWalmarkManager/ScriptWallmarksManager/CScriptParticles classes exist + GMLib.GetMaterialPair is the impact seam (verified call sites: FireShotmark, poltergeist, physics_game, material_manager). SGameMtlPair (CollideSounds/Particles/m_pCollideMarks from material_pairs.ltx), FireShotmark bullet-impact (add_Static/SkeletonWallmark), CWalmarkManager::PlaceWallmarks explosion path, script place/play bodies, IRender interface agent-traced. Engine renders, game triggers |
| flows/renderer-resources-shaders.md | verified | spine spot-verified: CResourceManager::Create (lua/cpp fork + stub_default fallback + FATAL), _cpp_Create (creationGuard, 6-element loop iElement 0–5, _ParseList, _cpp_Compile×6, _CreateElement×6, dedup v_shaders→_CreateShader), _CreateElement/_CreateShader (equal()-dedup + RF_REGISTERED), _CreateVS/_CreatePS/_CreateGS (name-dedup → getShaderPath()+ext → FS.r_open stub_default fallback → Render->shader_compile), ownership maps (m_blenders/m_textures/m_vs/m_ps) + vectors (v_states/v_constant_tables/v_passes/v_elements/v_shaders) + creationGuard verified in header. Blender _cpp_Compile/r_Pass body, constant-table reflection, DX10/11 texture surface load agent-traced. **UPDATED 2026-08-17:** texture-lifetime section added, read in code — UnloadAllTexturesOnLevelUnload now skips flags.bUser / `$` / `ui\` names; EvictStalledTextures (USE_DX11 only, opt-in r__tex_evict_enabled, cursor-resumed bounded batch keyed on CTexture::dwLastUsedFrame stamped in Apply, refs>1 and age gates, DEBUG-only stats); CTexture::apply_load queues the re-load on the shared textures_load_tasks PPL group (bLoading guard) and binds apply_normal for the current call instead of loading inline; surface_set marks `$user$` as bUser; call site is CRender::Render every r__tex_evict_interval frames |
| flows/renderer-scene-graph.md | verified | spine spot-verified: CDSGraphManager:IDSGraphManager + RGraph (R_dsgraph::DynamicSceneRgraph) member, per-view instances GMBase (r4.h, VQ_HOM|VQ_SSA|VQ_FADE) / GMLight (light.h, flags 0) / GMCascade (R_sun.cpp) — same code builds main + shadow queues, traverse/r_dsgraph_insert_static/_dynamic/AddToRenderQueue/r_dsgraph_render_static/_dynamic/_graph/_graph_sorted exist, r_dsgraph_render_graph body read in full (per-pass std::sort/xr_parallel_sort@4096, high-key state/GS/HS/DS batch + low-key VS/PS/Constants/Textures, dynamic set_xform_world+apply_object, calcLOD→pVisual->Render). SSA/fade math, CSector::traverse portal walk, capture_static/dynamic bodies agent-traced |
| flows/renderer-post-process.md | verified | spine spot-verified: phase_combine is the entry called from CRender::Render (r4_R_render.cpp Target->phase_combine()), ORDERED stage sequence read directly from phase_combine.cpp call order (SSAO→ssfx_ao/il→combine→sky→ssfx_ssr→ssfx_water→render_water→ssfx_rain→RenderLast→render_forward→combine_volumetric→bloom→sunshafts→ssfx_fog→ssfx_motionblur→blur→ssfx_bloom|pp_bloom→dof→lut→gasmask→night/heat/fakescope→smaa→ssfx_taa→final-combine→hdr10→flares→pp), SSFX additions marked by in-source // [SSFX] comments, gating pattern verified (RImplementation.o.* + ps_* + IsSVPFrame skip). RT member names (rt_Generic_*/rt_Bloom_*/rt_ssfx_*), per-stage internals, SKIMMED shared-stage bodies agent-traced |
| flows/renderer-blender-compile.md | verified | spine spot-verified: CRender::blender_create (r2_blenders.cpp CLSID switch: B_DEFAULT→CBlender_deffer_flat, B_MODEL→CBlender_deffer_model, B_DEFAULT_AREF→CBlender_deffer_aref; many legacy CLSIDs return 0 on R4), _GetBlender name lookup, CBlender_Compile::_cpp_Compile / r_Pass (R2 vs DX10 GS-overload) / r_End / r_dx10Texture / r_dx10Sampler exist with claimed signatures. PassSET_*/StageSET_* render-state setters, uber_deffer/uber_shadow helper bodies, full r_Pass/r_End bodies agent-traced |
| flows/renderer-visibility.md | verified | spine spot-verified: class CHOM (Load/Render(CFrustum&)/visible(Fbox3&/Fbox2&/Fsphere&/vis_data&/sPoly&)) — visible(Fbox3&) body read (!bEnabled||camera-inside⇒TRUE else _visible), CPortal:IRender_Portal + CSector:IRender_Sector + CSector::traverse(CFrustum&&,CDSGraphManager&), CRender::detectSector/detectLastSector, class R_occlusion + occq_begin/occq_end/occq_get. occRasterizer software-raster internals, Render_DB/propagate, CSector::traverse body, light_vis vis_prepare/vis_update scheduling agent-traced |
| flows/renderer-backend.md | verified | spine spot-verified: CBackend RCache global (R_Backend.cpp `CBackend RCache;` + R_Backend.h class ECORE_API CBackend + extern), set_Element body read in full (state→PS→VS→GS→HS/DS/CS→Constants→Textures; set_Shader=set_Element(E[0])), CBackend::Render two overloads + set_Constants (dx10R_Backend_Runtime.h), _VertexStream::Lock/_IndexStream::Lock with D3D_MAP_WRITE_DISCARD(wrap)/WRITE_NO_OVERWRITE(append) verified. Per-stage cbuffer redistribution, dx10StateManager::Apply, R_Backend_xform matrix upload, constants.flush body agent-traced |
| flows/renderer-visual-types.md | verified | spine spot-verified: dxRender_Visual:IRenderVisual base + Fvisual:dxRender_Visual,IRender_Mesh / FProgressive:Fvisual / FHierrarhyVisual:dxRender_Visual class decls, CModelPool::Create body read in full (lowercase→Pool recycle(Spawn+erase)→Instance_Find reference→Instance_Load→Instance_Duplicate+Registry insert), Instance_Create/Instance_Duplicate/Instance_Load/Discard exist, Fvisual::Render/FProgressive::Render(float LOD) exist. OGF chunk parsing, FSkinned HW-skinning vertex setup, FTreeVisual wind constants, FLOD(disabled) agent-traced |
| flows/renderer-skeleton.md | verified | spine spot-verified: CKinematics:FHierrarhyVisual,IKinematics + CKinematicsAnimated:CKinematics,IKinematicsAnimated + CSkeletonX class decls, CKinematics::Bone_Calculate body read (CLBone per bone + recurse children w/ BONE_INST.mTransform as parent), BoneChain_Calculate shows bone-callback(physics) seam, CalculateBones(frame-gated)/LL_PlayCycle(2 overloads)/UpdateTracks/LL_UpdateTracks/CSkeletonX::_Render exist. Key dequant/mix (animated BuildBoneMatrix), bone-matrix constant upload (SkeletonX HW path), soft skinning agent-traced |
| flows/renderer-lighting.md | verified | spine spot-verified: class light + embedded CDSGraphManager GMLight (flags 0, matches F49) + export_, CLight_DB + add_light(2 overloads) + sun_original/sun_adapted, light_Package LP_normal/LP_pending, CRender::render_lights(light_Package&), CRenderTarget::accum_point(light*)/accum_spot(light*), phase_smap_spot(light*) exist. accum stencil/shadow-matrix bodies, SMAP_Allocator::push bin-packing, light_GI/render_indirect, vis_prepare/vis_update query scheduling agent-traced. Sun cascades = separate F55 |
| flows/renderer-sun-shadows.md | verified | spine spot-verified: CRender::init_cacades (m_sun_cascades.resize, sizes ps_ssfx_shadow_cascades.x/y/z=20/40/160), struct cascade with CDSGraphManager GMCascade (flags 0, same per-view pattern as GMBase/GMLight), CRender::render_sun_cascades→render_sun_cascade(u32) exist + called from r4_R_render.cpp then accum_direct_blend, CRenderTarget::accum_direct/accum_direct_cascade/accum_direct_f + phase_smap_direct exist. Ortho frustum-fit (R_sun_support FixedConvexVolume), texel stabilization, per-cascade PCF sampling agent-traced |
| flows/renderer-wallmark-engine.md | verified | spine spot-verified: class CWallmarksEngine + AddStaticWallmark(2 overloads)/AddSkeletonWallmark(2 overloads)/RecurseTri/BuildMatrix/Render exist, dxWallMarkArray:IWallMarkArray + AppendMark/GenerateWallmark, CKinematics::RenderWallmark (skeleton skinning) + r_dsgraph_render_wmarks (sorted path) exist. Static-vs-skeleton split verified by symbol. CDB clip/RecurseTri body, BuildMatrix projection, Render streaming/aging agent-traced. Engine renders, game (F44) triggers |
| flows/renderer-hw-device.md | verified | spine spot-verified: class CHW (HW.h) with pDevice ID3D11Device1*/pContext ID3D11DeviceContext1*/m_pSwapChain IDXGISwapChain1*/pBaseRT/pBaseZB, CHW::CreateD3D/CreateDevice/UpdateViews/Reset/DestroyDevice (dx10HW.cpp) exist, CHWCaps::Update exist, PRESENT verified in dxRenderDeviceRender.cpp (HW.m_pSwapChain->Present(interval,flags) + Present(0,DXGI_PRESENT_TEST) device-lost probe; CHW::selectPresentInterval picks vsync). Swapchain-desc, caps internals, fullscreen transitions agent-traced |
| flows/renderer-factory-bridges.md | verified | spine spot-verified: class dxRenderFactory (abstract base commented out), global wiring xrRender_R4.cpp (::Render=&RImplementation, ::RenderFactory=&RenderFactoryImpl, UIRender=&UIRenderImpl, DRender=&DebugRenderImpl) + declared in xrAPI.cpp, dxRenderDeviceRender::Create/Reset/End (End contains m_pSwapChain->Present — confirms F60 present site), dxUIRender::StartPrimitive/FlushPrimitive/SetShader exist. Create* factory pattern, per-bridge bodies (font/console/debug streaming) agent-traced |
| flows/renderer-particles.md | verified | spine spot-verified: CParticleEffect:dxParticleCustom + CParticleGroup:dxParticleCustom + dxParticleCustom:dxRender_Visual,IParticleCustom (particle IS-A visual, confirms F51), CParticleEffect::OnFrame/Render + CPSLibrary + CModelPool::CreatePE/CreatePG exist, scene-graph B2F bypass verified (r__dsgraph_build.cpp: if(sh->flags.bStrictB2F && !pVisual->dcast_ParticleCustom())) + group child expand via dcast_ParticleCustom. PAPI ParticleManager sim internals, billboard fill, group child-spawning agent-traced |
| flows/renderer-environment.md | verified | spine spot-verified: dxEnvironmentRender:IEnvironmentRender (RenderSky/RenderClouds), dxRainRender:IRainRender, dxThunderboltRender:IThunderboltRender, dxLensFlareRender:ILensFlareRender, CGammaControl::Update (→SetGammaControl DX10/11 / SetGammaRamp DX9 = output ramp, distinct from phase_lut), CRender::render_rain (R4 wet-surface) exist. Frame placement (phase_combine RenderSky/Clouds/Last/Flares) from F56. Geometry fills (sky dome/clouds/rain streaks/bolt/flare quads), draw_rain, LUT math agent-traced |
| flows/renderer-detail-grass.md | verified | spine spot-verified: class ECORE_API CDetailManager (Load/UpdateVisibleM/Render/cache_Update/hw_Render(light*)), class ECORE_API CDetail:IRender_DetailModel, CRender::Details global (r4.h) exist; hw_Render takes light* → grass renders into shadow passes (ties F54/F55). Slot-cache ring-buffer, terrain-trace decompress, VS-instanced wind render, soft fallback agent-traced |
| flows/renderer-r1-path.md | verified | spine spot-verified: R1 CRender:IRender_interface,pureFrame (Calculate/Render), getShaderPath()=="r1\\" (verified; CORRECTS agent — R4 getShaderPath=="r3\\" confirmed in r4.h, matches renderer.md), CLightProjector:pureAppActivate/CLightShadows/CGlowManager exist. REFERENCE/CONTRAST flow (active build is R4). R1 render-loop order, projector/blob-shadow bodies agent-traced |
| flows/renderer-r2-r3-paths.md | verified | spine spot-verified: getShaderPath R2="r2\\" / R3="r3\\" (r2.h/r3.h), get_generation ALL THREE (R2/R3/R4)==GENERATION_R2 (verified — confirms agent's surprising claim; not a path discriminator), CRender::Render in r2_R_render.cpp + r3_R_render.cpp, backend split (R2 stdafx d3dx9 only=DX9; R3 stdafx adds d3d10_1+d3d11). Comparative OOS flow; deltas-vs-R4 + accum/combine bodies SKIMMED/agent-traced |
| flows/renderer-3dfluid.md | verified | spine spot-verified: dx103DFluidManager::Initialize/Update/RenderFluid exist, R4 wiring verified (r4.cpp includes manager, o.volumetricfog=ps_r2_ls_flags.test(R3FLAG_VOLUMETRIC_SMOKE), FluidManager.Initialize(70,70,70)/SetScreenSize/Destroy), dx103DFluidVolume Type=MT_3DFLUIDVOLUME. CORRECTS agent "ACTIVE" claim — dx103DFluidVolume::Render has FluidManager.RenderFluid() COMMENTED OUT (only debug box) → effectively DORMANT. Sim-pass order, ray-march, blenders agent-traced |
| flows/gameplay-boot.md | verified | spine spot-verified in _g.script: start_game_callback body (randomseed→axr_main.on_game_start→sim_board.get_sim_board→task_manager.clear_task_manager→dialog_manager.fill_phrase_table→sr_light.clean_up→pda.add_quick_slot_items_on_game_start), Register/Unregister/Send/AddScriptCallback wrappers→axr_main, LoadScheme→schemes/schemes_by_stype. axr_main.on_game_start + class_registrator + game_setup agent-traced; gamedata/ axr_main override noted |
| flows/scheme-engine-core.md | verified | spine spot-verified in xr_logic.script (activate_by_section, switch_to_section, try_switch_to_another_section + full trigger taxonomy, issue_event, assign_storage_and_bind, pick_section_from_condlist head) + xr_motivator.script (motivator_binder:update full body: dead-path, squad-commander, state_mgr:update, npc_on_update, 500ms __tmr throttle, ph_door, combat overrides). configure_schemes/initialize_obj/parse_condlist/generic-schemes agent-traced; condlist eval → flows/condlist-library.md (F67) |
| flows/db-registry.md | verified | db.script READ-IN-FULL (no gamedata override): storage/actor/actor_proxy + all category tables + add_obj/del_obj/add_actor/del_actor/add_stalker(dedup)/add_zone/add_anomaly/add_anim_obj/add_heli/add_smart_terrain/add_bridge/add_enemy. add_obj requires storage[id] pre-created by binder reinit (verified invariant). Binder call-sites agent-traced (→F88) |
| flows/condlist-library.md | verified | registry mechanism verified (bare global fns in xr_conditions.script namespaced as xr_conditions[name]; grep confirms no `function xr_conditions.NAME` — CORRECTS agent's syntax claim) + signature (actor,npc,p) + full evaluator call sites in xr_logic.pick_section_from_condlist (type 1-5, xr_conditions[]/xr_effects[] calls, missing-functor warning). Sampled fns is_actor_enemy_to_faction/is_npc_enemy_to_faction/toolkit_task_done read; rest of catalog + *_addon `script` dispatcher agent-traced |
| flows/logic-ltx-switching.md | verified | parse spine verified in xr_logic: parse_condlist (memoized, comma-split, {}→idx2/%%→idx3), extract_conditions (sign→type 1/2/3/4/5, params :-split tonumber-or-string), cfg_get_switch_conditions (line router, trailing-digit strip, full on_* set), cfg_get_overrides head. utils_data.get_scheme_by_section agent-traced; authoring format reconstructed-from-parser |
| flows/state-manager-core.md | verified | spine verified in state_mgr: global set_state/get_state wrappers, state_manager:set_state (state_lib validation, look/callback/extra, fire-exit weapon idle, idempotent), state_manager:update (callback-timeout + planner:update GOAP loop); state_lib.states entry structure read (weapon/movement/mental/bodystate/animstate/animation/direction). bind_manager head only; goap_graph + sub-controllers (F71) agent-traced; motivator seam verified via F66 |
| flows/state-mgr-controllers.md | verified | animation marker machine verified in state_mgr_animation: MARKER_IN/OUT/IDLE=1/2/3 constants + animation:animation_callback read in full (IN→advance into[]/set current_state=target; IDLE→next_rnd; OUT→advance out[]/set current_state=nil) + select_anim structure + set_state fast_set bypass. The 6 simple channels (movement/mental/bodystate/weapon/direction/animstate) eva/act pattern agent-traced (uniform: compare state_lib field → engine set_*); animstate = same animation class on animstate_list |
| flows/smart-terrain-core.md | verified | bridges verified in smart_terrain: setup_logic (job.ltx or self.ltx; xr_logic.configure_schemes/determine_section_to_activate/activate_by_section with job.section) + setup_gulag_and_logic_on_spawn (resolve smart via m_smart_terrain_id≠65535; no-job→select_npc_job, has-job→setup_logic; else→xr_logic.initialize_obj). on_register/register_npc/select_npc_job/load_jobs/STATE_Write/Read confirmed present, roles agent-traced. No gamedata override |
| flows/gulag-jobs.md | verified | job_info_by_job_type_id catalog read in full (types 0-20 w/ job_type+prior+precondition: surge=60, heli_hide=100, walker=15, camper=45, beh_trader/mechanic=59...) + get_job_prior/precondition/type exclusive-vs-job_type_id fork. load_job builder + init_*_jobs generators + add_exclusive_job + xr_gulag.get_npc_smart agent-traced. dynamic_ltx in-memory (not on disk). No functional gamedata override |
| flows/scheme-walker-patrol.md | verified | canonical scheme template verified in xr_walker: set_scheme (assign_storage_and_bind + cfg_get_switch_conditions + path_walk/look/sound_idle/use_camp/suggested_state), add_to_binder (motivation_action_manager add_evaluator/add_action, preconditions alive/danger=false/enemy=false/need_walker, effects, subscribe_action_for_events, gates xr_actions_id.alife), evaluator→xr_logic.is_active, execute entry. patrol (PatrolManager commander/follower) + remark (stationary state machine) deltas agent-traced. No gamedata override |
| flows/scheme-combat.md | verified | xr_combat verified: set_combat_type (combat_type condlist→script_combat_type, "nil"→nil), evaluator_check_combat (script_combat_type~=nil gate), add_to_binder grabs manager:action(stalker_ids.action_combat_planner) to gate engine combat. combat_ignore (set_enemy_callback is_enemy + overrides), xr_danger, xr_hit, zombied/camper/monolith styles agent-traced. No gamedata override of all 7 |
| flows/companion-system.md | verified | recruit verified in axr_companions: setup_companion_logic (give npcx_is_companion + load beh_companion.ltx + xr_logic.configure_schemes), remove_from_actor_squad disables info. xr_companion verified: set_scheme (st.behavior=beh_walk_simple), add_to_binder, action execute branches beh_walk_simple/beh_wait_simple. Command model = npcx_beh_* info portions (verified by give/disable pattern); setters/keybinds/teleport agent-traced. **No gamedata override (verified by listing — companions ship in unpacked here)** |
| flows/scheme-camper-cover.md | verified | xr_cover verified: action_cover:activate_scheme best_cover(self_random_position,enemy_random_position,cover_dist,1,150) vertex selection + set_scheme (smart/path_center/radius_min/max/anim). xr_camper agent-traced (sniper scan, radius lock, evaluator_close_combat gates combat planner). No gamedata override |
| flows/scheme-animpoint.md | verified | architecture verified in xr_animpoint: two-action split action_reach_animpoint:execute + action_animpoint:execute; animpoint:calculate_position/start/update/set_scheme confirmed present. calculate_position (smartcover description lookup) + update (camp director/listener via assoc_tbl) + predicates bodies agent-traced. smartcover seam = F33. No gamedata override |
| flows/scheme-sleeper-wounded-death.md | verified | wounded combat-gate verified: evaluator_can_fight→load_var(npc,"wounded_fight")~="false", Cwound_manager:update saves wounded_fight + :eat_medkit present. death verified: death_callback records db.storage[victim:id()].death.killer=who:id() or -1. sleeper symbols (set_scheme/reset_scheme/callback) present. help_wounded/eat_medkit(callback,plugin-gated)/smartcover agent-traced. No gamedata override of all 6 |
| flows/scheme-meet-abuse-bribe.md | verified | xr_bribe verified: at_peace (ceasefire+safe-distance, consumed by xr_combat_ignore.is_enemy F76), set_bribe (M_DATA per-faction), npc_on_hit_callback (break on hit). xr_meet.process_npc_usability called by xr_motivator:update (verified F66); Cmeet_manager distance tiers + notify_on_use→abuse + CAbuseManager agent-traced. No gamedata override |
| flows/scheme-gather-corpse.md | verified | xr_corpse_detection HasValuableLoot table verified (marked on death, gates evaluator). corpse_already_selected claim + Items[] gather claim + cstackprevent throttle + loot transfer agent-traced. CORRECTION: xr_box.script = ph_item_box stash manager NOT a scheme; NPC box scheme = axr_npc_vs_box (feel_touch + shoot). No gamedata override |
| flows/scheme-sensors-misc.md | verified | classification verified: most are NOT schemes — xr_sound/xr_hear = services hooked from xr_motivator, xr_zones = zone binder, xr_detector = ACTOR-only, xr_reach_task = the only real scheme. **xr_weapon_jam GAMEDATA OVERRIDE verified (only batch-6 file with gamedata copy)** — live version uses npc_get_misfire_probability callback (t.ret_value=1) vs unpacked unload_magazine. Bodies agent-traced |
| flows/restrictors-active-sr.md | verified | sr_teleport READ-IN-FULL (action_teleport: inside→pp effector 2006+timer→after timeout weighted-random set_actor_position/direction+tinnitus; point/look/prob config) + sr_crow_spawner READ-IN-FULL (crowkiller: throttled alife_create m_crow up to max_crows_on_level, >100m from actor, off during surge/storm). sr_monster (fake_monster: spawn/reveal st.monster on actor inside+appear sound), sr_cutscene (cam_effector_set+action_cutscene: level.add_cam_effector camera_effects\*.anm id 210408 + effector_callback, condlist effect sets), sr_camp (CCampManager storytelling s_story_1..11/guitar/harmonica, instantiated by bind_camp F89 — manager not scheme) characterized by structure+key calls; bodies agent-traced. All F66 scheme on stype_restrictor (except sr_camp manager). No gamedata override |
| flows/restrictors-passive-sr.md | verified | sr_idle READ-IN-FULL (the template: set_scheme=assign_storage_and_bind+cfg_get_switch_conditions; add_to_binder=action+subscribe_action_for_events; action:update=effect+try_switch_to_another_section). 8 others characterized by class/function structure + key engine-effect calls grepped: sr_silence (CSilence_zone music suppress), sr_no_weapon (zone_enter/leave force holster), sr_timer (action_timer HUD time2str), sr_postprocess (PPEffector+set_snd_volume), sr_light (check_light/check_stalker NPC torches by level/time), sr_particle (particles_object mode 1/2), sr_psy_antenna (PsyAntenna psy_hit/phantoms/pp distortion — mechanism verified, body agent-traced), sr_deimos (CDeimos intensity→add_pp/cam_effector — mechanism verified). All = F66 scheme on stype_restrictor; effects via F31 level.add_pp/cam_effector/set_snd_volume. No gamedata override |
| modlayer/gameplay/restrictors-sr.md | map | sr_* restrictor-scheme map; structure verified by listing unpacked/scripts/sr_* (14 files) + F91; passive (F91 ✅) vs active (F92 ✅) split; sr_camp=manager used by bind_camp; mechanics delegated to flows/restrictors-passive-sr.md (F91) + flows/restrictors-active-sr.md (F92) |
| flows/physics-object-schemes.md | verified | NO AGENT — 10 small ph_* scripts + ph_door READ-IN-FULL (joint door open/close force+fix, fastcall completion, NPC auto-open via xr_motivator→try_to_open_door/db.level_doors, not_for_npcs blacklist, lock_door_for_npc; button on_press; codepad ui_numpad on_code/on_check_code; idle holder; hit one-shot strike; on_hit generic listener via enable_generic_schemes — legacy ph_on_hit.set_scheme call is a nil-crash dead path, 0 base refs; ph_death registers scheme "ph_on_death"; oscillate joint force swing; ph_sound rewired to Zone FM radio, destr pstor) + ph_car/ph_minigun spine read (CCar via get_car; usable-vs-scripted modes; headlights via mob_capture+action F37; job-mode targeting via SIMBOARD.smarts_by_names+npc_by_job_section+enemy_id+combat_ignore; GetfHealth<=0→destroy_car→vehicle_on_death_callback+on_death_info→section nil); movement/fire interiors NOT read. set_fastcall=CScriptGameObject (script_game_object_use.cpp). ph_appforce UNREGISTERED (dead). LoadScheme stype_item rows verified in modules.script; base-config usage counted (door 235/idle 58/button 21/code 5/sound 2/rest 0). No gamedata override |
| modlayer/gameplay/physics-schemes-ph.md | map | ph_* family map; structure verified by listing unpacked/scripts/ph_*+phantom_manager (13 files) + F93 registration rows; scheme-vs-unregistered-vs-binder classification; mechanics delegated to flows/physics-object-schemes.md (F93 ✅) |
| flows/monster-schemes.md | verified | NO AGENT — all 11 mob_* files READ-IN-FULL (walker: capture+action queue patrol, patrol_path_in_point→waypoint_callback flags r/c/s/sig/b/t/a via utils_obj.path_parse_waypoints + move_mgr.choose_look_point, recapture-if-released; home: engine set_home at gulag-smart vertex or path_home, no capture, 230 base refs; remark: anim+sound action sequence, dialog_cond talk gate — snd path arg-mismatches mob_sound.pick_sound_from_set→abort; jump: look→jump(point,factor)→"jumped"+release; camp: UNREGISTERED dead FSM w/ db.camp_storage claiming; combat: DORMANT — no binder/engine caller for combat_callback (no callback.combat in luabind enum); death: bind_monster issues to st.mob_death → db.storage death.killer record, unknown-killer branch nil-crashes; trader: CAI_Trader anim/sound callbacks + global/head_animations tables + db.trader; trade: trade_perform_operation→all_money/exchanged + ActorMenu_on_trade_started/closed bus; state_mgr helper bloodsucker invis/vis only else abort; sound: legacy voice bank). LoadScheme stype_mobile×6 + stype_trader×2 verified; bind_monster (gamedata override) + bind_trader callback wiring verified. No gamedata mob_* override |
| modlayer/gameplay/monster-schemes.md | map | mob_* family map; structure verified by listing unpacked/scripts/mob_* (11 files) + F94 registration rows; scheme-vs-dead-vs-helper classification; stype_mobile/stype_trader binder routing; mechanics delegated to flows/monster-schemes.md (F94 ✅) |
| flows/helicopter-ai.md | verified | NO AGENT — bind_heli/heli_move/heli_fly/heli_look/heli_fire/heli_alife/heli_snd READ-IN-FULL (binder: is_story_heli spawn-ini [logic] fork, check_health flame 0.1/die 0.005 + smart unregister, on_hit→heli_enemy_flag actor-pstor/npc-storage = becomes-valid-enemy mechanic, stype_heli setup_gulag_and_logic_on_spawn, heli_spawn option self_destruct; heli_move: only stype_heli scheme — patrol/path_look=actor/stop_fire hover/coord nodes pt1..N+path_end loop|reverse|switch/retreat_mode bounding-volume exit, weapon params onto engine fields, pstor persistence; heli_fly Lagrange calc_point + braking-distance planner; heli_fire actor|all|story-id pick from db.heli_enemies; heli_alife: faction-sim REACH/STAY-6800gs FSM on simulation_objects top-5 props[player_id] scoring + smart register_npc population + force_target_destination help API [axr_radio_in_heli]). heli_combat SPINE read (is_enemy community+heli_enemy_flag+wounded/ignore gates, 25s target keep, flyby⇄round⇄search dist/vis transitions, round mgun burst toggle, exclusive-job opt-out, surge/night off); round/search/flyby/retreat bodies + heli_alife_warfare NOT read. heli_snd consumed only by xr_effects (grepped). No gamedata override. REFINED 2026-06-11 (during F105): the heli_alife.update→heli_alife_warfare.update swap lives only in dead override_functions — warfare helis created by heli_alife_warfare, updated by stock heli_alife; note + cross-link added |
| modlayer/gameplay/vehicles-heli.md | map | vehicles Lua-tier map; heli stack routed to F95; **F96 (cars) documented here (merged)**: bind_car+se_car READ-IN-FULL (car_binder = generic_physics_binder clone running stype_item → ph_car F93; hit also issued to st.ph_on_hit; se_car = F84 passthrough + keep_saved_data_anyway; driving = engine F29); no gamedata override |
| flows/task-system-lua.md | verified | NO AGENT — task_manager.script + task_objects.script READ-IN-FULL (CRandomTask singleton give/update/force/task_callback; CGeneralTask: tm_*.ltx wildcard config parse, give_task builds CGameTask + add_complete/fail_func("task_manager.task_complete/fail") = the F38 functor seam, check_task 500-1000ms throttle re-runs title/descr/target functors → live text+change_map_location, condlist_N → last_check_task, status_functor counters; task_callback on engine task_state callback → on_fail/on_cancel/on_complete effects + give_reward via dialogs.relocate_* to get_speaker() [dialog-only] + repeat_timeout reset stage 255 + cooldown removal w/ gossip news; check_level guider spots CoP-trio-only; USE_MARSHAL m_data save_state/load_state vs legacy packet path). VERIFIED BUG: reward_item reads the reward_money ini line. Wiring grepped: tick=bind_stalker:update, callback=bind_stalker_ext.actor_on_task_callback, give=xr_effects.give_task, save/load=bind_stalker. task_functor/task_status_functor structure + generic condlist/target_condlist verified; ~50 story functor bodies NOT read. No gamedata override |
| modlayer/gameplay/tasks-lua.md | map | Lua task family map; pieces classified (core F97 ✅, tasks_* dynamic types F98 ✅, axr_task_manager F114 ⬜); config tier tm_*.ltx wildcard; mechanics delegated to flows/task-system-lua.md + flows/dynamic-task-types.md + flows/tasks.md (F38 engine half) |
| flows/dynamic-task-types.md | verified | NO AGENT — plugin pattern verified: tasks_clear_map READ-IN-FULL (xr_conditions.task_stage_is/mutants_in_map* + task_functor.mutants_in_map_target injections; mutants_in_map scans SIMBOARD.squads + sets force_online=true side effect) + end-to-end config consumer verified (tm_deadcity.ltx merc task: target_functor=mutants_in_map_target, target field = functor argument "abuse", set_task_stage stepping) + tasks_bounty head (assignment-style injection xr_effects.setup_bounty_task / task_functor.general_bounty_task; SIMBOARD squad scan w/ faction list + map blacklist + base-smart filter) + function-shape survey of all 22 + callback wiring grepped (tasks_defense/fetch: RegisterScriptCallback actor_on_first_update/save_state/load_state/on_before_surge/on_before_level_changing). ~19 per-type bodies NOT read (leaves on the pattern). No gamedata override |
| flows/dialog-system-lua.md | verified | agent-researched, spine personally verified: dialog_manager.script READ-IN-FULL (add_dialog AddPhrase/AddPrecondition/AddAction bridge w/ findfunction validation; fill_phrase_table ← _g.start_game_callback; init_start_dialogs/init_new_dialog tree builds; calculate_priority full scoring incl. wounded-XOR + Tronex condition/important; action once→pin-255 + ignore_once + told-per-category; disabled_phrases cleared by xr_motivator use callback) + dialogs.script census + key bodies (who_is_npc/break_dialog/update_npc_dialog/relocate_*/task-offer set/npc_is_trader→start_trade) + attachment chain (character_desc #include character_dialogs.xml → start_dialog hello_dialog; no_guide variant) + dialog_manager.ltx shape/counts (854 sections; place=0 dormant; no smart fields). VERIFIED BUGS: precondition_no_more indexes nil-global PRT_subtable (errors when phrases available); anomalies smart gate INVERTED + dormant (corrects agent); once pins not hides (corrects agent). Level packs/companion/broker/surrender bodies agent-traced. No gamedata override |
| modlayer/gameplay/dialogs-info.md | map | dialogs & character-info family map; structure verified via F99/F100 (scripts + 26 dialogs*.xml + character_dialogs include chain + profile/info/encyclopedia data tier); mechanics delegated to flows/dialog-system-lua.md (F99 ✅) + flows/characters-info-encyclopedia.md (F100 ✅) + flows/dialog-trade.md (F18) — domain complete |
| flows/characters-info-encyclopedia.md | verified | NO AGENT — info_portions.script READ-IN-FULL (info-switch condlists on actor_on_info_callback; warlab task chain; on_game_load info resets) + ui_pda_encyclopedia_tab.script spine read (parse_categories mode-gated; unlock_article/note; actor_on_interaction bus + 7.5s-delayed leave_dialog/item_take/feeling_anomaly callbacks w/ 2h-in-zone anomaly gate + retroactive faction/level/inventory sweep; m_data.game_statistics persistence; encyclopedia_infos.xml EMPTY = vanilla XML article system dormant; UI classes SKIMMED) + engine selection read (CSE_ALifeTraderAbstract::specific_character: direct vs template pick w/ no_random skip, class match, RANK/REPUTATION_DELTA, used-id exclusion via CSpecificCharacterRegistry, default_for_community fallback + VERIFY3; CCharacterInfo::Init/accessors) + data shapes sampled (system.ltx [profiles]/[info_portions]; general faction packs #included not listed). DROVE the F40 correction (actor eInventoryInfo). No gamedata override |
| flows/trade-economy.md | verified | NO AGENT — trade_manager.script + axr_trade_manager.script READ-IN-FULL (trade_init via xr_logic `trade=` assignment, default trade_generic; update on xr_motivator use-callback + bind_trader per-frame for script_trader; 4-condlist re-pick → engine buy/sell_condition/CTradeParameters exports; buy_supplies eco-restock-gated → CInventoryOwner::buy_supplies/CPurchaseList read; m_data.trade_manager persistence; get_buy/sell_discount R_ASSERTed from trade2.cpp × eco factors; axr: npc_on_item_take intake vs [buy_sell] keep/consider/restock/mults, xr_conditions/xr_effects job glue + beh_gulag_jobs `trade` signal, sell=transfer+conjured give_money or alife_release w/o seller, buy=ammo-only [bw clause redundant], mechanic upgrades ≤5/visit money>500 via inventory_upgrades.force_upgrade) + Lua UI takeover chain read (CUIGameSP::StartTrade → actor_menu_inventory functor → ActorMenu_on_before_init_mode → ui_inventory.start/TMode_*; utils_item.get_item_cost cost×mult×discount×(cond·0.9+0.1)^exp + get_item_trade_status 1-4; TMode_Buy/Sell pure-Lua transfer+give_money) + LTX shapes (trade_generic/sidorovich goodwill supplies ladder; presets: valueless line = blacklist VERIFIED in status fn; discounts buy 0.1-0.4/sell 0.7-1.5) + game_difficulties eco factors. VERIFIED BUG: both sell loops `for i=count,1` missing -1 step — body runs only when count==1. No gamedata override |
| modlayer/gameplay/trade-economy.md | map | trade & economy family map; structure verified via F101 (36 trade_*.ltx + 2 presets + gulag_job_trade_buy_sell + 5 scripts); mechanics delegated to flows/trade-economy.md (F101 ✅) + flows/dialog-trade.md (F18 engine tier) + flows/monster-schemes.md (F94 trader schemes); F102 ✅ / F114 adjacency ⬜ |
| flows/item-behaviors-lua.md | verified | agent-researched, spine personally verified: _g.Parse_ITM/_ITM registry (repair←repair_tool, cook←cooking_tool, device←bind==item_device.bind, multiuse←max_uses; verified body) + itms_manager registrations/before_use boost-veto+required_tool/use hardcoded specials+multiuse_r recharge/drag combos (arty_container.ltx + craft.ltx item_combination)/ItemProcessor ammo branch + item_cooking OnCook/GetAvailFuel + item_parts.disassembly_weapon (status-2 attachments, random(100)<=chance+con*100 roll, se_save_var parts persistence) + item_device device_binder class + consumption-table build + item_knife whole file + item_mine header/registrations + both item_weapon censuses + gamedata fork diff (ammo_aggregation_full next-frame batching, per-weapon overheat/smoke, get_object_by_id) all read directly. VERIFIED BUG FAMILY: se_load_var(item_id,…) nil-global ×4 (itms_manager take, axr_companions ×2, death_manager guard dead); cooking is_campfire global typo benign (GetAvailFuel returns false at campfire); ItemProcessor debug new_con nil; item_mine UNUSED-but-live (axr_main auto-runs on_game_start). REJECTED 2 agent claims (knife precedence non-bug; agent's "only line 22" weapon diff — real diff 393 lines). repair/radio/artefact/recipe/backpack/money/tent/map_kit internals agent-traced |
| modlayer/gameplay/items-lua.md | map | item-behavior family map; structure verified via F102 (14 item_* + itms_manager + 8 items/settings LTX); registry/hub/cluster classification; mechanics delegated to flows/item-behaviors-lua.md (F102 ✅) + flows/inventory.md (F8 engine tier) + flows/binder-architecture.md (F88 device binder) |
| flows/actor-survival-status.md | verified | NO AGENT — actor_status + actor_status_sleep + actor_status_thirst + arszi_psy READ-IN-FULL; actor_effects spine READ-IN-FULL (init/settings, parse_anims wpn_fx inheritance, Update_ItemSwap, Update_Animations loadstring scheduler, play_item_fx/play_continuous_effect, breath sounds, full callbacks section incl. before_fire weapon-click veto ≤0.014 + on_key_release features), overlay bodies (Update_Rad/Mask/Fog/Breathing/Bleeding/Blood/Impact, HUD_fog/HUD_mask, Hit_* anims) SKIMMED. Grep-verified: game_difficulties drives toggle_feature; ui_sleep_dialog emits actor_on_sleep + hosts sleep_forced; BoosterID in _g; eat_sleepiness/thirstiness in items_food/drink/drugs/medical/monster LTX; surge_manager polls current_safe_cover SetEvent; gamemode_azazel sole arszi_psy consumer; [vodka] LTX shape sampled. KEY FACTS: arszi_psy zeroes type-4 hit power in actor_on_before_hit and debits its own meter (engine psy bar = SetPsyBar display only); zombification dead code (file-local false); sleep/thirst twin meters difficulty-toggled w/ add_indicator injection. VERIFIED QUIRKS: shared blur ppe id 39568 cross-suppression; get_satienty typo = real API; scan_safe_zone rewrite dead (old runs); force gates random(100)<=10001 always true; controller tube falloff 2D. No gamedata override of any file incl. actor_effects.ltx |
| modlayer/gameplay/actor-survival.md | map | actor-survival family map; structure verified via F103 (4 survival scripts + actor_effects + plugins/actor_effects.ltx + toggle/sleep-UI wiring) + F104 (actor_menu/actor_menu_inventory/ui_inventory/utils_ui); mechanics delegated to flows/actor-survival-status.md (F103 ✅) + flows/actor-menu-inventory-ui.md (F104 ✅) + flows/actor-condition.md (F30 engine tier) + flows/item-behaviors-lua.md (F102 use pipeline) — domain complete |
| flows/actor-menu-inventory-ui.md | verified | agent-researched, spine personally verified: entry chain READ (enable_feature hardwired true; ActorMenu_on_before_init_mode → start + flags.ret veto; overrides() replaces _G.hide_hud_inventory/_G.GetActorMenu/engine hide functor/ui_companion_inv.start; change_last_mode re-emits actor_menu.actor_menu_mode) + actor_menu.script & actor_menu_inventory.script READ-IN-FULL (mode mirror→info portions, set_msg/set_fade_msg HUD API on not_enough_money_mine static; 4 mode functors + EDDListType drag-drop + always-true policy functors) + engine callers grep-read (UIGameCustom/UIGameSP/UIActorMenu_script R_ASSERT/UIActorMenu_action/Inventory.cpp BeltWidth) + self.properties table & InitProperties body + Name/Mode/Cont/DB/Action_Custom (useN_functor doubles as precondition; useN_action_functor executed; defaults inventory-mode/actor-bags) + LMode_TransferInfo (corpse custom_data [logic] known_info → give_info) + quick slots = console slot_0..3 section state + npc_on_use/physic_object_on_use auto-open + monkey-patch sites READ (gamedata callbacks_gameobject TMode_Sell/Buy/On_Item_Exchange wrappers → actor_on_item_sell/buy synthesis + CInventory_BeltWidth injection; aaaa_script_fixes_mp Action_Move nil guard, fork identical in region). CORRECTED 2 AGENT CLAIMS: add_stats_table unusable (shares is_not_empty guard with prepare_stats_table — verified both); "no external callers" false (monkey-patch surface exists). VERIFIED QUIRKS: On_CC_Trasfer typo load-bearing; helm-cast printe wrong-var + continues on nil; quick_item accidental global; xr_meet_*_closed dead state; Add_Shadow double Show. Bag parsing/mode bodies/drag-drop branches/RMode delegation/UICellItem-Container anatomy agent-traced w/ line evidence. No gamedata override of the 4 core scripts |
| flows/warfare-metagame.md | verified | agent-researched, spine personally re-verified: _g.IsWarfare (axr_main.config new_game_warfare OR alife-storage enable_warfare_mode ← ui_mm_faction_select writes) + warfare.on_game_start UNCONDITIONAL registration + _G.WARFARE=true optimistic + resetter TimeEvent 60s unregister + actor_on_update IsWarfare early-exit + sim_squad_warfare squad.__lock=true per tick → sim_squad_scripted.generic_update native __lock bail (read both sides) + override_functions DEAD CODE (grep: no caller unpacked or gamedata → squad nulling redundant, heli_alife.update→heli_alife_warfare.update swap never installed; create_heli_weak/set_target still called direct from warfare_faction_control) + check_owner body (owned smart flips only when owner absent from defender tally; monster→none; manual point; influence refund on friendly capture; 300s save stickiness; keep_last_base; >25-squad spam TODO) + squad_on_npc_death rewards (money+rank×weight+influence, enemies-only, registered-only) + update_faction locals + warfare_faction_control head/possess-leader body/purchase debits. CORRECTED 2 AGENT CLAIMS (conditional registration; method-nulling integration). VERIFIED BUGS: calculate_target_density linked-loop indexes `level` not `lvl` (home smarts duplicated, linked ignored); update_faction patrol_squads undefined-global nil store over initialized 0. Faction/level tick internals, invasion/patrol state machines, fog-of-war spots, UI bodies agent-traced w/ line evidence. faction_expansions.script unread. No gamedata override of any warfare file |
| modlayer/gameplay/warfare.md | map | warfare family map; structure verified via F105 (12 warfare*/​*_warfare scripts surveyed, 8-piece classification); mechanics delegated to flows/warfare-metagame.md (F105 ✅) + flows/scripted-squad.md + flows/smart-terrain-core.md + flows/offline-combat.md + flows/helicopter-ai.md (F95 heli split) |
| flows/game-modes.md | verified | NO AGENT — game_difficulties + game_setup + gamemode_agony/azazel/ironman/timer READ-IN-FULL; ui_mm_faction_select flag-seeding region + item_artefact toggle/before-hit gating + CEntityCondition::ChangeBleeding grep-read. KEY FACTS: enable_*_mode flags + ironman{uuid,death_limit,…} seeded from axr_main.config character_creation; gdiff_N/econ_N families from plugins/difficulty.ltx w/ per-key diff_game/diff_eco overrides, push-applied (console cmds + engine setters + toggle_feature calls into F103/F102 features), tamper info portions; game_setup = dynamic world-item conservation loop (purge → eco-scaled spawn from dynamic_item_spawn.ltx recorded spots → spawn-on-take replacement, props deferred to pickup via alife_process_item) + 3 once-flag map fixups + cordon army removal; agony/timer = on_before_save_input vetoes (health tiers/bleed/rad/surge/combat; save-cooldown TimeEvent); ironman = on-death iterates ALL $game_saves$ .scoc sidecars, marshal-rewrites death_count by uuid, ui_load_dialog.delete_save_game at limit, Second Wind grants via life_cycle hours minus sleep_hrs; azazel = before-death veto + victim pick (warfare: 5 option-gated candidate classes w/ distance windows; else random sim_default_) + stalker_azazel corpse mimic + identity/loadout copy + task cancel + teleport/ChangeLevel + surge stop. VERIFIED BUG: toggle_feature_arty_degradation is ONE shared toggle, on_game_load double-call → bkpk value wins (arty=true+bkpk=false kills arty degradation; callback's internal per-factor gates verified). QUIRKS: 5× dead `local m` dist_mult stores, squad_mates never populated (task-cancel guard dead), bleeding=1 heals (ChangeBleeding=Incarnation, C++ read). No gamedata override of any file |
| flows/game-services.md | verified | NO AGENT — game_statistics + game_achievements + game_fast_travel + game_backpack_travel + game_autosave_new + game_autosave + game_registrator READ-IN-FULL + plugins/achievements.ltx + src/xrGame/ui/UIRankingWnd.cpp/UIAchievements.cpp (functor-driver spine). KEY FACTS: game_statistics = rank/reputation authority — increment_statistic/rank/reputation clamp (rank 0–56000, rept ±4000) + band-crossing news tips from creatures/game_relations.ltx rating/reputation lists vs actor_miscellaneous snapshot; npc_statistics save=true tallies persisted into NPC m_data via alife_storage_manager; actor_on_item_take artefact tracking w/ _G.ARTY_FROM_CONT container-exploit guard + FollowByPath + F90 on_artefact_take; MARSHAL save saves unlock-bools+counts NOT reward values (so config edits take effect). game_achievements = 24 functor/req/reward triplets; **pull-based: invoked ONLY by PDA Ranking tab — CUIRankingWnd reads [achievements] section→builds CUIAchievements per entry→SetFunctor(functor=)→Update calls f() each refresh, truthy+not-shown→unlock+reward fires** (C++ verified, no Lua poll/timer); rewards DISTRIBUTED via has_achievement() at ~15 consumer sites (bind_anomaly_zone/dialogs/axr_companions/gameplay_disguise/item_*/level_environment/xr_box/ui_mutant_loot/treasure_manager/xr_conditions), few act directly (tourist→3 stashes, wishful/absolver/collaborator→unlock factions via axr_main.config, iron_curtain→40000 RU). game_autosave_new = 60s TimeEvent→save tempsave on other/autosave_timer_new. fast/backpack travel = PDA map-spot menu (F39) to whitelisted smarts (faction_quick_travel.ltx, tristate off/visit-only/all) / deployed backpacks, shared guard battery (combat/weight/damage/storm) + time-forward + set_actor_position|ChangeLevel. VERIFIED DEAD: game_autosave entirely commented out ("now unused"); game_registrator vestigial MP clsid mapper (only "single" reachable); **bookworm_food commented out of [achievements] list+ignore=true → widget never built → functor never polled → unreachable achievement**. QUIRKS: articles/killed_stalkers counters unused for rewards, increment_reputation skips during bar_arena_fight, DEV_DEBUG sets permanent debug_mode_flag_on (disqualifies invictus); requirement-comment drift (murky_spirit/wishful_thinking/invictus comments name wrong questlines vs the info portions checked). No gamedata override of any file |
| modlayer/gameplay/gamemodes-services.md | map | gamemodes & services family map; structure verified via F106 (6 scripts READ-IN-FULL + flag-seeding chain) + F107 (7 game_* scripts READ-IN-FULL + C++ functor driver); mechanics delegated to flows/game-modes.md (F106 ✅) + flows/game-services.md (F107 ✅) + flows/warfare-metagame.md (F105) + flows/actor-survival-status.md (F103 toggles) + flows/save-load.md — domain complete |
| modlayer/gameplay/axr-addon.md | map | axr_* modded-exe addon layer map; DOMAIN COMPLETE (all axr_* files listed + clustered: boot orchestrator axr_main [F65] / behaviour schemes [F113 ✅] / managers task+companion+trade [F114 ✅]); mechanics delegated to flows/axr-behaviour-addons.md (F113 ✅) + flows/axr-managers.md (F114 ✅) + flows/gameplay-boot.md (F65 axr_main) + flows/script-npc-actions.md (F37) + flows/scheme-engine-core.md (F66) |
| modlayer/gameplay/utils-db.md | map | utility & registry foundation map; scaffolded at F115 (bottom-of-stack: utils_* helper libs [F115 ✅] / db global registry [F69 ✅] / script bootstrap _g+class_registrator [_g=F69, class_registrator ⬜]); mechanics delegated to flows/utils-libraries.md (F115 ✅) + flows/db-registry.md (F69 ✅ db+_g) + flows/gameplay-boot.md (F65) + flows/lua-callback-lifecycle.md; class_registrator not yet a dedicated flow (small/mechanical — fold into db-registry if needed); no gamedata override of any utils_* (verified) |
| modlayer/gameplay/configs-data.md | map | config/data tier map (Phase G9, FINAL gameplay map); scaffolded at F118 (clusters creatures/actor [F118 ✅] / zones [F119 ⬜] / items+weapons+outfits [F120 ⬜] / per-smart logic [F121 ⬜] / plugins+tweaks+presets+defines [F122 ⬜] / spawns+ALife+graphs [F123 ⬜]); LTX/DLTX parse+inherit([child]:parent)+#include+patch mechanism delegated to modlayer/config-ltx.md + flows/dxml-xml-interception.md; mechanics → flows/creatures-actor-configs.md (F118 ✅); cross-tier ownership rule (game_relations=F40/F108, route don't dup); DLTX = no gamedata full overrides expected |
| flows/axr-behaviour-addons.md | verified | NO AGENT (PARTIAL READ) — axr_beh (794) + axr_beh_patches (126) + axr_keybind (83) READ-IN-FULL; 6 combat/utility schemes (axr_fight_from_cover/stalker_panic/npc_vs_box/npc_vs_heli/radio_in_heli/turn_on_campfire) STRUCTURE-SURVEYED (headers + evaid/actid 188114-188120 + scheme-reg interface setup_generic_scheme/add_to_binder/configure_actions/disable_generic_scheme/npc_add_precondition + evaluator/action class shape; execute bodies NOT line-read). gamedata/scripts/axr_beh_patches.script DIFFED byte-for-byte vs unpacked = IDENTICAL (redundant copy). KEY FACTS: every add-on is a GOAP scheme on the F37 interface EXCEPT axr_keybind. axr_beh (188111, active_section, Alundaio) = companion escort/patrol/cover BRAIN, the AI axr_companions (F114) drives + source of npcx_beh_* info flags (PDA roster F110); init_custom_data condlists keyed on npcx_beh_* (behavior_state beh_wait/move/patrol, target commander/actor, keep_dist near/far, stealth substate hide/sneak/rush); set_desired_target vocab = story-obj/enemy/cover_near_actor/cover_wait/task_target_anomaly/task_target_hostage/waypoint/commander/scan_area/lvid:/id:; handlers beh_wait/beh_cover/beh_move(position_node offset so squad doesn't stack)/beh_path(waypoint ptN w/ look/animpoint/sound/sig/delay + path_end loop/reverse/switch); add_to_binder preconditions alive+no-enemy+no-danger, stock alife/state_mgr yield to beh; **use_camp branch COMMENTED OUT (TODO: Fix use camp)**. axr_beh_patches (GhenTuong, _AXR_BEH_EX_LOADED guard) monkeypatches set_scheme/execute/finalize so [beh@section] declares init/main/finalize=file.func hooks (target=nil+behavior_state=nil disables stock). Combat schemes share validate/lmove/lmove_ns + try_go_cover/strafe/backward skeleton: fight_from_cover(188118 home_point), stalker_panic(188115 can_cause_panic find_valid_target), npc_vs_heli(188117 best_heli, is_under_npc_vs_heli, pairs F95), radio_in_heli(188116 = Lua side of DynNews RadioInHeli F112), npc_vs_box(188120 is_box whitelist), turn_on_campfire(188114 gulag-job light_campfire F72/F73). axr_keybind = bind(dik_or_bind,functor,alias)→on_key_press/release/hold (typ 1/0/2), persists rebindable aliases to axr_main.config [global_keybinds], kCUSTOM1-15 from ui_keybinding.xml. Seams F37/F66 (scheme reg) + F65 (axr_main boot/callbacks) + F114 (companions consume axr_beh) + F95 (heli) + F112 (radio news) + F72/F73 (campfire jobs). axr_main itself = F65 not duplicated |
| flows/axr-managers.md | verified | NO AGENT (PARTIAL READ) — axr_task_manager callbacks + on_game_start + 2 honcho tables + generate_cache/CFG_CACHE + get_first_available_task READ-IN-FULL (1-230,359-468); drx_sl_* generators + generate_available/ongoing/finished_tasks dialog builders STRUCTURE-SURVEYED (function list + caller grep dialogs/xr_conditions/xr_effects/dynamic_news/stalker_generic/tasks_bounty). axr_companions MANAGER LAYER READ-IN-FULL (roster lifecycle 1-456: companion_squads/non_task_companions/assigned_items/_current_states, can_join_actor/add_to_actor_squad/remove*/setup_companion_logic, special-squad add/dismiss/release + fade_in_out.ppe + xr_effects.* story hooks add_task/speaker/task_giver_companion + setup_companion_task escort/hostage spawn, healing_timer/unstuck/on_level_changing; inventory 1336-1605: transfer_item/all + assigned_items ownership + is_overweight 30kg cap + OnItemBefore/AfterMove loot veto, on_key_release kCUSTOM1-5/18 + axr_keybind kCUSTOM3 move_to_point, cw_prepare wheel slices); command-mode setters (set/cycle/get_companion_*_mode 5 axes combat/move/stealth/loot/distance) + UIWheelCompanion/UICompanionList CUIScriptWnd STRUCTURE-SURVEYED (function list). axr_trade_manager read 1-200 + surveyed but OWNED BY F101 (trade-economy.md economy #3) — routed not duplicated. KEY FACTS: 3 high-level axr_* managers, all loaded/ticked via axr_main F65. axr_task_manager = DoctorX-Storyline task GENERATOR extending engine task core F97 (task_manager.get_task_manager/task_info/task_ini) — NOT a class, flat module of globals+callbacks+drx_sl_* dialog functors; bounties_by_id (task_id→target npc_id) + hostages_by_id (npc_id→smart_id) registries (save/load persisted); drx_sl_main_honcho_table (comm→main trader story-id) + drx_sl_honchos_table ({story_id,comm[,comm2]} all task-givers); CFG_CACHE pre-parses every task section's hot fields; get_first_available_task = per-NPC picker (stored-task cache 5400s/3hr + 75% chance gate + section-prefix pool <task_section|npc_section|npc_name|simulation>_task_ + prereq/sim_communities filter + random pick). Dialog surface: generate_available/ongoing/finished_tasks + get_task_job/fetch/complete_descr + npc_give_first_available_ordered_task + trigger_fetch/job_func. axr_companions MANAGER = roster (companion_squads all / non_task_companions voluntary count vs max_actor_squad_size [2, ×2=4 patriarch on_game_load]) + command system writing npcx_beh_* flags (the F113 axr_beh brain reads them) + special/story squads (setup_companion_task spawns escort, registers hostages_by_id, sim_offline_combat ignore) + inventory assignment + command wheel (kCUSTOM18) + healing(+0.1/10s)/unstuck(teleport-in-place/30s). Config ai_tweaks/axr_companions.ltx. Bridges F97/F98 (task core + tasks_bounty/defense), F82 (single-companion scheme model — F114 deepens its agent-traced manager half), F113 (axr_beh consumes npcx_beh_*), F101 (axr_trade_manager), F87/F105 (bounty gossip in dynamic news). VERIFIED QUIRKS: bounty no-credit branch changed stage 3(fail)→1(complete) = bounties never fail; is_overweight 30kg cap HARDCODED; unstuck = acknowledged teleport hack; backward-save cruft (companion_N migration, companion_squads saved false rebuilt live) "remove me later" still present; se_load_var(item_id,nil,"strapped_item") guard DEAD (F102 nil-global family); epic_hack misnamed = heli-danger helper. No gamedata override of any of the 3 scripts (verified find) |
| flows/utils-libraries.md | verified | NO AGENT (CATALOG READ) — full function-name inventory pulled for all 6 utils_* (~500 funcs); load-bearing regions READ-IN-FULL: utils_obj cover/movement (validate/lmove/find_*_cover/try_go_cover/strafe/backward, 470-660 — the db.used_level_vertex_ids reservation), utils_data cfg_file class (1060-1200 read/write INI, SaveExt preserves layout+comments) + parse_* family + findfunction, utils_item get_item_cost/get_item_trade_status (960-1050, fires on_get_item_cost callback, cost cache). utils_ui/utils_xml/utils_stpk STRUCTURE-SURVEYED (inventory + headers + class skeletons; bodies mechanical per-field, not line-read). KEY FACTS: 6 flat helper libs (no shared class hierarchy), the cross-cutting layer ~everything calls into. utils_obj = objects/squads/navigation + the cover/movement PRIMITIVES combat schemes reuse (F113 axr_beh + stock xr_* combat call straight in). utils_item = item/weapon/outfit/addon + CANONICAL pricing get_item_cost (base×trade-cfg×discount×cond-exponent, on_get_item_cost seam) consumed by trade F101; upgrade tree; amount/stack ops. utils_data = math/vector/CTime-serial/path + the parse_condlist/parse_* CONFIG PRIMITIVES the scheme/condlist engine F66 is built on + cfg_file (Lua read/WRITE INI; engine ini_file is read-only — this is what axr_main.config + plugin LTX persist through F65/F111/F112/F113) + findfunction (string→functor). utils_xml = UI XML load + color + RESOLUTION SCALING (correct_ratio/screen_ratio/adjust_*) + XmlParser DOM. utils_ui = the LUA INVENTORY-CELL ENGINE (UICellItem icon/bar/counter/upgrade/attach + UICellContainer grid drag/scroll/sort + UIInfoItem/UICellProperties/UIHint) = backbone of F104. utils_stpk = server (cse_alife) packet read/WRITE (get_/set_*_data per class + parse_/fill_*_properties_packet mirroring C++ STATE_Read/Write — fragile, must track engine; spawn_smart_cover/spawn_heli; net-packet primitives) used by debug spawners F111 + save editing. CROSS-CUTTING: mostly stateless except utils_item cost cache + cfg_file instances; cfg_file vs ini_file = write vs read; db.used_level_vertex_ids is shared cover-reservation. No gamedata override of any of the 6 (verified find) |
| flows/sound-music.md | verified | NO AGENT — xrs_dyn_music (435) + aaa_sound_object_patch (30) READ-IN-FULL; xr_sound get_sound_object/get_safe_sound_object (534-643) read; sound_theme/sound_manager/sound_ambient/mob_sound/ph_sound/xr_zones_sound STRUCTURE-SURVEYED (class/func inventory); level_weathers + game_levels_music.ltx characterized from headers (weather = sibling subsystem, not deep-read). gamedata diffed: aaa_sound_object_patch = cosmetic only (printf→printf_me comments), options_modded_exes_sound OVERRIDE adds options_modded_exes_sound_delay subpage (NEW gamedata-only). KEY FACTS: NO single sound manager — loose subsystems on engine sound_object. TWO music systems, no shared code: (1) engine ambient/background level music from game_levels_music.ltx (per-level/shared/day playlists, start-hr/stop-hr/vol, snd_volume_music channel); (2) xrs_dyn_music = Lua combat music state machine that DUCKS ambient. xrs_dyn_music: main_loop AddUniqueCall ~1s, gated by engine IsDynamicMusic(); get_theme_state start/idle/finish/nil — actor dead OR silence-zone OR no-stalker-enemy → finish, has IsStalker enemy in xr_combat_ignore.fighting_with_actor_npcs → start (ONLY stalkers, NOT monsters); 4 HARDCODED themes (music\combat\themeN_part_1..3) ×3 parts, random pick + gapless attach_tail chain via TRACK_SWITCH_DELTA; crossfade fade_theme up / fade_ambient (snd_volume_music) down by VOLUME_DELTA=ambient_vol/50, force_fade instant; surge coupling (surge_manager.sound_started/is_killing_all); silence zones = db.storage.silence_zone_table + 3 hardcoded *_no_assault restrictors; stereo_sound wraps sound_object s2d. xr_sound = scheme-facing player (sound logic field): play_sound/looped, get_sound_object(theme,t_type) random-no-repeat/seq/looped selection caching sound_object per path via get_safe_sound_object. sound_theme = config-driven theme registry (npc_sound/actor_sound/object_sound/looped_sound classes + load_sound/init_npc_sound) — the registry F112 spoken-news reads. aaa_sound_object_patch (aaa_ loads FIRST) = monkeypatches _G.sound_object → my_sound_object registering into soundCache + AddUniqueCall GC of finished sounds (lifetime/leak guard). Scheme sounds: sound_manager (sm scheme = CStory campfire storyteller phrases), sound_ambient (level ambient/wind/rain-on-helmet via actor_on_update), ph_sound (snd_source physics-object scheme), xr_zones_sound (zone_sound on_enter), mob_sound (pick_sound_from_set monster audio). level_weathers = Lua dynamic-weather cycle driver (AtmosFear-3 lineage, dynamic_weather_graphs.ltx) feeding engine CEnvironment (renderer-environment.md) — own subsystem, tangential. QUIRKS: combat music ignores monsters; themes hardcoded in script (not config); silence zones partly hardcoded; two cooperating sound_object caches (xr_sound path-cache + aaa lifetime-GC). Seams: engine IsDynamicMusic/snd_volume_music, F112 (sound_theme spoken news), F17 (sound options pages), F66/F37 (sound schemes), monster-schemes (mob_sound), renderer-environment (weather) |
| flows/treasures-secrets.md | verified | NO AGENT — treasure_manager.script READ-IN-FULL (header docstring + init_settings + create_/get_/set_random_stash + try_spawn_treasure + all callbacks + dual persistence + on_game_start; only get_random_stash candidate-search body structure-surveyed); tasks_stash head read (DRX quest-stash effects) + surveyed; treasure_manager.ltx + secrets.ltx config tiers characterized; item_radio stash coupling confirmed by grep. KEY FACTS (Tronex, "based on CoC treasure manager"): PROCEDURAL system spawns loot in ALL INVBOX objects, content decided on NEW GAME + frozen in save, spawned lazily on box open. Item pool from items\settings\treasure_manager.ltx: possible_items → item_list[kind][tier] (cost/kind/tier from system ini, economy-type 'eco' exclusion, multiuse tier-reduced); settings_common/rare = min/max weight budget + kind_tier_effect_* (per-kind chance/weight × per-tier chance_multi/weight_multi); blacklist/rare_stashes_names. set_random_stash = fill random stash_weight budget by shuffle kinds→shuffle tiers→roll chance*chance_multi+inc → pick item, subtract weight*weight_multi; inc ramps +10 anti-stall; rag_and_bone achievement +200 weight. RARE stash = gated toolkit_reward_count>=task_num × start_chance prob → cycles stash_rare.reward_bonus toolkit; uses treasure_searched map spot (vs treasure). try_spawn_treasure (physic_object_on_use_callback) = materialize caches[id] csv into box: weapons get random addon flag (scope/gl/silencer per *_status) + random ammo type + random 0-30% condition via utils_stpk.get/set_weapon_data (F115) + manual alife():register; else alife_create_item. caches[id] STATE MACHINE: false=registered-empty / "csv"=assigned-not-spawned / true=spawned-or-looted; actor_on_item_take_from_box (true→false) removes map spots + stashes_found stat + send_treasure(1) news + item_radio.clear_stash. on_game_load = scan 1..65534 INVBOX, register non-blacklisted caches[id]=false, create random stash in HALF (floor(count/2)). Dual persistence: save_state/load_state MARSHAL (caches table) + save(pk)/load(pk) net-packet fallback (USE_MARSHAL gate). PREDEFINED secrets.ltx (misc/, + per-level secret_zaton/jupiter/pripyat includes) = authored [treasure]=item,count,prob legacy CoP-style, consumed by se_item/task_functor/dialogs — distinct from procedural fill, both coexist. tasks_stash = DRX quest stash (xr_effects.drx_sl_create_quest_stash spawns quest item to random stash + drx_sl_quest_stash_bonus picks reward ammo). item_radio RF_stashes[lvl][id] = radio-frequency stash hint, cleared on empty. QUIRKS: loot new-game-locked (config edits don't re-roll existing boxes); only HALF boxes get stashes; "Save as Rare Stash" map tool only when enable_debug (writes treasure_manager.ltx live via ini_file_ex). Seams: F112 send_treasure, F97/F98 quest stashes, F115 utils_stpk weapon data, item-behaviors-lua/alife-services spawn, pda-map-news map spots. No gamedata override (verified) |
| flows/creatures-actor-configs.md | verified | NO AGENT (CONFIG READ) — monsters.ltx/stalkers.ltx/spawn_sections.ltx master includes read FULL; m_dog.ltx read as representative species (structure + inline immunity/damage/protection/vision sub-section refs at 246/285/334); actor.ltx/immunities.ltx/damages.ltx/game_relations.ltx heads read; monster_base located in configs/defines.ltx. ~30 m_*.ltx + ~27 spawn_sections_*.ltx are leaves on same pattern (not each re-read). KEY FACTS: configs/creatures/ data tier. Include structure: monsters.ltx #includes all m_<species>.ltx + [monsters_common]; stalkers.ltx #includes m_stalker/m_person/immunities/damages/zombied/monolith; spawn_sections.ltx #includes ~27 per-level. INHERITANCE [child]:parent: monster_base (in defines.ltx, NOT creatures/) → [m_<species>_<variant>]:monster_base → per-level spawn_section variant → spawned object. Monster m_*.ltx anatomy: ALife identity (GroupControlSection/$npc/visual/corpse_visual/icon/cform/class AI-id/script_binding=bind_monster.bind), stats (Health/MaxHealthValue/speeds/panic_threshold), ef_creature/weapon/detector_type, loot (Spawn_Inventory_Item_*), bones (bone_torso/head/fire), influences (fire_/psy_/radiation_ max-dist/power/linear/quadratic absorption), physics boxes, INLINE profile refs (damage=m_dog_damage, immunities_sect=dog_immunities, protections_sect, vision_free/danger_section, hit_fraction_monster — monster immunities are PER-SPECIES INLINE not from immunities.ltx). immunities.ltx = named STALKER resistance presets (no/low/medium/high_resistance) w/ burn/strike/shock/wound/radiation/telepatic/chemical_burn/explosion/fire_wound_immunity coeffs (lower=more resistant, multiplies incoming dmg). damages.ltx (~1150) = named bone-damage profiles (body_damage_*: per-bone (armor,multiplier) hit-zone scaling), referenced by damage=<profile>. game_relations.ltx = rank/reputation/rating threshold tables + monster_rating + goodwill + attitude_*_threshold + relations matrix — OWNED BY F40/F108 (relations-info/faction-relations-lua), rank/rep also read by F107 game-services; ROUTED not duplicated. actor.ltx = [actor] player creature def (class=S_ACTOR, bind_stalker.actor_init) + movement coefs (run/sprint/jump/crouch/climb/walk_accel) + max_item_mass + attachable_items + NV sounds + ph boxes + survival params (pairs F103 actor). spawn_sections_<level>.ltx = per-level ready-to-spawn section catalog (thin species overrides) → bridge to F123 spawns. helicopter.ltx → F95. QUIRKS: 2 immunity mechanisms (stalker named profiles vs monster inline); monster_base in defines.ltx a dir up; retune monster globally=edit m_*.ltx, per-level=edit spawn_sections_*; DLTX patches not full overrides (no gamedata copies). No gamedata override of configs/creatures/ (verified) |
| flows/zones-anomaly-configs.md | verified | NO AGENT — zones.ltx + zone_base.ltx + zone_mine_gravitational.ltx + zone_field_radioactive.ltx + zone_field_psychic.ltx + zone_teleport.ltx + zone_monolith.ltx + sr_teleport_sections.ltx READ-IN-FULL; zone_campfire head + class= catalog grepped across all 20 files; system.ltx #include verified. ~14 zone_*.ltx are variants on the same param block (not each re-read); runtime meaning of each param routed to F27 anomaly-zones.md not re-derived. KEY FACTS: configs/zones/ data tier. zones.ltx = pure #include manifest pulled into system.ltx (#include "zones\zones.ltx"). zone_base = inheritance root: shared light defaults + effective_radius + **script_binding = bind_anomaly_field.bind** (THE seam — every anomaly gets the F90 Lua artefact binder); [zone_base_noshadow] shadow variant; all else [..]:zone_base. SECTION→ENGINE CLASS via class = ZS_*/Z_* (clsid → CustomZone subclass, F27): ZS_MINCE/ZS_GALAN (gravi strong/avg), ZS_MBALD (~13 sects: gravi-weak/teleport/minefield/mosquitobald), ZS_RADIO (radioactive+psychic fields + monolith psy emitter), ZS_BFUZZ (burningfuzz/liana/hvatalka/studen grabbers), ZS_TORRD/Z_TORRID (fireball/generator thermal), ZS_NGRAV (nogravity), Z_CFIRE (campfire). Spawn meta: GroupControlSection=spawn_group_zone, $spawn (editor section), $def_sphere/$prefetch, ef_anomaly_type/ef_weapon_type (AI eval-fn types, same ef_*_type family as creatures F118). TIERS = thin inheritance leaves: [zone_field_radioactive]:zone_base full block → [..._weak]:[...] overrides only $spawn+max_start_power (0.015/0.02/0.03); read bottom-up. PARAM GROUPS (gravi mine richest): strength (min/max_start_power, attenuation, period); hit (hit_type strike/radiation/telepatic/burn → SHit type feeding damage chain F30, hit_impulse_scale, effective_radius); blowout FSM (awaking→accamulate→blowout_time + staggered *_time + disable_* cooldowns -1=never); particles (idle/blowout/hit/entrance/tele/tearing/torn); sound (idle/blowout/hit/entrance/body_tearing); light (blowout/idle toggles + color/range/time/height); impulse (throw_in/out/alive + atten + radius_percent); teleport-corpse (tele_height/time_to_tele/tele_pause); wind (blowout_wind_* + power); misc (ignore_nonalive/small/artefacts, shape_transp/edge_color, postprocess=postprocess_* preset F122). TWO FAMILIES: zone_field_* (passive DoT, tiny/0 power, no blowout, radiation/telepatic) vs zone_mine_* (blowout FSM + impulse + corpse fling). Oddballs: zone_monolith (ZS_RADIO telepatic max-power psy emitter, artefact economy OFF, visible_by_detector on), zone_campfire ([campfire_base]:zone_base Z_CFIRE, OWN script_binding = campfire world-object binder F89, distinct from anomaly binder). ARTEFACT SEAM: sections declare engine artefact fields (artefacts/BirthProbability/min_max_artefact_count/spawn_blowout_artefacts) but live artefact economy = Lua anomaly_zone_binder (F90) via zone_base script_binding — config fields ≈ declarative; F90 owns spawning. sr_teleport_sections.ltx (co-filed, NOT a zone) = LEVEL-CHANGER table for sr_teleport active restrictor (F92): one [<lvl>_space_restrictor_to_<dest>] per inter-level transition, fields enable(condlist)/spot(map_spots.xml icon)/text(Yes-No prompt)/hint(st_items_quest.xml)/path(Yes teleport waypoint)/reject_path(No)/autosave; distinct from per-script *_teleport*.ltx + sr_teleport_logic.ltx custom_data (F121). Seams F27 (CustomZone runtime) + F90 (artefact binder, script_binding) + F89 (campfire binder) + F30 (hit_type) + F118 (ef_*_type) + F92 (teleport table) + config-ltx (LTX mechanism) + F122 (postprocess preset). QUIRKS: read tiers bottom-up; config artefact fields ≠ live spawning; sr_teleport_sections is F92 data not an anomaly; DLTX = no gamedata full overrides (none at writing); two script_binding paths (anomaly vs campfire). No gamedata override of configs/zones/ or sr_teleport_sections.ltx (verified) |
| flows/items-weapons-outfits-configs.md | verified | NO AGENT — item_base.ltx + weapons/base.ltx + outfits/base.ltx + outfits/base_includes.ltx + items/items/base.ltx + item_upgrades.ltx READ-IN-FULL; w_ak74.ltx (representative gun, FULL) + o_seva.ltx (representative outfit) + weapon_ammo.ltx head + addon_scopes.ltx head + trade_bandit.ltx head read; shared base parents located in configs/defines.ltx; dir tree + counts listed (weapons 134, outfits 22, items 29, trade 35, settings 21). ~130 w_*/22 o_*/29 items_* are leaves on same per-family pattern (not each re-read); runtime meaning routed to engine flows. KEY FACTS: configs/items/ = biggest config cluster. system.ltx #includes items\item_base.ltx = manifest pulling 4 family roots (weapons action_fuse+base, outfits base+base_includes, items base; MP separate under mp\weapons_mp\). **WILDCARD AUTO-INCLUDE = defining convention**: weapons/base.ltx ends #include "w_*.ltx"+"m_*.ltx", outfits/base_includes.ltx #include "o_*.ltx", items/items/base.ltx #include "items_*.ltx" — drop a new prefixed file, loaded automatically, NO edit to shipped files (explicit NOTICE FOR MODDERS comments; the long commented-out explicit lists = legacy docs). Shared base parents (default_weapon_params/identity_immunities/hud_base/global_item_actions/default_item_params/weapon_probability) in defines.ltx (F122) NOT here. WEAPONS w_* (w_ak74 exemplar): root [wpn_xxx]:identity_immunities,weapon_probability,default_weapon_params,wpn_xxx_sounds + parallel [wpn_xxx_hud]:hud_base + variant subsections by inheritance: cosmetic/stat (_alt/_custom/_modern/_old/_rpk) + attach-state (_1p29/_kobra/_ps01 pre-fitted scope). Clusters: identity(class WP_*/slot/kind/weapon_class/cost/inv_weight/repair_type), ballistics(hit_power/hit_type/fire_distance/bullet_speed/rpm/fire_dispersion_*), mag/ammo(ammo_class list/ammo_mag_size/fire_modes), attach slots(scopes+scope_status, silencer_name+status, grenade_launcher_name+grenade_class+status; status 0=attachable/1=fitted/2=integral), condition/jam(misfire_*/condition_*_shot_dec), recoil(cam_*/zoom_cam_*/PDM_*), AI hit chance(hit_probability_gd_novice/stalker/veteran/master), upgrades(upgrades/installed_upgrades/upgrade_scheme), HUD section(item_visual/hands_position/aim_hud_offset_*/anm_* anim map). Owner F22 + state machine + fire callbacks; Lua overheat/ammo-agg fork F102 item_weapon. OUTFITS o_* (o_seva): outfit_base (:global_item_actions, class EQU_STLK/slot 6/repair_type outfit/helmet+backpack_avaliable + **use1_functor=gameplay_disguise.menu_patch** disguise hook F102/F104); [xxx_outfit]:outfit_base[,outfit_restrictions_N] + inline [sect_xxx_immunities]; clusters visuals/community/kind/cost + PROTECTION block (burn/shock/radiation/chemical_burn/telepatic/strike/explosion/wound/fire_wound_protection) + SEPARATE immunities_sect *_immunity coeffs (two mechanisms) + bones_koeff_protection (→ damages.ltx F118) + hit_fraction_actor + additional_inventory_weight + artefact_count + upgrades. outfit_restrictions_1..5 = helmet/backpack/sprint gating; helmets o_helmet.ltx. Owner F23. **QUIRK: outfit_restrictions_temp dummy must stay LAST (engine fails to read last section before includes)**. ITEMS items_* = consumable/misc categories (food/drink/drugs/medical/cooking/repair/tools/parts/money/quest/letters/trash/monster/explosives/attachments/container_*/patches/devices/artefacts); generic params=default_item_params (defines F122); owners F21/F24(artefacts+devices)/F25(consum+explosives)/F102(use plugins). AMMO weapon_ammo.ltx: [ammo_base]:identity_immunities,default_weapon_params (class AMMO/kind w_ammo/tier/box_size) + per-cartridge k_* ballistic multipliers (k_hit/k_impulse/k_ap/k_dist/k_disp/k_bullet_speed/k_air_resistance/buck_shot/tracer) feeding bullet sim F22 + disassemble_parts. ADDONS weapon_addons/silencers/scopes/sounds + addon_scopes.ltx (scope_detector/scope_nightvision*_2gen w/ pp_eff_name=*.ppe NV refs) + upgrades_properties; owner F22. UPGRADES item_upgrades.ltx master #includes action_fuse+upgrades_properties+weapons/upgrades/upgrade_presets+outfits/upgrades/upgrade_includes; up_gr_* trees + upgrade_scheme_*; owner F26. CROSS-TIER (route not dup): items/trade/trade_*.ltx (35) + presets (trade_presets/trade_discounts) = [trader] buy/sell condlists + supplies_N stock → OWNED BY F101; items/settings/ → treasure_manager(F117)/itms_manager+death_*(F102)/ph_box_*(F89)/dynamic_item_spawn(F106)/new_game+npc_loadouts(F123)/craft+cook+parts+devices+artefacts(subsystems). No gamedata override of configs/items/ (verified, no dir) |
| flows/per-smart-logic-configs.md | verified | NO AGENT — door.ltx + sr_sleep.ltx + sr_teleport_logic.ltx + camp.ltx + escape/esc_smart_terrain_2_12_smart_logic.ltx (full gulag job catalog) + escape/smart/esc_smart_terrain_1_11.ltx (smart respawn def) + the 1 gamedata DLTX delta READ-IN-FULL; dir tree + per-level counts listed (2016 total: jupiter 206, zaton 177, pripyat 130, sarcofag 107, warlab 78, bar 70, …). Per-level files = instances of same patterns, not each re-read; scheme/condlist runtime meaning routed to F66/F68/F67 + scheme-catalog flows. KEY FACTS: configs/scripts/ = the custom_data / logic-LTX tier. **NOT #included — loaded by NAME** as an object's/smart's custom_data pointer (set in spawn/binder/scheme cfg=); no master manifest, the filename IS the reference. Org: top-level reusable templates + one subdir per level. TWO FILE ROLES: (1) Object logic = single [logic] active=<scheme>@<sect> FSM + on_*/on_actor_inside/on_actor_outside condlist transitions (F68 format). Exemplars door.ltx (ph_door@unlocked locked/closed, F93), sr_sleep.ltx (sr_idle tutorial @wait→@run %=run_tutorial%), sr_teleport_logic.ltx (sr_idle@wait→@run %=sr_teleport%, the per-script teleport custom_data for sr_teleport F92, sibling of sr_teleport_sections F119). Attach to ONE object via custom_data; scheme sections = the Phase G2/G4/G5 catalog. (2) Gulag/smart logic = many [logic@<job>] each active=/prior=/optional suitable={condlist} true/monster_job=true = the F73 RANKABLE JOB CATALOG. On esc_smart_terrain_2_12_smart_logic: static walker_1..5(prior 40)/guard_1..4(camper prior 45); surge-reactive surge_work_1..11(suitable {=surge_started}, prior 60, walker@+out_restr hide); camp_work_1..18(suitable {!surge_started}/{=is_night}, prior 41, animpoint@ cover_name/use_camp/turn_on_campfire + day/night/RAIN variants chained via on_info {=is_heavy_rain} + [meet_random] inline F79); mutant-swarm faction_base_defense_enemy1..3(monster_job=true, suitable {=target_squad_name(...)}, prior 200, mob_home@ F94). **IN-FILE section inheritance** [logic@work_1]:logic@surge_def factors shared job fields (local, not #include). One smart-logic file = whole smart population behavior, condlist-gated by surge/time/weather/present-squad. SMART RESPAWN DEF = scripts/<level>/smart/<smart>.ltx ([smart_terrain] squad_id/max_population/respawn_params/respawn_idle + [respawn@..] + [spawn_all_weak] spawn_squads/spawn_num) read by F72 + F86; DISTINCT from *_smart_logic.ltx. TOP-LEVEL TEMPLATES (shared custom_data across levels): door family/ph_campfire/ph_sleepbag/ph_tent/inventory_box+treasure_inventory_box+actor_inventory_box/sr_sleep/sr_teleport_logic/bribes_logic/beh_companion/scenes/dummy. **camp.ltx SPECIAL: [camp] = campfire-storyteller registry (stories/guitar_themes/harmonica_themes) read by sound_manager sm scheme F116, NOT an FSM**. scripts/tasks/ = quest task logic (F97/F98). DLTX deltas apply: 1 gamedata override scripts/labx8/mod_lx8_door_lock_fix.ltx = @[ph_button@unlocked] anim/tooltip delta (surgical, not full copy). Seams F68(format)/F66(walk)/F67(condlist)/F73(job board)/F72+F86(respawn)/scheme-catalog/F116(camp)/F97-F98(tasks)/config-ltx. QUIRKS: reference-by-name (typo=no logic, engine-default idle); smart/<smart>.ltx≠*_smart_logic.ltx; suitable+prior decide assignment not active=; in-file inheritance local; camp.ltx=registry; DLTX deltas exist. 1 gamedata DLTX override (verified) |
| flows/plugins-tweaks-presets-configs.md | verified | NO AGENT — defines.ltx READ-IN-FULL (303 lines) + presets/includes.ltx + presets/gameplay_diff.ltx head + ai_tweaks/xr_combat_ignore.ltx + simulation_objects.ltx heads + plugins/guaranteed_drops.ltx head + axr_options.ltx (confirmed EMPTY 0 lines) + system.ltx include READ; full plugins(26)/ai_tweaks(18)/presets listings enumerated. Per plugin/tweak runtime behavior routed to owning flow not re-derived. KEY FACTS: the tuning-knob & shared-FOUNDATION tier = constants, not instances. **defines.ltx = inheritance ROOT** (#included FIRST in system.ltx, bottom of every [child]:parent chain): hud_base (weapon-HUD offset/strafe/shooting defaults, F120 wpn_*_hud parent) + hud_mid/low_interia + hud_movement_layers; immunity presets sect_identity/helm/zero_immunities; item chain global_item_actions→identity_immunities→booster (consumable base eat_*/boost_*, class II_FOOD, F103/F102)+booster_script/booster_multi+zero_immunities; **weapon parents (F120 routes here) weapon_probability + default_weapon_params:identity_immunities (hit_power/hit_impulse/use1_functor=item_weapon.menu_scope_inv)**; **creature root (F118 routes here) common_ph_friction_params_on_npc_death→monster_base (feel distances/actor_hit_effect/custom_data=default_custom_data.ltx)** — CONFIRMS F118/F120 "lives a dir up in defines.ltx". presets/ = bundles via includes.ltx: video_minimum..extreme (→renderer/F17), economy_diff+gameplay_diff (→F106; namespaced gameplay/gameplay_diff/<key>=value option tables actor_immunities/hit_power/dispersion/weight/thirst/sleep), warfare_default+warfare_yoko (→F105). **axr_options.ltx EMPTY STUB** — real options = F17 options_modded_exes_* + axr_main.config. plugins/ (26) routed: achievements→F107/actor_effects→F103/bounty_squad→F114-F98/difficulty→F106/disguise→F102/dynamic_anomalies→F90/dynamic_faction_relations→F108/encyclopedia(+interactivity)→F100/faction_profile→F108-F112/faction_quick_travel→F107/guaranteed_drops→F102 loot[N-corpses-no-item force drop counter]/inventory_tabs→F104/mlr→MLR/new_game_money+options+setup+start_locations→F106/news_tips_icons→F112/nimble_orders→quest/player_icons→UI/radio_zone_fm→F110/radioactive_water_settings→F103/spawner_blacklist→F117-F123/xr_sound→F116/xrs_debug_tools→F111(+debug/ subdir). ai_tweaks/ (18) routed: xr_combat_ignore→F76[combat_inertion per-faction/diff + forbidden_npcs]/simulation_objects→F85-F86[target_maps LINKED-LEVEL perf table]/axr_companions→F114/axr_smartcover→F33/sr_light→F91/xr_corpse_detection+xr_gather_items→F80/xr_help_wounded+xr_eat_medkit→F78/xr_danger+xr_visual+xr_weapon_jam+xrs_facer+xrs_kill_wounded→F81/rx_ff+rx_gl→combat/ai_tweaks→general. INSIGHT: tweaking numbers w/o new content = edit here, but behavior owned by system flow (router + 2 owned pieces defines+presets). QUIRKS: defines.ltx load-bearing (#included first, breaks all inheritors); axr_options empty; presets=option-key tables not object sections; plugins/ai_tweaks=config-not-code (read w/ owning flow). No gamedata override of plugins/ai_tweaks/presets/defines.ltx (verified) |
| flows/spawns-alife-graphs-configs.md | verified | NO AGENT — alife.ltx + game_graphs.ltx + game_story_ids.ltx + game_spawn_story_ids.ltx + game_maps_single.ltx + patrol_paths.ltx + dynamic_spawn.ltx READ; spawns/all.spawn LOCATED (binary, characterized structurally not byte-read). KEY FACTS: the world-instance + ALife tier (FINAL gameplay flow F123). **spawns/all.spawn = BINARY authored-world file** (unpacked/spawns/, not configs/): every static cse_alife_* object record [smart_terrains/restrictors/anomalies/INVBOX stashes F117/story NPCs/level-changers] + **way_ patrol paths** + level/graph xrefs; SDK/xrAI-built offline; loaded at NEW GAME to seed ALife (F16/F41); save diverges after; NOT text, edit via SDK/ACDC. The INSTANCE layer whose templates are F118–F122 + logic (custom_data) is F121. **alife.ltx [alife] = simulator tuning** (schedule_min/max/process_time/objects_per_update/update_monster_factor offline budget; time_factor 6 game-sec/real-sec; **switch_distance 750m online-switch** + auto_switch_* xr_patch perf system; start_time 09:00/start_date 25.10.2018; autosave_interval; **start_game_callback=_G.start_game_callback F65**) → F16. game_graphs.ltx = graph location/level tables: [location_0]=master ordered ~200 graph-point NAME catalog (smart_terrains/*_sim_N/lairs by id, all levels), [location_1/2]=access categories (limited/denied/not-for-player), [location_3]=per-vertex level membership; names the points sim_board F86/squads F85/transitions reference + draw-color palette (debug). game_story_ids.ltx [story_ids] = numeric story-ID→unique object-name registry (~74 lines, per-level ranges) → alife_object_by_story_id/story_objects, the STABLE script handle for a placed object (runtime ids change). game_spawn_story_ids.ltx [spawn_story_ids] = parallel spawn-time table, EMPTY in base (header only). game_maps_single.ltx = level registry [level_maps_single] (jupiter/k00_marsh/l01_escape…l13_generators) + PDA world-map [def_map]/[global_map] bound_rect/texture (F39) + [___level_music_tracks] per-level music (F116); game_maps_mp=MP variant. **patrol_paths.ltx EFFECTIVELY EMPTY (1 line) — patrol paths live as way_ elements IN all.spawn, NOT here**; walker/camper path_walk/path_look (F74) resolve against the binary spawn. dynamic_spawn.ltx near-empty stub (live dynamic-item-spawn = items/settings/dynamic_item_spawn.ltx F120/F106). UNIFYING: templates(F118-F122) × placements(all.spawn) × logic(F121) × constants(F122) = authored world, ALife(F16, tuned by alife.ltx) runs offline over graph(game_graphs), smart_terrain/sim_board(F72/F86) populate, story ids(game_story_ids) address objects, PDA map(F39) draws via game_maps_single. QUIRKS: all.spawn binary+new-game-only (edited spawn doesn't change saves); patrol_paths/dynamic_spawn stubs; story ids stable handle; DLTX applies to text files only not all.spawn. No gamedata override (verified). **COMPLETES the gameplay corpus M1–M23 + F65–F123 → whole KB done.** |
| flows/ui-news-tips.md | verified | NO AGENT — news_manager (263) + dynamic_news_helper (749) READ-IN-FULL; dynamic_news_manager (2857) SPINE read in full (update_settings/lifecycle/__init/destroy/TickNews/TickQuick/PushToChannel/NewsToggle) + ~90 composer methods structure-surveyed (signatures only). KEY FACTS: TWO tiers, meet only at engine give_game_news (F39). Tier1 news_manager = universal synchronous dispatcher (~50 callers: dialogs/tasks/trade/items/stats/combat) — send_tip (generic, sound_theme spoken-snd extends showtime, faction/character_icon, type 0), send_sound (voice/faction news type 1, characters_voice\ path→key), send_task (new/complete/fail/reversed/updated, pda_task, give_talk_message2 if talking), send_treasure, relocate_item/relocate_money (bottom-left in/out/docs showcase); on_game_start loads plugins\news_tips_icons.ltx→tips_icons. Tier2 Dynamic News (ARS→Alundaio→MLR→Tronex) = self-scheduling generator singleton: get_dynamic_news() lazy; **~27.5s startup delay** (actor_on_first_update 7.5s + actor_on_update 20s) then DRX questline gossip once (drx_sl_start_news guard); per-faction queue[community] + channel_status (monolith/army/greh/isg open only if actor IS that comm); PushToChannel enqueues LIFO msg table (Mg/Se/Dl/Ic/Snd/It/Ty/Id), gated by enable_news/NewsToggle/PDA-charge/counter≤max_cnt(3). SIX CreateTimeEvent ticks on "DynamicNewsManager" group: TickNews (drain, re-arm 5-20s → dynamic_news_helper.send_tip + arm TickQuick reaction), TickQuick (reactions), TickSpecial/TickTask/TickRandom/TickCompanion (config cycle_of_*_news, randomized v+1..v*2+1, generators). 3 callbacks: monster_on_death/npc_on_death/npc_on_get_all_from_corpse → DeathBy*/KillWounded/SOS*/loot gossip (offline=SOSBattleOffline F87, warfare=SOSWarfareCapture F105). **NewsToggle gate (key rule): suppress when paused (anti-crash) / invalid map (helper invalid_maps underground+labs) / surge+psi_storm (ERASES queue to stop post-surge flood) ; emits comm-centre on/off tip per surge edge (pda_welcome/pda_communication_lost) + one-time WelcomeToNetwork on new game (trx_dynamic_news_welcome_to_network); requires item_device.is_pda_charged**. Helper toolkit: level_points landmarks + GetPointDescription (nearest + 8-way compass → "near X to the east"), GetTimeString 6-accuracy-tier phrasing ("at 8 PM"…"soon"), Pick/Get over faction_expansions (PickFaction natural skips game_relations enemies F108), GetCommunity/Monster/WeaponDescription, OWN send_tip (sound-by-type pda_tips/alarm/communication_lost/beep/welcome, DRAINS PDA battery slot 8 via item_device.drain_device_on_event, %c[default] color). faction_expansions = faction/level/mutant profile data tier. Seams F39/F17(~22 dynamic_news toggles+4 cycles+message_duration)/F102+F110(PDA battery)/F87/F105/F94+F34(monster desc)/F107(CompanionAboutActor rank-rep)/F108+F40(PickFaction)/surge_manager+psi_storm_manager. VERIFIED QUIRKS: shw_stash hardcoded false; ResponseOnDeathBySurges [NOT USED]+ReportDeathBySurge commented (live surge chatter = *_Fake fabricated variants); helper PickRandomKey DEAD+BROKEN (pairs(v) undefined global, no callers); GetTimeString tier-5 copy-paste bug (else reassigns hrs_L not hrs_R → drops to tier-4). No gamedata override of any of the 3 scripts |
| flows/ui-debug-tools.md | verified | PARTIAL AGENT — spine READ-IN-FULL (ui_debug_launcher open/registration/inject/editor-launch, xrs_debug_tools main_loop/activate_feature/toggle-keys/on_game_start, debug_cmd_list API, ui_debug_main function surface + commented-out opener, _g DEV_DEBUG, all 3 gamedata files); editor INTERIORS structure-surveyed only (ui_debug_item/weather/lighting/wpn_hud function lists + entry points + shared ui_debug_launcher.xml, NOT line-read). KEY FACTS: **DEV_DEBUG (set in _g.script) gates the whole subsystem** — ui_debug_launcher + xrs_debug_tools on_game_start skip all callback registration unless DEV_DEBUG (so inert in shipping build); also gates _g error reporter + printf. THREE generations: (1) ui_debug_launcher = LIVE launcher (Tronex "Anomalous Debugger" 2019), opened by main_menu_on_keyboard when dik_to_bind==kCUSTOM17 (F7) + level.present + actor alive → start_debug_main; sub-windows UIDebugMain/UIDebug_ItemSpawn/UIDebug_ObjSpawn/UIDebug_Executer/UIDebug_FactionSwitch (CUIScriptWnd via start_ui/Unregister_UI); inject(key,v) extensibility API (verified callers bind_anomaly_zone "Spawn artefacts", txr_routes "Unlock all routes"); hide_ui policy controls unpause; launches editors (ui_debug_weather/lighting/wpn_hud.start(GUI), ui_debug_item weapon/outfit/artefact, ui_workshop). (2) ui_debug_main = LEGACY debug_ui menu, DORMANT (its xrs_debug_tools "S"/keybind_debug_ui opener is commented out). (3) xrs_debug_tools = HUD INSPECTOR (Alundaio 2012), NOT a dialog: main_loop @1s throttle builds 3 hud_tool custom-statics (cs_debug/cop_debug/cop2_debug) + dynamic-dispatch _G[..][slot.."_show"](hud); panels = squad/planner(GOAP action-id→name map)/heli/monster/zone/feel-touch/wounded/surge/actor-slots/stalker-data; toggle keys 1/2/3 cycle slots (persisted axr_main.config xrs_debug_tools/toggle_*_key); activate_feature sets smart_terrain.dbg_hud + _ALIFE_CACHE_RECORD; reads plugins\xrs_debug_tools.ltx; needs other/debug_hud + level.present. debug_cmd_list = named command registry (command_give/get_list/exists), shared bus for launcher cmd= + console + print_debug_hud. GAMEDATA DELTA (runtime-winning): zz_ui_debug_inputs (NEW) monkeypatches launcher UIDebug_* methods for per-field input-history recall from axr_main.config [better_debug_inputs] (zz_ prefix loads after launcher; brittle vs method renames); imgui_lua_debug (NEW) ImGui "Debug→LuaDebug" menu → debugger_attach/stop/reconnect (EmmyLua remote); options_modded_exes_debug (OVERRIDE) debug options page logging+errors subpages (F17). Seams: F65 callbacks (all DEV_DEBUG-gated), F19 CUIScriptWnd/custom-static, F12 GOAP/squad introspection, F17 options (debug_hud/debug_error) |
| modlayer/gameplay/ui-scripts.md | map | ui_* scripts family map; structure verified via F109 (ls of ~38 ui_*.script + cluster classification + two engine seams CUIScriptWnd/custom-static); mechanics delegated to flows/gameplay-ui-screens.md (F109 ✅) + flows/pda-tabs.md (F110 ✅) + flows/ui-debug-tools.md (F111 ✅) + flows/ui-news-tips.md (F112 ✅) + flows/actor-menu-inventory-ui.md (F104 ✅) + flows/hud-ui-load.md (F19) + flows/options-runtime.md (F17) + flows/pda-map-news.md (F39); ui_* flow coverage COMPLETE (leaf dialogs remain per-file leaves; axr/utils = F113–F117) |
| flows/faction-relations-lua.md | verified | NO AGENT — game_relations.script (1044 lines) + creatures/game_relations.ltx (313 lines) READ-IN-FULL; dynamic_faction_relations.ltx controls + relation_registry(_actions).cpp bridge. KEY FACTS: DoctorX Dynamic Faction Relations = Lua layer over engine relation_registry (F40); 5-component goodwill sum (personal+reputation+rank+community+community_to_community) computed by engine, thresholds attitude_*_threshold ±999 (personal) vs FRIENDS/ENEMIES ±1000 (community matrix); API wrappers (is_factions_enemies/friends/neutrals, squad/gulag goodwill getters+setters) consumed by ~35 scripts; seeding via reset_all_relations from [communities_relations], per-pair persistence drx_df_*_relations save_var + new_game_relations alife flag (on_game_load: load saved pairs OR reseed on new game); dynamic relations = online_npc_on_death (npc_on_death_callback) + offline_npc_on_death (sim_offline_combat F87) → calculate_relation_change weighted by rank_table/rep tables + give_range jitter + friend/enemy_count_limit damping (warfare drops limits to -1 unless fog_of_war), change_faction_relations clamps friend_limit 4100/enemy_limit -5000 + war-footing news tips, gated by alife/general/dynamic_relations option. game_relations.ltx SHARED config: rating/reputation bands (game_statistics F107), communities_relations matrix (engine native load), rank/reputation_relations, rank_kill_points + action_points (engine F40), ranks_properties (engine, crash-if-removed), smart_terrain_rank_change (F72), monster_communities/relations (F34/F94). VERIFIED BUGS: factions_table_all (30, actor/bandit/dolg/…) does NOT match shipped [communities_relations] order (32, actor/monster/trader/army_npc/greh_npc/bandit/…) despite ATTENTION comment → reset_all_relations indexes columns positionally → new-game reseed maps mismatched cells (file-compare verified, in-world effect not runtime-observed); reset_goodwill precedence bug (`==f1 or "actor_"..f1` always truthy → always first branch); friend_num dead branch (duplicate elseif → friends counted as natural); set_squad_goodwill_to_npc reads undefined global new_relation (param is new_goodwill) → always neutral; get_squad_relation_to_actor_by_id returns "friends" plural + no nil guard; tutorial comment swaps rank/reputation_relations labels. No gamedata override |
| flows/gameplay-ui-screens.md | verified | NO AGENT — ui_main_menu/ui_dyn_msg_box/ui_enemy_health/ui_dosimeter/ui_extra/ui_freeplay_dialog/ui_registrator READ-IN-FULL + UIDosimeter.cpp + consumer greps. KEY FACTS: two engine seams (CUIScriptWnd dialog [engine pumps Update/OnKeyboard] + HUD custom-static AddCustomStatic/GetCustomStatic/RemoveCustomStatic), XML via CScriptXmlInit:ParseFile (F19). ui_main_menu = SP menu from ui_mm_main.xml → new game (ui_mm_faction_select F106)/options (ui_options F17)/load/save/last-save; CALLBACK HUB (main_menu_on_init/quit/keyboard, on_localization_change, MAIN_MENU_RELOADED); Save+F6 fire on_before_save_input veto (F106), F2 DEV new-game test, StartGame console start server(all/single/alife/new). ui_enemy_health = last-hit-enemy bar (cs_enemy_health, on monster/npc/heli_on_hit, gated show_health/option + who:id==0, encyclopedia-name lookup, 30s auto-remove). ui_dyn_msg_box = reusable toolkit (msg_box_ui/multi_choice/context_menu/context_props) consumed by ui_load_dialog/ui_sr_teleport/xr_effects (NOT ui_inventory — it uses utils_ui). ui_freeplay_dialog = Pripyat pri_a28 leave/stay prompt driven by xr_effects. DEAD/DORMANT: ui_extra fully unreferenced (commented actor_on_update reg + nil-self get_protection); Lua ui_dosimeter has no base-corpus get_UI caller (engine UIDosimeter.cpp renders the device, item_device.script F102 holds the logic); ui_registrator vestigial MP UI clsid (sibling of game_registrator F107). No gamedata override of any F109 file |
| flows/pda-tabs.md | verified | pda.script + pda_actor + pda_flagger READ-IN-FULL + UIPdaWnd/UIRankingWnd bridge grepped; 5 tab modules agent-traced + spines SPOT-VERIFIED. KEY FACTS: engine UIPdaWnd.cpp calls pda.set_active_subdialog(section)→tab module get_ui() returns CUIScriptWnd; section map eptNPC/eptRelations/eptContacts[**warfare tab replaces contacts when _G.WARFARE — verified**]/eptRadio/eptEncyclopedia[F100]; eptTasks/Ranking/Logs engine-native. pda.script also: map-spot context menu property_box_*→map_spot_menu_* bus (F39); get_stat panel (game_statistics F107); Top-100 ranking via calculate_rankings[from info_portions]+engine coc_rankings_* (UIRankingWnd/UIRankingsCoC — same window that polls achievement functors F107); fill_primary_objects/discover_spots anomaly+sleep map spots (non-warfare ScanForSpots); actor_menu_mode records m_data.actor_contacts on talk; quick slots slot_0..3 (F104). pda_actor/pda_flagger = WARFARE-only F39 map-menu plugins (commander influence/control + squad/smart target flagging). TABS (spot-verified): contacts=nearby OnlineStalkers in device_pda radius²+faction color+axr_main.config checkboxes; npc=dead-NPC PDA reader, register_pda from death_manager, faction_expansions msgs+treasure stash+RF_stashes(item_radio)+faction_encyption+**1% kill_the_strelok easter egg**; radio=Zone-FM 2 channels[hardcoded num_of_ch]+sessions+2 playlists from $game_sounds$ file_list_open, radio_zone_fm.ltx hotkeys, is_pda_charged battery + emission/underground interference; relations=relation_registry.community_relation matrix+community_goodwill(AC_ID) row+FRIENDS/ENEMIES color (F108/F40)+axr_companions roster w/ npcx_beh_* tags; warfare=registered_squads+SIMBOARD.smarts_by_names dominance board, **sets warfare_actor_faction_domination at ≥50% [= iron_curtain achievement gate F107]**. QUIRKS: is_widescreen hardcoded true (dead non-WS branch) in contacts+relations; warfare sort no tie-break; ui_pda_encyclopedia_tab is single-line file (wc false-empty, NOT a stub — F100 module). No gamedata override. **UPDATED 2026-08-17:** the section map is no longer closed — gamedata/scripts/pda_dynamic_tabs.script registry (register/unregister/build_extra_tabs, ids ept_dynamic_<n>) wraps pda.set_active_subdialog and is driven by CUIPdaWnd::Init / RebuildExtraTabs; icons are looked up under the caption string-table id. Engine mechanics in features/pda-tab-scroll.md |
| flows/anomaly-field-binders.md | verified | bind_anomaly_zone (anomaly_zone_binder: __init parses anomal_zone cfg→layers[random cur_layer]/artefacts/coeffs/artefact_ways/fields/mines/caps; spawn_artefact_randomly chance=game_difficulties arty_chance×surge/psi×geologist→arty_collection→alife_create_item at artefact_ways patrol point; refresh restock-to-cap; respawn_artefacts_and_replace_anomaly_zone surge layer-swap; disable_anomaly_fields enables only cur layer; on_artefact_take/turn_off/turn_on; net_spawn db.add_anomaly) + bind_anomaly_field (anomaly_field_binder READ-IN-FULL: fields_by_names register, set_enable→enable/disable_anomaly; dyn_anomalies_spawn/suffle/update/refresh = map-wide dynamic-anomaly spawn+relocate from dynamic_anomalies.ltx; anomaly_on_before_activate veto = only bolts+live creatures trip) read. **Resolves F27's "C++ zone doesn't spawn artefacts" — Lua binder does.** pulse-anomaly agent-traced. No gamedata override |
| flows/world-object-binders.md | verified | bind_physic_object (generic_physics_binder READ-IN-FULL: conditional init invbox/logic/box-visual; reinit db.storage; update=xr_logic.initialize_obj stype_item+issue_event+hit/death/use callbacks; net_spawn db.add_obj; net_destroy scheme net_destroy+on_offline_condlist+db.del_obj; death_callback box loot xr_box.spawn_items+boxes_smashed; save/load xr_logic.save_obj/load_obj), bind_camp (camp_binder→sr_camp.CCampManager in camps[id], update ticks), bind_faction (faction_binder→alife_object update), bind_campfire (campfire_binder: get_campfire, campfires_all+db.campfire_table_by_smart_names registry, on/off state restore from alife_storage_manager campfire_states) READ. container/dynamic_light/dynamo_hand/door_labx8/awr structurally confirmed (object_binder subclass + bind/init attach grepped; bodies agent-traced). Two sub-patterns: scheme-running vs manager/registry. No gamedata override |
| flows/binder-architecture.md | verified | bind_stalker.script (actor_binder net_spawn/net_destroy/reinit/update + init/actor_init attach) + class_registrator.script READ-IN-FULL + gamedata/bind_monster.script (generic_object_binder reinit/update = scheme tick via xr_logic.issue_event + squad-follow) + xr_motivator AddToMotivator/motivator_binder. Attach verified: actor_init→bind_object(actor_binder); init→AddToMotivator→bind_object(motivator_binder); bind→bind_object(generic_object_binder). object_binder=C++ CScriptBinder base (F5), every method chains object_binder.X(self). class_registrator register()=cs/c/s_register clsid→client(C++)+server(se_) + _ALIFE_UNREGISTER. **bind_monster=live gamedata override (528 vs 477 lines).** reinit creates db.storage+registers callbacks (runs every load); net_spawn one-time (veto on false) |
| modlayer/gameplay/binders.md | map | bind_* binder family map; structure verified by listing unpacked/scripts/bind_* + F88; core binders (actor/motivator/generic) classified, object-binder catalog (world/anomaly/sr/ph F89–F93) enumerated; class_registrator role; mechanics delegated to flows/binder-architecture.md (F88 ✅) + F66 + F89–F93 (⬜) |
| flows/offline-combat.md | verified | sim_offline_combat.script spine read: on_game_start (CreateTimeEvent ocs→coordinator + load/save_state), coordinator (option full=1/on_smarts_only=2/off; (un)register squad_on_update/smart_terrain_on_update on mode change; build squads_by_level from SIMBOARD.squads common+non-story+non-task, limit_by_level = count > pop×MAX_SQUAD_COUNT(30) → crowded-levels-only; exclude task-giver squads; warfare option swap), squad_on_update/smart_terrain_on_update (scan level/smart→validate_enemy), validate_enemy (≠self, offline, opponents-free, non-task, is_factions_enemies, distance<opt_dist 20²/40²; story rules incl one-story-same-target instakill), simulate_battle (power=Σnpc, damage=power×(0.4+rand×0.2), kill NPC if dmg>rank or whole squad if power≤0, on_death), set_battle_outcome (game_relations.offline_npc_on_death + SOSBattleOffline news + attacker rank/kill gain), calculate_npc_power (monster=m_class[clsid]×27000±factor; stalker=rank(); ×faction_power), init_settings (offline_combat_simulator.ltx). Verified: offline_npc_on_death (game_relations, comment confirms caller), option in ui_options, ltx sections. No gamedata override |
| flows/sim-board.md | verified | sim_board.script READ-IN-FULL: simulation_board singleton (smarts{id→{smrt,squads,population}}/smarts_by_names/squads/tmp_assigned_squad), register_smart/init_smart/unregister_smart (called from smart_terrain F72), assign_squad_to_smart (deferral via tmp_assigned_squad→leave-old recompute population via smart_terrain_squad_count + squad_on_leave_smart→join-new +population unless script-target + squad_on_enter_smart), get_squad_target (iterate object_registry, evaluate_prior+target_precondition filter, top-5 buffer, 50/50 random-or-best Tronex pick, iempty_table reuse), fill_start_position (simulation.ltx/survival, alife_stalker_pop/mutant_pop scale, common flag; called from se_actor F84), create_squad/create_squad_at_named_location, setup_squad_and_group (change_team_squad_group), general_*_precondition (squad=false; base/territory faction+in_time_interval gates; resource=10-6; lair=true), get_sim_board lazy singleton + clear() new-game nil. Helpers grepped: register_smart caller=smart_terrain, smart_terrain_squad_count=smart_terrain, in_time_interval/change_team_squad_group=_g. No gamedata override |
| flows/scripted-squad.md | verified | sim_squad_scripted spine read (update, get_script_target, specific_update/generic_update move→reach→stay FSM, assign_smart/assign_squad_member_to_smart, remove_npc/on_npc_death, check_online_status→forced_online_status→can_switch_*, target interface get_location/am_i_reached/on_reach_target/evaluate_prior) + simulation_objects READ-IN-FULL (register/get_props/evaluate_prior type-gates→props×behaviour×1/dist, level-link checks, on_game_start PH-thread sim_avail). **gamedata/sim_squad_scripted override = 10 lines/2 sites: assign_smart→sim_board:assign_squad_to_smart (verified by diff).** SIMBOARD:get_squad_target/assign_squad_to_smart/setup_squad_and_group/general_squad_precondition confirmed in sim_board (F86). sim_squad_bounty/warfare are separate managers NOT subclasses (verified no class decl). is_squad_monster=_g.script. Engine base cse_alife_online_offline_group=C++ |
| modlayer/gameplay/squads-simulation.md | map | squad-sim family map; structure verified by listing sim_* + F85; pieces (sim_squad_scripted=class, simulation_objects=scorer, sim_board=board F86⬜, sim_offline_combat=F87⬜, bounty/warfare=separate managers) classified; mechanics delegated to flows/scripted-squad.md (F85 ✅) + F86/F87 (⬜) |
| flows/se-others.md | verified | all 9 se_* files READ-IN-FULL: se_monster (near-verbatim se_stalker clone, one class for all monster clsids), se_artefact (can_switch_offline false within 150m of actor), se_item (CATALOG of ~19 classes: outfit/helmet/weapon×5/item/torch/physic/lamp/ammo/grenade/eatable/invbox/explosive/pda/detector/mgun — all minimal passthroughs; se_physic AWR repack.direction; keep_saved_data_anyway on physic/lamp/eatable), se_zones (se_zone_anom/torrid/visual + se_restrictor teleport-route map spots from sr_teleport_sections.ltx), se_actor (simulation_objects.register + SIMBOARD:fill_start_position + simulation-target interface get_location/on_reach_target/get_alife_task/sim_available/evaluate_prior), se_smart_cover (registered_smartcovers* registries + loophole persistence validated vs smart_covers.descriptions), se_level_changer (enabled/hint), se_car (minimal), se_heli (se_helicopters table + target/respawn u16 persistence + smart unregister). class_registrator mappings grepped (all monsters→se_monster; items by clsid; ce_script_zone→se_restrictor; CActor→se_actor; CHelicopter→se_heli). set_save_marker(_g.script)/simulation_objects.register confirmed. No gamedata override. Engine cse_* bases = engine/server-entities.md (C++) |
| flows/se-stalker.md | verified | se_stalker.script READ-IN-FULL: class "se_stalker"(cse_alife_human_stalker) + se_trader(cse_alife_trader); __init seeds db.offline_objects[id]; can_switch_offline (group_id≠65535→true) the F16 gate; STATE_Write/Read (offline level_vertex_id + active_section + placeholder bool; read gated script_version>10); on_register (engine→callback→can_choose_alife_tasks(false)→SIMBOARD:get_smart_by_name→smart:register_npc); on_death→smart_terrain.on_death+squad:on_npc_death. Registration cs_register(AI_STL_S/AI_TRD_S), SIMBOARD=_G simulation_board, db.offline_objects={} all grepped. **No gamedata override (gamedata adds se_projector/se_stmgun only).** Engine cse_* base bodies = engine/server-entities.md (C++, not re-read) |
| modlayer/gameplay/_overview.md | map | gameplay-corpus L1 hub; structure verified by listing unpacked/scripts prefixes + unpacked/configs subdirs; architecture (boot→db→scheme engine→schemes→jobs→binders→systems) backed by F65/F66/F69 verified flows + existing bridges; M2–M23 sub-maps scaffolded just-in-time; rest delegated to ROADMAP-gameplay.md (⬜) |
| modlayer/gameplay/server-entities-lua.md | map | se_* Lua server-entity family map; structure verified by listing unpacked/scripts/se_* (10 files) + gamedata/scripts/se_* (se_projector/se_stmgun, non-overrides); uniform subclass pattern + cs_register binding + per-class engine bases all verified via F83+F84 (class decls read); mechanics delegated to flows/se-stalker.md (F83 ✅) + flows/se-others.md (F84 ✅) |
| modlayer/dxml.md | verified | same code path |
| engine/physics-collision.md | verified | CFORM (xrCDB.h TRI/MODEL/COLLIDER/AABB), ODE (ode_redefine.h), ignore_collision consumers |
| engine/sound.md | verified | doppler confirmed in Sound.h (Cribbledirge) |
| features/spom.md | verified | branch-specificity confirmed via git grep |
| GLOSSARY.md | verified | entries code-traced; scheduler/shared_str/ALife/hot-reload/gamedata entries deepened in 2026-06-05 pass; only luabind upstream "version", R2+ dynamic-lighting passes, GAMMA/EFP (external) remain inferred |
| engine/xrCore.md | verified | DLTX (flow), shared_str intern via str_container::dock (xr_hash=robin_hood::hash, SRW-locked custom bucket table), FS=CLocatorAPI, smart_cast; xr_unordered_*=robin_hood (verified _stl_extensions.h) |
| engine/engine-core.md | verified | device loop + CConsole + IInputReceiver; scheduler internals read (xrSheduler.cpp: UpdateRT/UpdateDeferred→ProcessStep is time-budgeted SERIAL, SRW lock guards queue not execution — "MT threads the batch" README claim corrected) |
| engine/server-entities.md | verified | CScriptEngine:CScriptStorage + export_classes + CSE_*/net; ALife offline brain verified (CALifeMonsterBrain::update: select_task/process_task/movement; on_switch_online/offline); only the radius threshold + counterpart spawn left inferred |
| engine/_overview.md | verified | file counts + CEngineAPI compile-time renderer selection/static-link; xrGame dependency edges verified from xrGame.vcxproj ProjectReferences (xrServerEntities compiles into xrGame); other modules' full edge sets not enumerated |
| engine/xrGame.md | map | verified L1 map: structure/symbols verified (Actor*, Weapon, CScriptGameObject, ai/, ui/) + **callback mechanism corrected** (per-object `GameObject::e*`); every major mechanism AND gameplay subsystem now delegated to a verified flow (F1–F44); **no residual `(INFERRED)` markers remain** — the last (rest of Level.cpp) resolved by F43 `flows/level-runtime.md`. Complete as a map; only OOS multiplayer undocumented by design |
| engine/renderer.md | verified | verified L1 map; CRender/IBlender/getShaderPath()="r3\\" (R3+R4)/shader load; blender Compile/pass pattern (blender_dof example); **every renderer subsystem now delegated to a verified flow (F46–F62)** — R4 deferred-lighting passes resolved by flows/renderer-lighting + renderer-sun-shadows + renderer-post-process. R1 = flows/renderer-r1-path. Only OOS R2/R3 generations (F63) + DX10 3D fluid (F64) undocumented |
| engine/subsystems-misc.md | verified | xrXMLParser→XMLLuaCallback; xrCPU_Pipe=CPU point-light vertex calc (PLC.cpp, corrected from "job pool"); ReShadeCompat=ReShade addon; compressor=mod.db0 packager; only xrParticles sim internals inferred |
| modlayer/_overview.md | verified | layout + `gamedata` FS entry point (LocatorAPI::LoadArchive); override mechanism verified = m_files dedup, last-registration-wins (LocatorAPI::Register); only the loose-vs-packed scan *order* (fs.ltx-dependent) left inferred |
| modlayer/scripting.md | verified | callback bus + callbacks_gameobject bridge + class_registrator; common scripts = script.ltx-driven; kernel.lua (global/dynamic_callbacks/socket/LuaPanda) verified DEBUG-ONLY; hot-reload mechanism verified (process_file reload=true, init _G); only aaa/zz Lua convention + runtime reload trigger + MT-GC inferred |
| modlayer/config-ltx.md | verified | DLTX verified (flow); full `ltx_help_ex.script` read; 10 params across categories spot-traced to engine consumers |
| modlayer/assets.md | verified | on-disk formats (.ps/.vs/.s/.h; .dds; level folders); materials.ltx/material_pairs.ltx fields read (friction/damping/spring/flags; pair sounds/particles/marks); only base level *contents* (not in repo) inferred |
| modlayer/unlocalizer.md | verified | read `CScriptStorage::do_file` (script_storage.cpp); supported forms derived from the two regexes; section match is lowercased |
| modlayer/options-ui.md | verified | options_builder + options_modded_exes + ui_options_modded_exes; get_opt_table-vs-id-walk quirk; imgui_helper.script read (HelpMarker, ImGuiMod, ImGui_Icon table); only the full engine-side ImGui.* bind surface inferred |
| INDEX/CONVENTIONS/METHODOLOGY | meta | about the KB itself; repo facts cited are verified |

### Known unverified claims still in the KB (genuinely external or disproportionate)

After the 2026-06-05 pass, every *tractable* inline `INFERRED` was driven to code. What remains is
either upstream/external, not present in this repo, or would require reading an entire large subsystem
for a one-line characterization — each is honestly scoped inline:

- **luabind "updated" version** (`GLOSSARY`) — upstream fork-history; luabind *is* the bind mechanism
  (verified), but the version claim isn't a code fact.
- **(renderer fully documented)** — every `src/Layers/` subsystem now has a verified flow (F46–F64),
  including the R4 deferred lighting/post passes (`flows/renderer-lighting.md`, `-sun-shadows.md`,
  `-post-process.md`), R1 (`flows/renderer-r1-path.md`), R2/R3 (`flows/renderer-r2-r3-paths.md`,
  comparative), and the dormant DX10 fluid (`flows/renderer-3dfluid.md`). Per-blender `Compile` bodies
  beyond `blender_dof` + the R2/R3/fluid sim-pass internals are read on demand (the *mechanism* +
  *shape* are verified; exhaustive per-pass HLSL is disproportionate).
- **ALife online-radius distance threshold + online-counterpart spawn/destroy** (`server-entities`,
  `GLOSSARY`) — the offline brain update + `on_switch_online/offline` hooks are verified; the
  threshold lives in `alife_simulator` (xrGame) + config, unread.
- **FS loose-vs-packed scan *order*** (`modlayer/_overview`, `GLOSSARY`) — the dedup mechanism
  (last-registration-wins) is verified; which is registered last depends on `fs.ltx` + directory walk.
- **Full engine-side `ImGui.*` binding surface** (`options-ui`) — the Lua helper scripts are read; the
  C++ ImGui bind is large and tangential.
- **xrParticles emitter/simulation internals** (`subsystems-misc`) — async manager verified; sim
  internals unread.
- **`aaa*`/`zz*` Lua load-order convention** (`scripting`) — a base-game (`unpacked/`) Lua convention,
  not engine-enforced.
- **Mid-session hot-reload trigger** (`scripting`) — the reload mechanism + init `_G` caller are
  verified; a runtime/console trigger for arbitrary modules wasn't found (may not exist).
- **Base level folder contents** (`assets`) — not shipped in this repo.
- **Per-module vcxproj edge sets beyond `xrGame`** (`_overview`) — xrGame's edges verified; others not
  enumerated.
- **`engine/xrGame` is a verified `map` (2686 files)** — structure/symbols verified, and every major
  mechanism AND gameplay subsystem is delegated to a verified `flows/` file (F1–F44). It now has **no
  residual `(INFERRED)` markers** (the last, rest of `Level.cpp`, was resolved by F43). The only
  xrGame-area gap is **multiplayer netcode (OOS, F45)** — undocumented by design.
- ≈70 of the ≈80 `ltx_help_ex.script` params not individually traced (10 sampled all resolved).

## Maintenance

- When adding/removing a KB file, update the file map, quick-routing table, and the audit table.
- A file may only be marked **verified** when every claim in it is traced to `file:symbol` read this
  session; **map** for an L1 map whose mechanisms are all delegated to verified flows (breadth items
  inferred); otherwise **partial** (with inline inferred markers) or **unaudited**.
- Keep each entry's one-line description and tags in sync with the file's frontmatter.
