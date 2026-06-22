---
domain: flows/scheme-gather-corpse
tags: [modlayer, gameplay, unpacked, lua, schemes, loot, corpse, flow]
sources: [unpacked/scripts/xr_gather_items.script, unpacked/scripts/xr_corpse_detection.script, unpacked/scripts/axr_npc_vs_box.script, unpacked/scripts/xr_box.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-engine-core.md, flows/inventory.md, flows/relations-info.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: gather-items / corpse-detection / box schemes (F80)

How NPCs **pick up dropped items**, **loot corpses**, and **break/search boxes** — generic schemes on
the F66/F74 framework that share one shape: *detect a target in visible memory → claim it so peers don't
race → walk to it → play a search/pickup state → transfer items → un-claim*.

**Provenance:** the **corpse loot-marking is verified** in `xr_corpse_detection.script` (`HasValuableLoot`
table: marked on death, read by the evaluator). The gather `Items[]` claim, the per-corpse
`corpse_already_selected` lock, the memory-scan throttle (`cstackprevent`), and the `axr_npc_vs_box`
shoot-the-box behavior are *(agent-traced — not personally re-read)*. **`xr_box.script` is NOT an NPC
scheme** — it's the `ph_item_box` stash manager; the NPC box-search scheme is `axr_npc_vs_box`. No
`gamedata/` overrides of these files (verified by listing).

## Corpse detection — verified marker + claim

- **`xr_corpse_detection.script:HasValuableLoot`** (verified) — a module table marking which corpses are
  worth looting: set true on a stalker's death (and conditionally for mutants), cleared when looted. The
  evaluator only targets corpses with `HasValuableLoot[id] == true`.
- **Claim (agent-traced):** the evaluator picks the nearest valid corpse from `npc:memory_visible_objects`
  (within `AlwaysDetectDistance`, accessible, not already claimed) and locks it via
  `db.storage[corpse_id].corpse_already_selected = npc:id()` so two NPCs don't fight over one corpse;
  the action walks there, plays `search_corpse`/`field_dress`, then `get_all_from_corpse` transfers the
  lootable items (filtered by a loot table; weapons/armor/quest items excluded), plays a good/bad loot
  sound, increments stats, and un-claims on finalize.
- Loot table built from `system.ltx` (`can_trade`, cost>0, not quest, not the armor/helmet blacklist);
  mutants gated by community + an optional analysis/field-dress mode.

## Gather items — claim + pickup (agent-traced)

- `xr_gather_items` detects gatherable items in visible memory (within `MaxDetectDistance`, gated by the
  `gather_items_enabled` condlist + overweight/danger/surge checks), claims via a **global**
  `Items[item_id] = npc_id`, walks over, plays `pickup_crouch` (artifacts get an extra `search`/`probe`
  state + detector requirement), then `gather_item` transfers (clone weapon / create item / release
  artefact from its anomaly), and un-claims on finalize.
- A `cstackprevent` flag throttles the full memory rescan to ~250 ms to avoid stack blowups on NPCs with
  large visible-object lists. *(agent-traced.)*

## Box — shoot the box (agent-traced)

`axr_npc_vs_box` (the NPC scheme) uses **touch sensing** (`iterate_feel_touch`) rather than visible
memory: when an armed NPC is within ~2 m of a whitelisted box visual, it aims and fires for ~8 s, then
`alife_release_id` destroys the box (spilling its contents). No claim (one box at a time). `xr_box.script`
is the separate `ph_item_box` stash object manager (not a scheme).

## Function digests (spine)

- **`xr_corpse_detection.script:HasValuableLoot` (table) + getter/setter** — *Does:* mark corpses worth
  looting (set on death). *(verified.)*
- **`xr_corpse_detection.script:evaluator_corpse` / `action_search_corpse` / `get_all_from_corpse`** —
  *Does:* select+claim corpse, walk, loot, un-claim. *(agent-traced.)*
- **`xr_gather_items.script:eva_gather_itm` / `act_gather_itm` / `gather_item`** — *Does:* detect+claim
  item (`Items[]`), pick up, transfer. *(agent-traced.)*
- **`axr_npc_vs_box.script:evaluator_npc_vs_box` / `action_npc_vs_box`** — *Does:* touch-detect + shoot
  box. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `xr_corpse_detection.HasValuableLoot` (module) + per-corpse `corpse_already_selected`
  (db.storage); `xr_gather_items.Items` (module). Same scheme framework (F66/F74).
- **Claim seams:** the two claim tables are the contention guards — they live outside the action manager
  so peers coordinate at the Lua level before GOAP arbitration.
- **Engine seams:** `memory_visible_objects` (perception), `iterate_feel_touch` (box), inventory transfer
  (`flows/inventory.md`), `alife_create_item`/`alife_clone_weapon`/`alife_release_id`.
- **Death seam:** `HasValuableLoot` is set from the death path (`flows/scheme-sleeper-wounded-death.md` /
  `flows/damage-hit.md`).
- **Camper hook:** camper (F75) and other schemes register the corpse/gather actions as sub-actions, so a
  camping NPC can loot a nearby corpse.

## Threading / failure / edge cases

- **Claim leaks:** if an NPC dies mid-loot without finalize, a `corpse_already_selected`/`Items[]` entry
  can linger and block others — the schemes clear on finalize, but interrupted flows are a known risk.
- **Overweight blocks gather:** an overweight NPC won't gather — a common "why won't it pick up" cause.
- **Box whitelist is narrow:** `axr_npc_vs_box` only targets a hardcoded box visual whitelist; other
  containers are ignored.
- **`cstackprevent` throttle:** detection isn't instant — it rescans memory ~4×/sec.

## Deeper

- Corpse: `unpacked/scripts/xr_corpse_detection.script` (`HasValuableLoot`, `evaluator_corpse`,
  `action_search_corpse`, `get_all_from_corpse`)
- Gather: `unpacked/scripts/xr_gather_items.script` (`eva_gather_itm`, `act_gather_itm`, `gather_item`)
- Box: `unpacked/scripts/axr_npc_vs_box.script`; stash object: `unpacked/scripts/xr_box.script`
- Framework: `flows/scheme-walker-patrol.md`; inventory transfer: `flows/inventory.md`

## Maintenance

- Verified: corpse `HasValuableLoot`. Promote the claim/loot bodies + gather to verified when a task
  needs exact loot rules.
- Note: `xr_box.script` ≠ the NPC scheme (it's the stash manager); the NPC scheme is `axr_npc_vs_box`.
- Check `gamedata/scripts/` for overrides before trusting (none at writing).
