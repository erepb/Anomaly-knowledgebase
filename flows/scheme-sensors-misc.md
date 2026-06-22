---
domain: flows/scheme-sensors-misc
tags: [modlayer, gameplay, unpacked, lua, schemes, sound, sensors, flow]
sources: [unpacked/scripts/xr_sound.script, unpacked/scripts/xr_hear.script, unpacked/scripts/xr_zones.script, unpacked/scripts/xr_zones_sound.script, unpacked/scripts/xr_detector.script, unpacked/scripts/xr_weapon_jam.script, gamedata/scripts/xr_weapon_jam.script, unpacked/scripts/xr_reach_task.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-engine-core.md, flows/ai-perception.md, flows/ai-squad-logic.md, flows/anomaly-zones.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: sensor / misc schemes & services (F81)

A cluster of NPC support modules. **Important correction (from research):** most of these are **not
standalone schemes** — `xr_sound` and `xr_hear` are shared services called from `xr_motivator`, `xr_zones`
is a zone-object binder, `xr_detector` is **actor-only**, and `xr_weapon_jam` is a callback service with
a **gamedata override**. Only `xr_reach_task` is a full scheme. Documented as a cluster so the catalog is
complete.

**Provenance:** the **gamedata override of `xr_weapon_jam` is verified** (it is the only batch-6 file with
a `gamedata/scripts/` copy — confirmed by listing). The per-module spines are *(agent-traced — not
personally re-read)*; the "what each thing actually is" classification is the verified, load-bearing part.

## What each module is (the key facts)

- **`xr_sound.script` — shared sound service** (no override). API `set_sound_play(npc_id, theme, …)` +
  per-frame `update(npc_id)` (called from `xr_motivator:update`, verified in F66). Many schemes
  (walker idle sound, meet hello/bye, combat) call it. Manages per-NPC sound themes + completion
  callbacks. *(agent-traced.)*
- **`xr_hear.script` — hear callback** (no override). `hear_callback(obj, who, sound_type, pos, power)`
  registered as the NPC hear hook (called from `xr_motivator`); decodes the sound-type bitmask, matches
  `on_sound` rules parsed from the logic section, and can `xr_logic.switch_to_section` on a match (audio
  triggers a logic switch). Fires `npc_on_hear_callback`. *(agent-traced.)*
- **`xr_zones.script` — zone-object binder, NOT an NPC scheme** (no override). Binds arena/script zone
  objects (`on_enter`/`on_exit` tracking). **NPC anomaly *reaction* is not here** — it's in
  danger/combat (F76) and the engine. (Correction to the naive "NPC zone scheme" assumption.)
- **`xr_zones_sound.script` — zone ambient audio** (no override): plays randomized ambient sounds when an
  object enters a zone. *(agent-traced.)*
- **`xr_detector.script` — ACTOR-only** (no override): the player's PDA detector / dead-zone radiation
  logic. **There is no NPC detector scheme here** (correction). *(agent-traced.)*
- **`xr_weapon_jam.script` — misfire service with a GAMEDATA OVERRIDE** (verified the override exists):
  - `unpacked/` version: a `npc_on_update` tick that tracks ammo spent and, per NPC rank, may
    `unload_magazine()` (a jam).
  - `gamedata/` override (**the live one**): a callback architecture — `GetConditionMisfireProbability`
    sends `npc_get_misfire_probability` and returns `t.ret_value` (sets `=1` instead of unloading) so the
    **engine** applies the misfire. Same rank-based chance; different integration (callback flag vs direct
    unload). *(bodies agent-traced; the override's existence + shape verified.)*
- **`xr_reach_task.script` — the one real scheme** (no override): drives an NPC/squad to its ALife
  smart-terrain task target. `evaluator_reached_task_location` (reached?) + `action_reach_task_location`
  with commander/soldier/single execute (formation pathing); replaces the engine
  `action_smart_terrain_task`. Ties to F12 (`flows/ai-squad-logic.md`). *(agent-traced.)*

## Function digests (spine)

- **`xr_sound.script:set_sound_play` / `update`** — *Does:* enqueue + tick per-NPC sound themes. *Called
  by:* schemes + `xr_motivator:update`. *(agent-traced.)*
- **`xr_hear.script:hear_callback`** — *Does:* react to heard sounds → maybe switch section. *(agent-traced.)*
- **`xr_weapon_jam.script` (gamedata) `GetConditionMisfireProbability` / `npc_misfire`** — *Does:*
  rank-based misfire via engine callback (`t.ret_value`). *(override existence verified; body agent-traced.)*
- **`xr_reach_task.script:evaluator_reached_task_location` / `action_reach_task_location`** — *Does:* path
  the NPC/squad to its smart-terrain task target. *(agent-traced.)*
- **`xr_zones.script` / `xr_detector.script`** — *Does:* zone-object binder / actor PDA detector — **not
  NPC schemes** (classification verified by reading their roles).

## Isolation ledger (the seams)

- **Services vs schemes:** `xr_sound`/`xr_hear`/`xr_weapon_jam` are services hooked from `xr_motivator`/
  engine callbacks (F66), not action-manager schemes. `xr_reach_task` is the only action-manager scheme.
- **Engine seams:** sound playback (`npc:play_sound`), hear callback (`npc:set_hear_callback`), misfire
  (`GetConditionMisfireProbability` engine hook), reach-task replaces `action_smart_terrain_task`.
- **Logic seam:** `xr_hear` can `switch_to_section` (F66/F68) — audio-driven logic.
- **Override seam:** `xr_weapon_jam` is in `gamedata/` (the live version) — re-check it, not the unpacked
  copy (`METHODOLOGY.md`).
- **Not-here:** NPC anomaly avoidance (danger/combat F76); actor detector is a separate actor concern.

## Threading / failure / edge cases

- **Don't treat `xr_zones`/`xr_detector` as NPC schemes** — a common misread; the file names suggest NPC
  behavior but they are a zone binder and actor PDA respectively.
- **`xr_weapon_jam`: read the gamedata copy** — the unpacked `unload_magazine` path is superseded by the
  callback-flag override; conclusions from the unpacked version will be wrong for the live build.
- **`xr_sound` is shared** — a sound bug may originate in the calling scheme, not the service.

## Deeper

- Sound/hear: `unpacked/scripts/xr_sound.script`, `unpacked/scripts/xr_hear.script`
- Zones: `unpacked/scripts/xr_zones.script` (binder), `unpacked/scripts/xr_zones_sound.script` (ambient)
- Detector (actor): `unpacked/scripts/xr_detector.script`
- Misfire: `gamedata/scripts/xr_weapon_jam.script` (**live**) vs `unpacked/scripts/xr_weapon_jam.script`
- Reach task: `unpacked/scripts/xr_reach_task.script`; squad/task: `flows/ai-squad-logic.md`; framework:
  `flows/scheme-engine-core.md`

## Maintenance

- Verified core: the classification (services vs scheme; `xr_detector` actor-only; `xr_zones` binder) +
  the `xr_weapon_jam` gamedata override existence. Promote the per-module bodies on next read.
- Always read the `gamedata/` `xr_weapon_jam` (the live one); re-check for new overrides per session.
