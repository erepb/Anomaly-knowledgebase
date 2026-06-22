---
domain: meta
tags: [meta, roadmap, flows, coverage, methodology]
sources: [claudedocs/kb/, src/xrGame/, src/xrEngine/, src/Layers/, gamedata/]
related: [INDEX.md, METHODOLOGY.md, CONVENTIONS.md, ROADMAP-gameplay.md]
updated: 2026-06-10
---

# KB Coverage Roadmap

The authoritative plan for building the KB to full coverage. **Read this (with `INDEX.md`'s audit
table) to know what is done and what to do next** — it is written down precisely so it survives
context compaction. After a compaction: read `INDEX.md` audit table (per-file *status*) + this file
(the *next pending flow* + how to execute it), then pick the highest-priority ⬜ item.

## Guiding insight (why this is bounded)

xrGame is 2686 files / ~875 classes, but file count = **breadth, not distinct mechanisms**. Deep
inheritance concentrates each mechanism in a spine (~6–12 functions); the hundreds of leaf classes are
overrides/config on a documented spine. So total coverage = a **bounded set of ~20 distinct flows**
(not thousands of files). L0 (glossary) and L1 (domain maps) are already complete; the remaining work
is **L2 flows + L3 digests** for the behavioral spines.

**Pointers in the "entry" column below are starting hints, NOT verified claims** — verify each against
code when you pick the flow up (per `CONVENTIONS.md` no-line-numbers + the trust rule).

## State snapshot (2026-06-05)

- **L0 Glossary, L1 domain maps (engine + modlayer), meta files** — ✅ complete & verified (see
  `INDEX.md` audit table). `engine/xrGame.md` is now a verified `map` (structure verified; every major
  mechanism delegated to a verified flow; residual breadth items marked inline *(INFERRED)*).
- **Flows done (20): all F1–F20. Phases 1–4 complete.** F1 lua-callback, F2 dltx-config, F3 dxml,
  F4 weapon-fire, F5 object-lifecycle, F6 actor, F7 weapon-state-machine, F8 inventory, F9 damage-hit,
  F10 ai-planner, F11 ai-perception, F12 ai-squad-logic, F13 engine-boot-frame, F14 render-frame,
  F15 save-load, F16 alife-online-offline, F17 options-runtime, F18 dialog-trade, F19 hud-ui-load,
  F20 animation-callbacks.
- **Core-spine coverage (F1–F20): ✅ complete.** The ~20 cross-cutting behavioral spines are done,
  xrGame.md is a verified `map`, audit table has no stale `partial`, integrity green.
- **Full xrGame subsystem coverage (F21–F45): 🔄 P1 + P2 + P3 COMPLETE (2026-06-06).** All P1 (F22, F23, F24, F26, F27, F30, F38, F40), all P2 (F21, F25, F28, F29, F31, F32, F33, F34, F35, F37, F39, F41, F42), and all P3 (F36, F43, F44) documented & wired. Remaining: **OOS F45 (multiplayer) only**, on demand. F43 resolved `engine/xrGame.md`'s last `(INFERRED)`. F1–F20 documented the
  *mechanisms* (how an object/weapon/NPC/UI/save works in general); F21–F45 document the concrete
  *gameplay subsystems built on those mechanisms* (specific item/weapon/outfit/zone/vehicle/AI/task
  families). Enumerated from a full `src/xrGame/` listing on 2026-06-05 — see **Phase 5+** below. This
  is the answer to "document all of xrGame," scoped so a cold start can pick the next item.
- **Renderer (`src/Layers/`) subsystem coverage (F46–F64): ✅ FULLY COMPLETE (2026-06-07).** Mechanism
  tier (`engine/renderer.md` map + F14 master frame) + **ALL 19 subsystem flows done: F46–F64** —
  resource/shader, blender compile, R_Backend/RCache, scene-graph, visibility, visual types, skeleton,
  detail/grass, dynamic lights, sun cascades, post/SSFX, wallmark engine, particles, environment, HW
  device, render factory & bridges, the R1 reference path, the R2/R3 comparative generations, and the
  (dormant) DX10 3D fluid. **`src/Layers/` is fully documented — nothing left.** See **Phase 11+** below.

## Flow inventory

Legend: ✅ done · ⬜ pending · attaches = which spine flow it builds on.

