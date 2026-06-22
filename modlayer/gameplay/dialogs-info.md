---
domain: modlayer/gameplay/dialogs-info
tags: [modlayer, gameplay, unpacked, lua, dialog, info-portion, map]
sources: [unpacked/scripts/dialog_manager.script, unpacked/scripts/dialogs.script, unpacked/configs/gameplay/, unpacked/configs/misc/dialog_manager.ltx]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/dialog-system-lua.md, flows/characters-info-encyclopedia.md, flows/dialog-trade.md, flows/relations-info.md, flows/scheme-meet-abuse-bribe.md, flows/task-system-lua.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Dialogs & character info — L1 map (M14)

The conversation tier: how NPC dialogs are attached, authored (XML graphs vs script-built trees),
gated and filled with dynamic chatter, plus the character-descriptor/info-portion data they hang
off. Mechanics live in `flows/dialog-system-lua.md` (F99); the engine phrase machine is
`flows/dialog-trade.md` (F18).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| dynamic chatter + `add_dialog` bridge | `dialog_manager.script` + `configs/misc/dialog_manager.ltx` | hello/job/anomalies/information/tips priority-picked small talk; the script-built-tree helper every `<init_func>` dialog uses | F99 ✅ |
| shared functor library | `dialogs.script` | ~250 global preconditions/actions: plumbing, transfers, predicates, task-offer glue, trade/upgrade entries, broker/surrender | F99 ✅ |
| story packs | `dialogs_<level>.script` (~14: escape/zaton/mlr/agr_u/…) | per-level/story preconditions+actions referenced from that level's XML | F99 ✅ (bodies agent-traced) |
| companion menus | `dialogs_axr_companion.script` | script-built companion command dialogs | F99 ✅ (bodies agent-traced); system → `flows/companion-system.md` (F82) |
| phrase graphs | `configs/gameplay/dialogs*.xml` (26 files) | `<dialog>`/`<phrase>` trees naming Lua functors + info gates | F99 ✅ / engine F18 |
| attachment | `configs/gameplay/character_desc_*.xml` + `character_dialogs(_no_guide).xml` | per-character `<actor_dialog>` lists; the include gives every NPC `hello_dialog` + the universal set | F99 ✅ |
| character descriptors / info portions / encyclopedia | `npc_profile*.xml` + `character_desc_*.xml`, `info_*.xml` + `info_portions.script`, `ui_pda_encyclopedia_tab.script` + `plugins/encyclopedia*.ltx` | NPC identity pick (profile→specific_character), the info-flag registration + actor info-switch Lua, and the PDA article plugin | F100 ✅ → `flows/characters-info-encyclopedia.md` (engine store → F40) |

## How it fits

Talk (engine F18, gated by the meet scheme F79) → the NPC's `<actor_dialog>` list →
per-dialog XML preconditions → phrase tree (static XML, or `<init_func>` building it via
`dialog_manager.add_dialog`) → per-phrase Lua functors in `dialogs.script`/packs (predicates,
transfers, task offers via F97/F114, trade/upgrade jump-outs) + engine info-portion transfer
(F40). Generic small talk is the `dialog_manager` priority engine over `dialog_manager.ltx`,
rebuilt each session (no persistence).

## Subsystem deep-dives (delegates to flows)

- Tiers, chatter priority system, `add_dialog` bridge, library census + verified quirks
  (`precondition_no_more` nil-global bug, inverted+dormant `smart` gate, `once`=pin-at-255) →
  `flows/dialog-system-lua.md` (F99) ✅
- Engine phrase graph / trade pricing → `flows/dialog-trade.md` (F18)
- Characters / info portions / encyclopedia (profile selection, info registration + actor
  info-switches, PDA article plugin) → `flows/characters-info-encyclopedia.md` (F100) ✅ —
  domain complete
- Task-offer dialogs' backend → `flows/task-system-lua.md` (F97) + F114 ⬜

## Maintenance

- Domain complete (F99 + F100). Re-check `gamedata/scripts/{dialog*,info_portions,
  ui_pda_encyclopedia_tab}` for overrides on pickup (none at writing).
