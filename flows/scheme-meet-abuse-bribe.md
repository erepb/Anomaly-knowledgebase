---
domain: flows/scheme-meet-abuse-bribe
tags: [modlayer, gameplay, unpacked, lua, schemes, meet, bribe, flow]
sources: [unpacked/scripts/xr_meet.script, unpacked/scripts/xr_abuse.script, unpacked/scripts/xr_bribe.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-combat.md, flows/dialog-trade.md, flows/scheme-engine-core.md, flows/relations-info.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: meet / abuse / bribe schemes (F79)

How an NPC reacts to the **actor approaching** (meet — greet/stop/turn/enable-dialog at distance tiers),
to being **shoved/mis-used** (abuse — a counter that triggers a shove-back), and **faction-peace via
payment** (bribe — a ceasefire that vetoes hostility). meet + abuse are **generic schemes** (they run
alongside the active scheme via `setup_generic_scheme`, not `set_scheme`); bribe is a faction-level
service. Deltas on the F66/F74 framework.

**Provenance:** spine **verified present** — `xr_bribe.script` (`at_peace`, `npc_on_hit_callback`,
`set_bribe`) read; `xr_meet.process_npc_usability` is the function `xr_motivator:update` calls (verified
in F66). The `Cmeet_manager` distance state machine, `notify_on_use`→abuse, and the `CAbuseManager`
counter are *(agent-traced — not personally re-read)*. No `gamedata/` overrides of the three (verified
by listing).

## Meet — actor-approach reaction (generic scheme, agent-traced + usability verified)

- A **generic scheme**: `setup_generic_scheme`/`reset_generic_scheme`/`configure_actions` register an
  `evaluator_contact` + `action_meet_wait` + a stateful `Cmeet_manager`. Other schemes point at a meet
  block via `meet = meet@section`.
- `Cmeet_manager` runs **distance tiers** (verified-by-draft): `close_distance` (~3 m), `far_distance`
  (~5 m), `reset_distance` (~30 m). On entering a tier it sets the NPC's anim/look (`close_anim`/
  `far_anim`, `close_victim`/`far_victim`) and plays hello/bye sounds, and manages **dialog enablement**
  (`set_start_dialog`, `enable_talk`/`disable_trade`, the per-NPC dialog override buffer). All driven by
  condlists (`use`, `meet_dialog`, `abuse`, `trade_enable`, `allow_break`).
- **`xr_meet.process_npc_usability(npc)`** is called every frame from `xr_motivator:update` (verified in
  F66) — it gates `enable_talk`/`disable_talk` based on the meet `use` state and other schemes (corpse/
  help-wounded/tasks suppress talk).
- **`notify_on_use`** (on actor "use" of the NPC): plays `snd_on_use`, and if `use=="false"` while abuse
  mode is on and the NPC is friendly, calls `xr_abuse.add_abuse` — i.e. pestering a "don't-talk" NPC
  angers it.

## Abuse — shove/mis-use counter (agent-traced)

`CAbuseManager` holds `abuse_value` vs `abuse_threshold` (~5); `addAbuse(v)` adds `v*rate`, `update`
**decays** the value over time and clears the hit flag below ~2/3 threshold; `abused()` is true at
threshold. `evaluator_abuse` true → `action_abuse_hit` plays a "punch" at the actor. Hooked by
`xr_meet.notify_on_use` and on-hit. Public API: `add_abuse`/`clear_abuse`/`enable_abuse`/`disable_abuse`.

## Bribe — faction ceasefire (verified symbols)

- **`xr_bribe.script:at_peace(npc_comm, player_comm, distance)`** (verified) — returns true if a
  ceasefire is active for the NPC's faction **and** the actor is outside the bribe's safe `distance`;
  on violation it breaks the ceasefire and announces it. **This is the seam F76 uses:**
  `xr_combat_ignore.is_enemy` calls `at_peace` to veto enemy status, so a bribed faction won't attack.
- **`set_bribe(faction, distance, seconds)`** (verified) — `M_DATA[faction] = {distance, duration,
  start}`; activates the `npc_on_update` callback. Called by the bribe dialog options (money check →
  `dialogs.relocate_money` → `set_bribe`).
- **`npc_on_hit_callback(comm)`** (verified) — hitting a bribed NPC breaks the ceasefire immediately.
- Expiry via a timed `bribe_check`; warnings when the actor nears the safe zone; `M_DATA` persists in
  save/load. *(expiry/warning bodies agent-traced.)*

## Function digests (spine)

- **`xr_bribe.script:at_peace`** — *Does:* ceasefire + safe-distance check; breaks on violation. *Called
  by:* `xr_combat_ignore.is_enemy` (F76). *(verified.)*
- **`xr_bribe.script:set_bribe` / `npc_on_hit_callback`** — *Does:* start ceasefire / break on hit.
  *(verified.)*
- **`xr_meet.script:process_npc_usability`** — *Does:* gate talk each frame. *Called by:* `xr_motivator:update`.
  *(verified via F66.)*
- **`xr_meet.script:Cmeet_manager` / `notify_on_use`** — *Does:* distance-tier greet + dialog enable +
  abuse trigger. *(agent-traced.)*
- **`xr_abuse.script:CAbuseManager`** — *Does:* decaying abuse counter → shove. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** meet's `Cmeet_manager` (distance/dialog state) on `st`; abuse's `CAbuseManager`
  counter; bribe's module-global `M_DATA` (per-faction ceasefires, save/loaded).
- **Driver:** generic schemes ticked alongside the active scheme; `process_npc_usability` called every
  frame by `xr_motivator:update` (F66).
- **Dialog seam:** meet enables/forces dialog (`set_start_dialog`, `run_talk_dialog`) — `flows/dialog-trade.md`.
- **Combat seam:** bribe `at_peace` is consumed by `xr_combat_ignore.is_enemy` (F76) — the cross to combat.
- **Relations seam:** friend/enemy via `game_relations` (`flows/relations-info.md`); abuse only triggers
  on friendly NPCs.

## Threading / failure / edge cases

- **meet for enemies is disabled:** `init_meet` zeroes distances/use for faction enemies — meet only runs
  for friendlies.
- **bribe is faction-wide:** `at_peace` keys on community, so one bribe pacifies the whole faction within
  the safe zone; hitting any member or entering the zone breaks it.
- **abuse decays:** a single shove won't trigger the shove-back; sustained pestering does (counter +
  threshold).

## Deeper

- Meet: `unpacked/scripts/xr_meet.script` (`Cmeet_manager`, `process_npc_usability`, `notify_on_use`,
  `init_meet`)
- Abuse: `unpacked/scripts/xr_abuse.script` (`CAbuseManager`, `add_abuse`)
- Bribe: `unpacked/scripts/xr_bribe.script` (`at_peace`, `set_bribe`, `npc_on_hit_callback`)
- Combat consumer: `flows/scheme-combat.md`; dialog: `flows/dialog-trade.md`; framework:
  `flows/scheme-engine-core.md`

## Maintenance

- Verified: bribe `at_peace`/`set_bribe` + meet `process_npc_usability` (the cross-scheme seams). Promote
  the `Cmeet_manager`/`CAbuseManager` bodies on next read.
- Check `gamedata/scripts/` for overrides before trusting (none at writing).