| ID | Flow | Area | Status | Attaches | Entry hints (verify on pickup) |
|----|------|------|--------|----------|-------------------------------|
| F1 | Lua boot + callback lifecycle | modlayer | ✅ | — | `flows/lua-callback-lifecycle.md` |
| F2 | DLTX config load | engine/core | ✅ | — | `flows/dltx-config-load.md` |
| F3 | DXML XML interception | modlayer | ✅ | — | `flows/dxml-xml-interception.md` |
| F4 | Weapon fire → bullet → hit | xrGame | ✅ | F5 | `flows/weapon-fire-callbacks.md` |
| F5 | Object lifecycle (spawn/update/destroy) | xrGame | ✅ | — | `flows/object-lifecycle.md` (**backbone**) |
| F6 | Actor (condition, camera, input, death) | xrGame | ✅ | F5 | `flows/actor.md` |
| F7 | Weapon state machine (equip/idle/fire/reload/switch + HUD) | xrGame | ✅ | F5,F4 | `flows/weapon-state-machine.md` |
| F8 | Inventory & items (add/move/equip; slots/belt/ruck) | xrGame | ✅ | F5 | `flows/inventory.md` |
| F9 | Damage / hit resolution (hit → condition → death) | xrGame | ✅ | F4,F5 | `flows/damage-hit.md` |
| F10 | AI think loop / GOAP planner | xrGame/ai | ✅ | F5 | `flows/ai-planner.md` |
| F11 | AI perception (memory / vision / feel) | xrGame/ai | ✅ | F10 | `flows/ai-perception.md` |
| F12 | AI squad / logic seam (smart_terrain ↔ xr_logic Lua) | xrGame+mod | ✅ | F10,F1 | `flows/ai-squad-logic.md` |
| F13 | Engine boot → device init → frame loop | xrEngine | ✅ | — | `flows/engine-boot-frame.md` |
| F14 | Render frame (R4/DX11): cull→g-buffer→light→post→present | Layers | ✅ | F13 | `flows/render-frame.md` |
| F15 | Save / load lifecycle | xrGame+server | ✅ | F5 | `flows/save-load.md` |
| F16 | ALife online↔offline transition (**crash-prone**) | server-entities | ✅ | F5 | `flows/alife-online-offline.md` |
| F17 | Options framework runtime | modlayer | ✅ | F1,F3 | `flows/options-runtime.md` |
| F18 | Dialog / trade | xrGame | ✅ | F5 | `flows/dialog-trade.md` |
| F19 | In-game HUD / UI load | xrGame/ui | ✅ | F3 | `flows/hud-ui-load.md` |
| F20 | Animation / motion callbacks | xrGame | ✅ | F5 | `flows/animation-callbacks.md` |

## Phasing (priority + what can parallelize)

- **Phase 1 — player-facing core** (highest value; all attach to the done F5 backbone; F6/F7/F8 are
  independent → fan out in parallel; F9 after F4+F7): **F6, F7, F8, F9.**
- **Phase 2 — AI** (the one area that splits; do F10 first, then F11/F12 build on it): **F10 → F11, F12.**
- **Phase 3 — engine pipelines** (independent of xrGame; F14 attaches to F13): **F13, F14.**
- **Phase 4 — systemic / persistence** (mostly independent; parallelizable): **F15, F16, F17, F18, F19, F20.**

`engine/xrGame.md` has been promoted from `partial` to a verified `map` now that every core-spine
section is backed by a verified flow. Phase 5+ (below) deepens it from a map toward full subsystem
coverage.

## Phase 5+ — full xrGame subsystem coverage (F21–F45)

**Why this exists:** F1–F20 cover the *mechanisms*. xrGame also contains ~25 concrete *gameplay
subsystems* built on those mechanisms, each a real class family verified to exist by listing
`src/xrGame/` on 2026-06-05. This is the plan to document them. **Entry hints are existence-verified
file pointers, NOT verified behavioral claims** — on pickup, read the spine and verify symbols per the
protocol + trust rule. Each flow follows the same recipe (Research → spot-verify → write
`flows/<name>.md` `verified: code` → wire into INDEX/this file/L1 → integrity check).

Priority: **P1** = high modding value (what Anomaly mods touch most) · **P2** = medium · **P3** =
low / niche · **OOS** = out of scope for Anomaly SP (document only if a task needs it).

