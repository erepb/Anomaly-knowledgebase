---
domain: flows/scheme-sleeper-wounded-death
tags: [modlayer, gameplay, unpacked, lua, schemes, wounded, death, sleeper, flow]
sources: [unpacked/scripts/xr_sleeper.script, unpacked/scripts/xr_wounded.script, unpacked/scripts/xr_help_wounded.script, unpacked/scripts/xr_eat_medkit.script, unpacked/scripts/xr_death.script, unpacked/scripts/xr_smartcover.script]
related: [flows/scheme-walker-patrol.md, flows/scheme-combat.md, flows/scheme-engine-core.md, flows/damage-hit.md, flows/state-manager-core.md, flows/smart-cover.md, modlayer/gameplay/npc-schemes.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: sleeper / wounded / help-wounded / eat-medkit / death (F78)

A cluster of survival/incapacitation schemes — **deltas on the walker template** (F74). **sleeper** =
sleep at a point; **wounded** = the incapacitation hub that gates combat and lets an NPC be finished or
recover; **help_wounded** = an ally walks over and heals a wounded NPC; **eat_medkit** = self-heal
(callback-driven, not action-based); **death** = on-death callback (loot/info/killer). **smartcover** =
generic smartcover occupation (named only). Read F74 first.

**Provenance:** **verified** spine points: `xr_wounded.script:evaluator_can_fight` (returns
`load_var(npc,"wounded_fight") ~= "false"` — the combat gate) and `Cwound_manager:update`/`:eat_medkit`
exist; `xr_death.script:action_process_death:death_callback` (records
`db.storage[victim:id()].death.killer`) read; `xr_sleeper.script` symbols (`set_scheme`,
`action_sleeper_activity:reset_scheme`/`callback`) confirmed present. `help_wounded`, `eat_medkit`
bodies, and `smartcover` are *(agent-traced — not personally re-read)*. No `gamedata/` overrides of any
of the six files (verified by listing).

## Wounded — the incapacitation hub (verified gate)

- **Enter:** `evaluator_wound` flips when the wounded condlist/HP threshold is met (config via
  `init_wounded`); the wounded action takes over with `npc:wounded(true)` and a wounded `state_mgr`
  state. `Cwound_manager:update` (verified present) reads HP/psy and picks the wounded state/sound from
  the `hp_state`/`hp_state_see`/`psy_state` tables, and **saves `wounded_fight`** (verified:
  `save_var(npc,"wounded_fight", self.fight)`).
- **Combat gate (verified):** `evaluator_can_fight:evaluate` returns `wounded_fight ~= "false"` — when a
  wounded NPC's config says it cannot fight, this evaluator is false and **gates the combat/danger
  planners**, so the NPC stops fighting. This is the key cross-scheme effect: wounded suppresses combat.
- **Recover/finish:** auto-heal after a timeout via `Cwound_manager:eat_medkit` (verified present: sets
  health to 1, releases medkits, `help_thanks`), or the NPC can be finished (kill-wounded). A monolith
  wounded near the actor may suicide-grenade *(agent-traced)*.

## Death — on-death callback (verified)

`xr_death.script:action_process_death:death_callback(victim, who)` (verified) records the killer:
`db.storage[victim:id()].death.killer = who:id()` (or `-1` if unknown). `setup_generic_scheme`/
`add_to_binder` wire it; `reset_generic_scheme` parses `on_info`/`on_info2` condlists applied on death
(via `pick_section_from_condlist`). Death is **callback-only** — no per-frame action loop.

## Sleeper — sleep machine (verified present)

`set_scheme` reads `path_main` + `wakeable`; `action_sleeper_activity:reset_scheme` parses the path and
`move_mgr:reset`; `action_sleeper_activity:callback` fires on reaching the bed waypoint and
`state_mgr.set_state("sit"|"sleep", {look_position})` (sit if `wakeable`, else sleep). Standard walker
preconditions (alive/no-danger/no-enemy) make it yield to threats.

## Help-wounded / eat-medkit / smartcover (agent-traced)

- **`xr_help_wounded`** — `evaluator_wounded_exist` scans `db.OnlineStalkers` for an alive wounded ally
  (distance/relation/`not_for_help` checks; marks `wounded_already_selected`); the action navigates there
  and on arrival `help_wounded()` does `alife_create_item("medkit_script", ...)` +
  `xr_wounded.unlock_medkit` + a thanks sound. Gates the combat planner while helping.
- **`xr_eat_medkit`** — **not** an action; a `npc_on_update` callback gated by `ai_tweaks\xr_eat_medkit.ltx`
  (`plugin.enable`). On HP/bleed thresholds it schedules a delayed `consume_medkit` (release item) then
  heals over ~15 ticks. Combat-context gated (`in_combat`/`out_combat`).
- **`xr_smartcover`** — generic smartcover occupation: a `smartcover` `state_mgr` state with peace/combat
  action variants; blocked when wounded (`sidor_wounded_base=true`). Detail → F33 `flows/smart-cover.md`.

## Function digests (spine)

- **`xr_wounded.script:evaluator_can_fight:evaluate`** — *Does:* `wounded_fight ~= "false"` → gates
  combat/danger. *(verified.)*
- **`xr_wounded.script:Cwound_manager:update` / `:eat_medkit`** — *Does:* pick wounded state + save
  `wounded_fight` / auto-heal. *(verified present.)*
- **`xr_death.script:action_process_death:death_callback`** — *Does:* record `death.killer`; apply
  on-death info. *(verified.)*
- **`xr_sleeper.script:action_sleeper_activity:callback`/`reset_scheme`/`set_scheme`** — *Does:* walk to
  bed → sit/sleep. *(verified present.)*
- **`xr_help_wounded` / `xr_eat_medkit` / `xr_smartcover`** — *Does:* heal allies / self-heal / occupy
  cover. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** per-NPC savevars (`wounded_fight`, `wounded_state`, `wounded_victim`,
  `wounded_already_selected`), `db.storage[id].death.killer`. Same scheme framework (F66/F74).
- **Combat gate seam:** wounded/help-wounded gate `action_combat_planner` via their evaluators (F76).
- **Engine seams:** `npc:wounded(true)` flag; the **death callback** (`flows/damage-hit.md`); `state_mgr`
  states (F70); `alife_create_item`/`alife_release_id` (medkits).
- **Config seams:** `init_wounded` tables, `ai_tweaks\xr_eat_medkit.ltx`.

## Threading / failure / edge cases

- **Wounded is the hub:** an NPC that "won't fight after being shot" is in wounded state with
  `wounded_fight=="false"` — check that savevar (verified gate).
- **eat_medkit is plugin-gated:** disabled if `xr_eat_medkit.ltx plugin.enable=false`; it is a callback,
  not a scheme — it won't show as an active scheme/section.
- **Death is one-shot:** the death callback fires once; on-death info portions apply then. No loop.
- **help_wounded contention (agent-traced):** `wounded_already_selected` prevents multiple healers
  racing for the same ally.

## Deeper

- Wounded: `unpacked/scripts/xr_wounded.script` (`evaluator_can_fight`, `Cwound_manager:update`/
  `:eat_medkit`); death: `unpacked/scripts/xr_death.script` (`death_callback`)
- Sleeper: `unpacked/scripts/xr_sleeper.script`; help: `unpacked/scripts/xr_help_wounded.script`;
  self-heal: `unpacked/scripts/xr_eat_medkit.script`; smartcover: `unpacked/scripts/xr_smartcover.script`
- Template: `flows/scheme-walker-patrol.md`; combat gate: `flows/scheme-combat.md`; hit/death:
  `flows/damage-hit.md`; smartcovers: `flows/smart-cover.md`

## Maintenance

- Verified core: the wounded combat-gate (`evaluator_can_fight`→`wounded_fight`) + death killer-record;
  promote help_wounded/eat_medkit bodies on next read.
- Check `gamedata/scripts/` for overrides before trusting (none at writing — but wounded/death are
  commonly modded; re-verify per session).
