---
domain: flows/companion-system
tags: [modlayer, gameplay, unpacked, lua, companion, axr, squad, flow]
sources: [unpacked/scripts/axr_companions.script, unpacked/scripts/xr_companion.script, unpacked/scripts/dialogs_axr_companion.script, unpacked/scripts/ui_companion_inv.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-engine-core.md, flows/dialog-trade.md, flows/relations-info.md, flows/ai-squad-logic.md, flows/inventory.md, flows/axr-managers.md, flows/axr-behaviour-addons.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/axr-addon.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: companion system (F82)

Recruited NPCs that join the actor's squad, follow, take commands, and share inventory. A companion is
a normal stalker switched to the `xr_companion` scheme on the `beh_companion.ltx` logic, with all
**commands stored as `npcx_beh_*` info portions** the scheme reads each frame. Built on the scheme
template F74 (`flows/scheme-walker-patrol.md`) + F66 (`flows/scheme-engine-core.md`).

**Provenance:** **verified** in `axr_companions.script` (`setup_companion_logic` gives `npcx_is_companion`
+ `xr_logic.configure_schemes`; `remove_from_actor_squad` disables the info) and `xr_companion.script`
(`set_scheme` sets `st.behavior = beh_walk_simple`; `add_to_binder`; `action_companion_activity:execute`
branches on `st.behavior`). The command setters, keybinds, teleport/unstuck, and the squad/inventory
plumbing are *(agent-traced here — but **personally read and documented in F114**,
`flows/axr-managers.md`)*; the **info-portion command model** is verified by
the give/disable_info_portion calls. **No `gamedata/` override** of these files (verified by listing) —
the companion system ships in `unpacked/` here.

## Concepts (verified where noted)

- **companion** — a stalker with the `npcx_is_companion` info, in the actor's squad, running the
  `xr_companion` scheme. *(verified: `setup_companion_logic` gives the info + configures schemes.)*
- **commands = info portions** — follow/wait, combat stance, distance, stealth, loot, patrol are each a
  `npcx_beh_*` info toggled on the NPC; the scheme + `beh_companion.ltx` condlists read them per frame.
  *(verified by the give/disable_info_portion pattern; the specific mapping is agent-traced.)*

## Sequence (spine)

### Recruit — `axr_companions.script:setup_companion_logic(npc, st, loaded, cannot_dismiss)` (verified)
Verified: `npc:give_info_portion("npcx_is_companion")` (and optionally a cannot-dismiss info), load
`scripts\beh_companion.ltx`, then `xr_logic.configure_schemes(npc, ltx, ltx_name, modules.stype_stalker,
"logic", "")` (+ `activate_by_section`), and unregister the NPC from its smart_terrain. So recruiting =
**flip an info + reconfigure the NPC onto the companion logic** (F66). `add_to_actor_squad` records it in
`companion_squads`/`non_task_companions`; `become_actor_companion` is the dialog entry. Squad callbacks
(`squad_on_first_update`/`_update`) set `scripted_target = "actor"` and relay flags.

### The xr_companion scheme — `xr_companion.script` (verified structure)
Standard scheme template (F74): `set_scheme` sets `st.behavior = beh_walk_simple` (verified);
`add_to_binder` registers `evaluator_need_companion` (→ `xr_logic.is_active`) + `action_companion_activity`;
`action_companion_activity:execute` branches on `st.behavior` → `beh_walk_simple` (follow the actor:
pick a flanking waypoint near the actor, pathfind, `state_mgr.set_state` raid/rush/assault by distance,
threat at the actor when close) or `beh_wait_simple` (hold, look at actor). The `beh_companion.ltx`
condlists select behavior from the `npcx_beh_*` infos (e.g. `{+npcx_beh_wait} beh_wait, beh_move`).

### Commands — info-portion toggles *(model verified; setters agent-traced)*
Issued from dialogs (`dialogs_axr_companion`) or keybinds (`axr_companions.on_key_release`, DIK_1–5 + a
Companions Wheel). Each setter gives/disables an info: follow/wait → `npcx_beh_wait`; combat →
`npcx_beh_ignore_combat`(+`_ignore_actor_enemies`); distance → `npcx_beh_distance_far`; stealth →
`npcx_beh_substate_stealth`; loot → `npcx_beh_gather_items`/`_loot_corpses`; patrol → `npcx_beh_patrol_mode`.
A move-to-point command raycasts a target vertex and saves a rally point. The scheme reads these next
frame — no separate command FSM. *(the exact setter list + keybinds are agent-traced.)*

### Dismiss — `axr_companions.script:remove_from_actor_squad(npc)` (verified head)
Verified: disables `npcx_is_companion`; restores the NPC's original scheme/logic
(`xr_logic.restore_scheme_and_logic`), clears companion vars, re-registers to its smart if applicable.

### Positioning seam *(agent-traced)*
On level change, companion squads teleport to the actor (`on_level_changing` → `TeleportSquad`, gated by
not-waiting/not-patrol); a 30 s `unstuck` timer in-place-teleports a companion that is stuck far behind.

## Function digests (spine)

- **`axr_companions.script:setup_companion_logic`** — *Does:* give `npcx_is_companion`, load
  `beh_companion.ltx`, `xr_logic.configure_schemes`/`activate_by_section`, unregister smart. *(verified.)*
- **`axr_companions.script:add_to_actor_squad` / `become_actor_companion`** — *Does:* register in
  `companion_squads`, recruit entry. *(agent-traced.)*
- **`axr_companions.script:remove_from_actor_squad`** — *Does:* dismiss → restore original logic.
  *(verified head.)*
- **`xr_companion.script:set_scheme` / `add_to_binder` / `action_companion_activity:execute`** — *Does:*
  the companion scheme (behavior branch: follow vs wait). *(verified structure; beh bodies agent-traced.)*
- **command setters (`set_companion_to_*`) + keybinds** — *Does:* toggle `npcx_beh_*` infos. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `companion_squads`/`non_task_companions` registries; per-NPC `npcx_*` info portions
  (the command state); `assigned_items` (shared-inventory bookkeeping).
- **Scheme seam:** runs as the `xr_companion` scheme on `beh_companion.ltx` (F74/F66); behavior chosen by
  condlists over the `npcx_beh_*` infos (F67/F68).
- **Squad/ALife seam:** the actor's squad (`get_object_squad`, `scripted_target="actor"`); teleport via
  `sim:teleport_object`/`TeleportSquad` (`flows/alife-services.md`, `flows/ai-squad-logic.md`).
- **UI/dialog seams:** `dialogs_axr_companion` (recruit/command dialogs, `flows/dialog-trade.md`),
  `ui_companion_inv` (shared inventory, `flows/inventory.md`), the Companions Wheel + keybinds.
- **Info-portion seam:** all commands + the companion marker are info portions (`flows/relations-info.md`).

## Threading / failure / edge cases

- **Commands are state, not events:** toggling an info changes behavior on the *next* scheme tick — there
  is no immediate callback. Debug commands by checking the `npcx_beh_*` infos on the NPC.
- **Teleport gates:** waiting or patrol-mode companions are excluded from level-change teleport; a
  companion left behind is usually in one of those modes.
- **Dismiss restores prior logic:** a recruited story NPC returns to its original scheme on dismiss — and
  `npcx_beh_cannot_dismiss` blocks dismissal of story-locked companions.
- **Squad lifetime:** the companion squad is deleted when its last member dies/unregisters (agent-traced)
  — a stale `companion_squads` entry indicates a missed cleanup.

## Deeper

- Manager: `unpacked/scripts/axr_companions.script` (`setup_companion_logic`, `add_to_actor_squad`,
  `remove_from_actor_squad`, command setters, keybinds, teleport/unstuck)
- Scheme: `unpacked/scripts/xr_companion.script` (`set_scheme`, `add_to_binder`,
  `action_companion_activity:execute`/`beh_walk_simple`/`beh_wait_simple`)
- Dialogs/UI: `unpacked/scripts/dialogs_axr_companion.script`, `unpacked/scripts/ui_companion_inv.script`
- Logic config: `gamedata/configs/scripts/beh_companion.ltx` (the companion behavior LTX)
- Template + framework: `flows/scheme-walker-patrol.md`, `flows/scheme-engine-core.md`

## Maintenance

- The verified core (recruit = info + configure_schemes; commands = `npcx_beh_*` infos; scheme behavior
  branch) is stable; re-read if the recruit/command model changes.
- Promote the command setters / keybinds / teleport to verified when a task needs exact behavior.
- Re-check `gamedata/scripts/` for overrides of `axr_companions`/`xr_companion` before trusting (none at
  writing — but this is a commonly-modded system, so verify per session).