### Phase 5 — items & equipment (attach F7/F8/F9; F21 first, then parallel)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F21 | Item base hierarchy & attach/detach | ✅ P2 | **DONE** → `flows/item-base.md` |
| F22 | Weapon class hierarchy & addons | ✅ P1 | **DONE** → `flows/weapon-hierarchy-addons.md` |
| F23 | Outfits & helmets | ✅ P1 | **DONE** → `flows/outfit-helmet.md` |
| F24 | Detectors & artefacts | ✅ P1 | **DONE** → `flows/detector-artefact.md` |
| F25 | Consumables & explosives | ✅ P2 | **DONE** → `flows/consumables-explosives.md` |
| F26 | Upgrades system | ✅ P1 | **DONE** → `flows/upgrade-system.md` |

### Phase 6 — world objects (attach F5/F9)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F27 | Anomaly zones | ✅ P1 | **DONE** → `flows/anomaly-zones.md` (note: zones do NOT spawn artefacts in this path) |
| F28 | Physics objects, holders & death ragdoll | ✅ P2 | **DONE** → `flows/physics-objects-ragdoll.md` |
| F29 | Vehicles & holder seam | ✅ P2 | **DONE** → `flows/vehicles-holder.md` (helicopter is NOT a holder) |

### Phase 7 — actor depth (attach F6)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F30 | Actor condition & survival | ✅ P1 | **DONE** → `flows/actor-condition.md` |
| F31 | Camera & post-process effectors | ✅ P2 | **DONE** → `flows/camera-pp-effectors.md` |

### Phase 8 — AI depth (attach F10/F11/F12)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F32 | Stalker execution layer | ✅ P2 | **DONE** → `flows/stalker-execution.md` |
| F33 | Smart-cover system | ✅ P2 | **DONE** → `flows/smart-cover.md` |
| F34 | Monster AI (control architecture) | ✅ P2 | **DONE** → `flows/monster-ai-control.md` (FSM + controls, not GOAP) |
| F35 | Agent/squad combat coordination | ✅ P2 | **DONE** → `flows/agent-squad-combat.md` (danger is per-NPC; macro squad = Lua sim_squad) |
| F36 | Trader / crow / phantom AI | ✅ P3 | **DONE** → `flows/special-actors.md` |
| F37 | Script-driven NPC actions (C++ side of xr_logic) | ✅ P2 | **DONE** → `flows/script-npc-actions.md` |

### Phase 9 — gameplay systems

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F38 | Tasks | ✅ P1 | **DONE** → `flows/tasks.md` (monolithic, no objectives; completion polls info-portions) |
| F39 | PDA, map & news | ✅ P2 | **DONE** → `flows/pda-map-news.md` |
| F40 | Relations, reputation & info-portions | ✅ P1 | **DONE** → `flows/relations-info.md` |
| F41 | ALife simulation services | ✅ P2 | **DONE** → `flows/alife-services.md` (surge = C++ stub, Lua does emission) |

### Phase 10 — binding & runtime

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F42 | CScriptGameObject Lua surface | ✅ P2 | **DONE** → `flows/cscript-game-object.md` |
| F43 | Level runtime (the rest of Level.cpp) | ✅ P3 | **DONE** → `flows/level-runtime.md` (resolves xrGame.md residual INFERRED) |
| F44 | Wallmarks, particles & decals (game side) | ✅ P3 | **DONE** → `flows/wallmarks-particles.md` |

### Out of scope (document only on demand)

| ID | Flow | Pri | Entry hints |
|----|------|-----|-------------|
| F45 | Multiplayer modes & netcode | OOS | `game_cl_*`/`game_sv_*` (deathmatch/CTA/TDM/artefacthunt), `gamespy/`, `Level_network_*.cpp`, `account_manager`/`login_manager`/`reward_manager` — largely irrelevant to Anomaly SP |

**Suggested order for a cold start:** ✅ **all P1, P2, AND P3 done (2026-06-06).** Only **OOS** remains:
- **F45** (OOS) — multiplayer modes & netcode (`game_cl_*`/`game_sv_*`, `gamespy/`, `Level_network_*`,
  `account_manager`/`login_manager`/`reward_manager`); largely irrelevant to Anomaly SP — document only
  if a netcode/MP task explicitly needs it. On pickup, verify the `game_cl_*`/`game_sv_*` class set and
  the `gamespy/` directory still exist.
The single-player KB is complete; F45 is the only remaining (out-of-scope) item.

## Phase 11+ — renderer (Layers/) subsystem coverage (F46–F64)

