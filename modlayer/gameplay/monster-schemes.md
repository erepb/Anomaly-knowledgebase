---
domain: modlayer/gameplay/monster-schemes
tags: [modlayer, gameplay, unpacked, lua, schemes, ai, map]
sources: [unpacked/scripts/mob_walker.script, unpacked/scripts/mob_home.script, unpacked/scripts/mob_trader.script, unpacked/scripts/modules.script, gamedata/scripts/bind_monster.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, modlayer/gameplay/binders.md, flows/monster-schemes.md, flows/scheme-engine-core.md, flows/monster-ai-control.md, flows/script-npc-actions.md, flows/special-actors.md, GLOSSARY.md]
updated: 2026-06-10
verified: map
---

# Monster schemes (`mob_*`) — L1 map

Logic schemes for **monsters** (`stype_mobile`) and the **scripted trader** (`stype_trader`) — the F66
scheme engine's non-stalker NPC tiers, run by `gamedata/bind_monster.script` (**live override**, F88) and
`bind_trader.script`. Unlike stalker `xr_*` schemes there is no GOAP/state_mgr: schemes either capture the
monster and drive the script-entity action queue (F37) or leash the engine FSM (F34) via `set_home`.
Mechanics live in `flows/monster-schemes.md` (F94) — this map routes the family.

## The family (11 files, all verified in F94; no `gamedata/` override)

| Scheme | File | What |
|--------|------|------|
| `mob_home` | `mob_home.script` | engine `set_home` territory leash — the workhorse (230 base refs) |
| `mob_walker` | `mob_walker.script` | monster patrol on the action queue + waypoint flags (99 refs) |
| `mob_remark` | `mob_remark.script` | scripted anim(+sound) performance (`snd=` path is broken) |
| `mob_jump` | `mob_jump.script` | look→`jump(point, factor)`→signal `jumped` |
| `mob_combat` | `mob_combat.script` | generic `on_combat` listener — **dormant** (no caller fires it) |
| `mob_death` | `mob_death.script` | generic `on_death` — records `db.storage[id].death.killer` |
| `mob_trader` | `mob_trader.script` | `CAI_Trader` performance (engine trader anim/sound callbacks) |
| `mob_trade` | `mob_trade.script` | generic `on_trade` — trade open/close/deal switch triggers |
| *(none)* | `mob_camp.script` | home-camp FSM — **unregistered, dead in base corpus** |
| *(helper)* | `mob_state_mgr.script` | `state=` applier (bloodsucker `invis`/`vis` only; else aborts) |
| *(helper)* | `mob_sound.script` | legacy trader voice bank + `pick_sound_from_set` |

Registration: `modules.script` `LoadScheme` rows (`stype_mobile` × 6, `stype_trader` × 2).

## How it fits

Engine monster (F34) → `bind_monster` (F88, gamedata override) attaches + routes
waypoint/hit/death/sound callbacks → `xr_logic` schemes `stype_mobile` (F66). The trader (F36
`CAI_Trader`) → `bind_trader` → `stype_trader`. Smart-terrain monster jobs (F72/F73) emit
`mob_home`/`mob_walker` sections with gulag-prefixed paths.

## Subsystem deep-dives (delegates to flows)

- All `mob_*` mechanics + dead/dormant paths → `flows/monster-schemes.md` (F94) ✅
- The binder + callback routing → `flows/binder-architecture.md` (F88)
- Engine monster FSM / control elements → `flows/monster-ai-control.md` (F34)
- The action queue the schemes drive → `flows/script-npc-actions.md` (F37)
- The trader engine class → `flows/special-actors.md` (F36)

## Maintenance

- Re-list `unpacked/scripts/mob_*` + grep `gamedata/scripts/mob_*` for overrides on pickup (none at
  writing). Status lives in `ROADMAP-gameplay.md` (Phase G5).
