---
domain: modlayer/gameplay/gamemodes-services
tags: [modlayer, gameplay, unpacked, lua, gamemodes, services, map]
sources: [unpacked/scripts/game_difficulties.script, unpacked/scripts/game_setup.script, unpacked/scripts/gamemode_azazel.script, unpacked/scripts/ui_mm_faction_select.script, unpacked/scripts/game_statistics.script, unpacked/scripts/game_achievements.script, unpacked/scripts/game_fast_travel.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/game-modes.md, flows/game-services.md, flows/warfare-metagame.md, flows/actor-survival-status.md, flows/save-load.md, GLOSSARY.md]
updated: 2026-06-12
verified: map
---

# Game modes & services — L1 map (M19)

The opt-in rule modifiers (`gamemode_*`), the difficulty-factor store (`game_difficulties`), the
dynamic-world-item setup (`game_setup`), and the `game_*` service modules
(statistics/achievements/autosave/travel). Mode mechanics live in `flows/game-modes.md` (F106),
services in `flows/game-services.md` (F107); warfare is its own domain
(`modlayer/gameplay/warfare.md`, M18).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| flag seeding | `ui_mm_faction_select.script` | new-game screen reads `axr_main.config` character_creation → writes `enable_*_mode` flags + `ironman` table into alife storage | F106 ✅ |
| difficulty store | `game_difficulties.script` | gameplay (`gdiff_N`) + economy (`econ_N`) factor families from `plugins/difficulty.ltx`, per-key overrides, push-applied via console cmds/engine setters/feature toggles | F106 ✅ |
| world setup | `game_setup.script` | dynamic world-item conservation loop (spawn-on-take from recorded spots, deferred uses/condition), new-game purges, one-time map fixups, cordon newbie protection | F106 ✅ |
| save policies | `gamemode_agony.script`, `gamemode_timer.script` (+ `bind_campfire` campfire mode) | `on_before_save_input` vetoes: health/bleed/rad/surge/combat; post-save cooldown; campfire-only | F106 ✅ |
| death rules | `gamemode_ironman.script`, `gamemode_azazel.script` | cross-save-file death accounting + save deletion at limit + Second Wind; death-as-body-swap into a living stalker | F106 ✅ |
| stats engine | `game_statistics.script` | rank/reputation authority: per-event counters→rank/rept, band-crossing news tips, NPC-side `m_data` tallies, visited-place tracking, achievement recorder + MARSHAL persistence | F107 ✅ |
| achievements | `game_achievements.script` (+ `plugins/achievements.ltx`) | 24 functor/requirement/reward triplets; **pull-based, polled only by the PDA Ranking tab** (`CUIRankingWnd`/`CUIAchievements` C++); rewards distributed via `has_achievement()` consumer sites | F107 ✅ |
| autosave | `game_autosave_new.script` (`game_autosave` dead) | 60s TimeEvent → `save … - tempsave` on the `other/autosave_timer_new` interval | F107 ✅ |
| map travel | `game_fast_travel.script`, `game_backpack_travel.script` | PDA-map context-menu (F39) travel to whitelisted smarts / deployed backpacks; guard battery + time-forward | F107 ✅ |
| MP clsid | `game_registrator.script` | vestigial: empty `register` + `get_game_clsid` mode→`SV_*`/`CL_*` (only "single" reachable) | F107 ✅ |

## How it fits

All modes are flag-keyed callback plugins on the F65 bus: save policies hook
`on_before_save_input` (flags veto), death rules hook `actor_on_before_death`,
`game_difficulties` re-applies on `on_game_load` and on each option change. The difficulty
factors are the read surface for trade pricing (F101), survival toggles (F103), artefact spawn
chance (F90), world-item density (game_setup itself), and weapon degradation (engine console).

## Subsystem deep-dives (delegates to flows)

- Flag registry, factor families, world-item loop, the four modes, verified quirks (arty/bkpk
  shared-toggle bug, dead azazel distance multipliers, ironman save-sidecar mutation) →
  `flows/game-modes.md` (F106) ✅
- Stats/rank/reputation engine, achievement detector (pull-based UI poll, dead `bookworm_food`),
  autosave, fast/backpack travel, vestigial `game_registrator`, distributed achievement rewards →
  `flows/game-services.md` (F107) ✅
- Warfare mode → `flows/warfare-metagame.md` (F105)
- Sleep/thirst features the gameplay factors toggle → `flows/actor-survival-status.md` (F103)
- Save pipeline → `flows/save-load.md`

## Maintenance

- F106 ✅ + F107 ✅ — **domain complete**. No `gamedata/` override of any F106/F107 file at
  writing; re-check on pickup. Rank/reputation band tables live in the relations data tier
  (F108/F118), not here.