**Why this exists:** the renderer mirrors xrGame's two-tier story. The *mechanism* tier is already
done — `engine/renderer.md` (verified L1 map: structure, `CRender`/`IBlender`, `getShaderPath()`,
blender `Compile`/pass pattern) plus `flows/render-frame.md` (**F14**, the master R4 deferred frame:
cull→G-buffer→shadows→light→combine/post→forward→present, spine-verified). What remains is the
*subsystem* tier: the per-stage deep-dives that F14's steps and `renderer.md`'s `(INFERRED)` markers
delegate to. F46–F64 are that buildout.

**Why bounded:** `src/Layers/` is 613 files / 8 projects, but the active build is **one shared layer
(`xrRender`, 252 files) + one backend (`xrRenderDX10` → DX10/DX11) + the R4 path (104 files)**;
R1/R2/R3 are alternate paths. Each subsystem concentrates in a spine (a manager class + its per-frame
entry + a handful of phase functions), so coverage is a bounded ~16 flows, not 613 files. Same recipe
as F21–F44 (Research → spot-verify spine → write `flows/<name>.md` `verified: code` → wire into
INDEX/this file/`engine/renderer.md` → integrity check).

**Entry hints are existence-verified file/class pointers** (files listed + central classes grepped
2026-06-06: `CResourceManager`, `CBackend`, `CBlender_Compile`, `CDSGraphManager`, `CHOM`, `CSector`,
`CPortal`, `CModelPool`, `dxRender_Visual`, `CDetailManager`, `CWallmarksEngine`, `CHW`,
`dxRenderFactory`) — **NOT verified behavioral claims.** Verify the spine + symbols on pickup.

Priority: **P1** = highest leverage (what shader/visual mods + SSFX touch, and the central data path)
· **P2** = medium · **P3** = niche · **OOS** = alternate/rare paths, on demand.

### Phase 11 — resource & GPU-submission pipeline (how shaders/geometry reach the GPU)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F46 | Resource manager: shader/texture/constant load & dedup | ✅ P1 | **DONE** → `flows/renderer-resources-shaders.md` |
| F47 | Blender → pass compilation system | ✅ P1 | **DONE** → `flows/renderer-blender-compile.md` (CLSID factory `blender_create`; recorder `r_Pass`/`r_End`; many legacy CLSIDs return 0 on R4) |
| F48 | R_Backend: draw submission, state & constant caches | ✅ P2 | **DONE** → `flows/renderer-backend.md` (global RCache state-dedup; deferred constants/RT flushed in Render; R_DStreams discard/no-overwrite) |

### Phase 12 — scene assembly & culling (what gets drawn)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F49 | Scene-graph (dsgraph) build & phased render | ✅ P1 | **DONE** → `flows/renderer-scene-graph.md` (per-view instances GMBase/GMLight/GMCascade — same code builds main + shadow queues) |
| F50 | Visibility & occlusion | ✅ P2 | **DONE** → `flows/renderer-visibility.md` (three-tier funnel: HOM software depth + sector/portal traverse + GPU occlusion queries; disabled/timeout ⇒ visible) |
| F51 | Visual types & model pool | ✅ P2 | **DONE** → `flows/renderer-visual-types.md` (dxRender_Visual tree + CModelPool two-tier instancing: shared geometry, reference/instance/pool) |
| F52 | Render-side skeleton animation / kinematics | ✅ P2 | **DONE** → `flows/renderer-skeleton.md` (CKinematics IS-A FHierrarhyVisual; Bone_Calculate recursion; bone-callback = physics/ragdoll override; pairs F20) |
| F53 | Detail / grass manager | ✅ P3 | **DONE** → `flows/renderer-detail-grass.md` (camera-centered slot cache + terrain-trace decompress + VS-instanced wind-swayed render; grass shadows via hw_Render(light*)) |

### Phase 13 — lighting & shadows (deferred)

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F54 | Dynamic light accumulation & shadow maps | ✅ P2 | **DONE** → `flows/renderer-lighting.md` (per-light GMLight dsgraph for casters; SMAP atlas; accum_point/spot; LP_normal/LP_pending occlusion split; sun = F55) |
| F55 | Sun & cascaded shadow maps | ✅ P2 | **DONE** → `flows/renderer-sun-shadows.md` (3 cascades, per-cascade GMCascade dsgraph; render_sun_cascade→phase_smap_direct→accum_direct_cascade; pairs F54) |

