---
domain: modlayer/gameplay/axr-addon
tags: [modlayer, gameplay, unpacked, lua, axr, scheme, manager, map]
sources: [unpacked/scripts/axr_main.script, unpacked/scripts/axr_beh.script, unpacked/scripts/axr_companions.script, unpacked/scripts/axr_task_manager.script, unpacked/scripts/axr_trade_manager.script, unpacked/scripts/axr_keybind.script]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/axr-behaviour-addons.md, flows/axr-managers.md, flows/companion-system.md, flows/dynamic-task-types.md, flows/gameplay-boot.md, flows/script-npc-actions.md, flows/scheme-engine-core.md, flows/task-system-lua.md, flows/trade-economy.md, flows/relations-info.md, GLOSSARY.md]
updated: 2026-06-21
verified: map
---

# AXR addon layer — L1 map (M21)

The `axr_*` family: the **modded-exe addon layer** that Alundaio's Call-of-Chernobyl lineage bolts onto
the stock A-Life corpus. Three kinds of thing live under the `axr_` prefix — the **boot orchestrator**
(`axr_main`, the F65 module loader + callback bus + config store), a set of **behaviour schemes**
(GOAP reactions on the F37 interface), and the **gameplay managers** (tasks, trade, companions). This
map routes them; the engine scheme machinery is F37/F66 and boot is F65.

## The pieces

| Cluster | File(s) | Role | Flow |
|---------|---------|------|------|
| boot orchestrator | `axr_main` | module loader (`on_game_start` fan-out), the `RegisterScriptCallback` bus, `axr_main.config` ltx store | F65 (`flows/gameplay-boot.md`) |
| companion brain | `axr_beh`, `axr_beh_patches` | the `beh` escort/patrol/cover scheme + its customization hook | F113 ✅ |
| combat/utility schemes | `axr_fight_from_cover`, `axr_stalker_panic`, `axr_npc_vs_heli`, `axr_radio_in_heli`, `axr_npc_vs_box`, `axr_turn_on_campfire` | auto-active GOAP reactions (cover-fighting, panic, hide-from-heli, radio, attack-boxes, light-campfires) | F113 ✅ |
| keybinds | `axr_keybind` | scriptable key→functor dispatcher (`kCUSTOM*` / DIK), rebindable via `axr_main.config` | F113 ✅ |
| task manager | `axr_task_manager` | DRX-storyline task generator + bounty/hostage registries | F114 ✅ (`flows/axr-managers.md`) |
| trade manager | `axr_trade_manager` | autonomous NPC↔NPC gulag-job economy | F101 (`flows/trade-economy.md`); routed by F114 |
| companions | `axr_companions` | squadmate roster + command layer (writes `npcx_beh_*`, drives `axr_beh`) | F82 (scheme) + F114 (manager, `flows/axr-managers.md`) |

## How it fits

Everything is loaded and ticked through **`axr_main`** (F65): its `on_game_start` calls each module's
`on_game_start`, and the `RegisterScriptCallback` bus is how every addon hooks frame/key/death events.
The **behaviour schemes** register through the standard scheme interface (`flows/script-npc-actions.md`,
F37) and are installed into each NPC's `motivation_action_manager` — they are normal GOAP schemes that
happen to ship in the addon layer. The **managers** are higher-level systems that consume the lower
tiers: `axr_companions` assigns the `axr_beh` scheme to squadmates and reads/writes the `npcx_beh_*`
info flags; `axr_task_manager`/`axr_trade_manager` build on the Lua task (F97/F98) and trade (F101)
flows. Persistent addon state lives in `axr_main.config` (an ltx store) and per-object `db.storage`.

## Subsystem deep-dives (delegates to flows)

- Boot orchestrator (`axr_main` loader + callback bus + config) → `flows/gameplay-boot.md` (F65)
- Behaviour schemes (`axr_beh` companion brain + `axr_beh_patches` hook + the six combat/utility
  schemes + `axr_keybind`) → `flows/axr-behaviour-addons.md` (F113) ✅
- AXR managers (task / companion / trade) → `flows/axr-managers.md` (F114) ✅ — `axr_task_manager` is
  the DRX-storyline task generator (bounty/hostage registries) on the F97 core; the `axr_companions`
  **manager layer** (roster, command system writing `npcx_beh_*`, special squads, inventory, command
  wheel) deepens F82's scheme-side coverage; `axr_trade_manager` is routed to F101
- Scheme engine the behaviours plug into → `flows/script-npc-actions.md` (F37),
  `flows/scheme-engine-core.md` (F66)

## Maintenance

- F113 ✅ (behaviour schemes + keybind) and F114 ✅ (managers, `flows/axr-managers.md`). The axr-addon
  layer is now fully mapped: boot (F65) + behaviours (F113) + managers (F114). `axr_companions` (F114
  manager) is the consumer that drives `axr_beh` (F113) via the `npcx_beh_*` flags — they cross-link.
- `axr_trade_manager` is documented under F101 (`flows/trade-economy.md`, economy #3), not duplicated;
  F114 routes to it. `axr_companions`' single-companion scheme model is F82 (`flows/companion-system.md`);
  F114 owns the manager/command/roster half.
- `axr_main` itself is documented under F65 (boot), not duplicated here — this map points at it.
- `gamedata/scripts/` ships overrides/additions in this layer (`axr_main`, `axr_beh_patches` — the
  latter byte-identical). Re-check `gamedata/scripts/axr_*` on pickup; the addon layer is where
  modded-exe overrides accumulate.