### Phase 14 — post-processing, effects & environment

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F56 | Post-processing chain (SSFX) | ✅ P1 | **DONE** → `flows/renderer-post-process.md` (verified ordered chain in `phase_combine`; `// [SSFX]` markers distinguish SSFX additions) |
| F57 | Wallmark / decal engine (engine side of F44) | ✅ P2 | **DONE** → `flows/renderer-wallmark-engine.md` (static decals clipped to CDB tris + aged in CWallmarksEngine; skeleton decals skinned via CKinematics::RenderWallmark / sorted Wmark path) |
| F58 | Particle rendering | ✅ P3 | **DONE** → `flows/renderer-particles.md` (particle system IS-A dxRender_Visual; PAPI sim + engine anim/collision; billboards; B2F sort bypassed) |
| F59 | Environment / sky / rain / weather | ✅ P3 | **DONE** → `flows/renderer-environment.md` (game owns CEnvironment state, Layers dx*Render draws sky/clouds/rain/thunder/flares; gamma=output ramp not phase_lut) |

### Phase 15 — device & engine-integration seams

| ID | Flow | Pri | Entry hints (verify on pickup) |
|----|------|-----|-------------------------------|
| F60 | HW device, swapchain & caps | ✅ P2 | **DONE** → `flows/renderer-hw-device.md` (CHW owns D3D11 device/context/swapchain/base RTV-DSV; CreateDevice/Reset/UpdateViews; present in dxRenderDeviceRender::End; CHWCaps gates features) |
| F61 | Render factory & dx*Render bridges | ✅ P2 | **DONE** → `flows/renderer-factory-bridges.md` (global Render/RenderFactory/UIRender wiring; dxRenderDeviceRender device glue + present in End; dxUIRender/font/console/debug → RCache) |

### Out of scope / niche (document only on demand)

| ID | Flow | Pri | Entry hints |
|----|------|-----|-------------|
| F62 | R1 static / light-map path | ✅ P3 | **DONE** → `flows/renderer-r1-path.md` (reference/contrast: forward + baked light-maps + projector/blob shadows; getShaderPath r1\ vs R4 r3\; build is R4) |
| F63 | R2/R3 render generations (comparative) | ✅ OOS | **DONE** → `flows/renderer-r2-r3-paths.md` (R2=DX9/`r2\`, R3=DX10/`r3\`; same deferred shape as R4; all report GENERATION_R2; deltas-vs-R4) |
| F64 | DX10 3D fluid (volumetric) | ✅ OOS | **DONE** → `flows/renderer-3dfluid.md` (GPU NS grid sim; manager initialized in r4.cpp but per-volume RenderFluid commented out → dormant) |

**Renderer status:** ✅ **ALL 19 renderer flows done (F46–F64)** — nothing left. `src/Layers/` is fully
documented: the active R4 pipeline end-to-end, the engine↔Layers seam, plus the R1 / R2-R3 / DX10-fluid
alternate-or-dormant paths. A cold start needs no further renderer work; use the flows + `engine/renderer.md`.

Each renderer flow follows the same per-flow recipe + DoD below. On landing, also promote the
matching `(INFERRED)` in `engine/renderer.md` (R2+ deferred-lighting passes → F54/F55; post passes →
F56) and add the flow to its `related`.

## Per-flow execution protocol (the validated recipe)

For each ⬜ flow (this is exactly how F5 was done):

1. **Research** — spawn ONE read-only **Explore** agent per domain with the tight contract: read the
   spine files, report each function as `` `file:Symbol` `` + *Does/Reads/Writes/Calls/Fires* +
   provenance tag (`READ-IN-FULL`/`SKIMMED`/`INFERRED`), plus the isolation-ledger seams. **No line
   numbers, no pasted code.** (Small/known flows: do inline, skip the agent.)
2. **Spot-verify** — read the actual bodies of the spine + load-bearing claims yourself. Agent output
   is a *draft*; only what you personally confirm becomes `verified`.
3. **Write** `flows/<name>.md` — frontmatter (`verified: code`), Trigger → Sequence (L2) → Function
   digests (L3, spine only per the spine-function rule) → **Isolation ledger** → Threading → Failure
   cases → Deeper → Maintenance. Mark unread sub-steps *(agent-traced)*; mark un-read-by-anyone
   *(INFERRED)*.
4. **Wire in** — add to `INDEX.md` (flows list + audit-status row + quick-routing if apt); flip the
   status here (⬜→✅) and in `METHODOLOGY.md` backlog; update the relevant L1 domain file (`related` +
   a pointer), and promote any inline *(INFERRED)* the flow resolves.
5. **Integrity check** — sources exist, no code fences, **no `:\d` line refs** (except CONVENTIONS
   examples), `related` links resolve. Bump `updated` on every changed file.

## Definition of done (per flow)

L2 sequence + L3 digests for its spine functions + an isolation ledger, **every spine claim
spot-verified against code this session**, wired into `INDEX.md`, integrity green. Anything not
personally verified is explicitly marked `(agent-traced)` or `(INFERRED)` — never passed off as
`verified`.

## Definition of done

Two tiers — be precise about which is claimed:

**Tier 1 — core-spine coverage (F1–F20): MET (2026-06-05).** All F1–F20 ✅; `engine/xrGame.md`
resolved as a verified `map`; `INDEX.md` audit table all `verified`/`map`/`meta` with no stale
`partial`; integrity green (42 md, 20 flows, 0 code fences, line-refs only in CONVENTIONS, all sources
+ cross-links resolve). The KB answers "how does X *mechanism* work" for every cross-cutting spine
without re-reading the tree.

**Tier 2 — full xrGame subsystem coverage (F21–F45): P1 + P2 + P3 MET (2026-06-06).** Every **P1**
(F22–F24, F26, F27, F30, F38, F40), **P2** (F21, F25, F28, F29, F31, F32, F33, F34, F35, F37, F39, F41,
F42), and **P3** (F36, F43, F44) row has a `verified: code` flow wired into `INDEX.md` + this file +
`engine/xrGame.md` `related`, each with its spine spot-verified against code and agent-traced parts
marked. Integrity green throughout (0 code fences, line-refs only in CONVENTIONS, all sources/cross-links
resolve). F43 resolved `engine/xrGame.md`'s **last `(INFERRED)`** marker (rest of `Level.cpp`), so the
map file now has none. **Remaining: OOS F45 (multiplayer) only** — out of scope for Anomaly SP. The KB
now covers the mechanism level (F1–F20) **and** every single-player gameplay subsystem (P1+P2+P3); only
multiplayer netcode is undocumented, by design.

**Tier 3 — full renderer (`src/Layers/`) coverage (F46–F64): FULLY MET (2026-06-07).** All **19**
subsystem flows **F46–F64** have a `verified: code` flow wired into `INDEX.md` + this file +
`engine/renderer.md` `related`, each with its spine spot-verified against code and agent-traced parts
marked. Integrity green (0 code fences, line-refs only in CONVENTIONS, all sources/cross-links resolve).
Together with the mechanism tier (`engine/renderer.md` map + F14 `flows/render-frame.md`) `src/Layers/` is
documented end-to-end for the active R4 build: device → resources/shaders → blender → cull → scene-graph
→ visuals/skeleton/detail/particles → backend submit → dynamic+sun lighting → wallmarks → post/SSFX →
environment → the engine↔Layers factory seam — **plus** the alternate/dormant paths (R1 forward, R2/R3
generations, DX10 3D fluid). **Nothing left in `src/Layers/`.** `engine/renderer.md` has no residual
`(INFERRED)` content markers.

## Part 3 — Gameplay corpus (`unpacked/`, F65–F123) → `ROADMAP-gameplay.md`

This file (F1–F64) covers the **engine** (`src/xrGame`, `src/xrEngine`, …) and the **renderer**
(`src/Layers/`). The game's other half — the **gameplay layer** in `unpacked/` (~456 base-game Lua
scripts + the `configs/` data tier: schemes, smart_terrain/gulag jobs, squads, tasks, dialogs, trade,
survival, warfare, UI) — has its own dependency-ordered plan in **`ROADMAP-gameplay.md`**:
**23 L1 maps (M1–M23) + 59 flows (F65–F123)**, F-numbered to continue from F64 without collision. Same
KB, same recipe/DoD; new maps live under `modlayer/gameplay/`. Engine↔gameplay are normal `related:`
links. **Status: plan created 2026-06-10, buildout not started.** Run cold sessions against that file.

## Maintenance

- This file + `INDEX.md`'s audit table are the two sources of truth for status. Keep them in sync: when
  a flow lands, flip it here AND add its INDEX row.
- If a pending flow turns out to split (like AI did) or merge, edit the inventory — keep the count
  honest, don't let it drift into fiction.
- New mechanisms discovered mid-work (e.g. a Vulkan render path, a networking rework) get a new F-row
  here before being documented.
