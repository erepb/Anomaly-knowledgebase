---
domain: flows/dialog-system-lua
tags: [modlayer, gameplay, unpacked, lua, dialog, npc, flow]
sources: [unpacked/scripts/dialog_manager.script, unpacked/scripts/dialogs.script, unpacked/scripts/dialogs_zaton.script, unpacked/scripts/dialogs_axr_companion.script, unpacked/configs/misc/dialog_manager.ltx, unpacked/configs/gameplay/dialogs.xml, unpacked/configs/gameplay/character_dialogs.xml]
related: [flows/dialog-trade.md, flows/condlist-library.md, flows/scheme-meet-abuse-bribe.md, flows/task-system-lua.md, flows/dynamic-task-types.md, flows/relations-info.md, flows/scheme-engine-core.md, flows/gameplay-boot.md, modlayer/gameplay/dialogs-info.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: dialog system — Lua/data tier (F99)

How conversations are *authored and gated* on top of the engine phrase-graph machinery (F18,
`flows/dialog-trade.md` — `CPhraseDialog::SayPhrase`, functor resolution, info gates). Three tiers:
**attachment** (which dialogs an NPC owns, character XML), **static phrase graphs** (XML trees whose
nodes name Lua precondition/action functors), and **script-built graphs** (`<init_func>` → Lua
constructs the tree at dialog-init, the home of the dynamic-chatter, dynamic-task, broker, surrender
and companion menus). Plus the **dynamic-chatter engine** (`dialog_manager`) that fills generic NPC
small talk from `dialog_manager.ltx` with a per-NPC priority system.

**Provenance:** agent-researched, spine personally verified — `dialog_manager.script` READ-IN-FULL;
`dialogs.script` full function census + key bodies read (`who_is_npc`, `break_dialog`,
`update_npc_dialog`, `is_wounded`, `relocate_money`/`relocate_item_section`, the
`npc_has_task`/`npc_give_task`/`npc_skip_task`/`text_npc_has_task` task-offer set,
`is_npc_in_current_smart`, `set_enemy`); `_g.start_game_callback` caller, `character_dialogs.xml` +
`character_desc_zaton.xml` include, `hello_dialog`/`actor_break_dialog` XML, `dialog_manager.ltx`
section shape + category counts all personally verified. Level packs (`dialogs_<level>`),
`dialogs_axr_companion` bodies, and the long tail of `dialogs.script` *(agent-traced)*. **No
`gamedata/` override of `dialog_manager`/`dialogs`/`dialogs_*` (verified by listing).** Two agent
claims were corrected against code (see Failure cases: the anomalies `smart` gate is inverted and
dormant; `once` *pins* a phrase rather than hiding it).

## Tier 1 — attachment (who owns which dialogs)

Every `<specific_character>` in `configs/gameplay/character_desc_*.xml` ends with
`#include "gameplay\character_dialogs.xml"` (verified in `character_desc_zaton.xml`), which gives
every generic NPC: `<start_dialog>hello_dialog</start_dialog>` (the greeting the talk window opens
with) plus the universal `<actor_dialog>` list — `dm_init_stalker_trade`, the
`dm_*ordered_task*`/`dm_sim_ordered_task*` give/turn-in/cancel dialogs, `dm_delivery_dialog`,
`dm_guide_job`, the companion dialogs (`friendly_companion_dialog`, `dm_is_actor_companion_dialog`,
`dm_companion_patrol`/`leave`, `paid_companion_dialog`), `dm_universal_dialog` (the "let's talk"
chatter menu) and `actor_break_dialog`. `character_dialogs_no_guide.xml` is the reduced variant
(companions + universal + break only — no trade/task/guide rows). Story NPCs add their own
`<actor_dialog>` ids before the include. The engine filters this list per talk via each dialog's
own `<precondition>`/`<has_info>` gates (F18).

## Tier 2 — static XML phrase graphs

`configs/gameplay/dialogs.xml` + per-level packs (`dialogs_zaton.xml`, `dialogs_mlr.xml`, … —
26 files). A `<dialog>` carries optional dialog-level `<precondition>`/`<has_info>`/
`<dont_has_info>`; its `<phrase_list>` nodes carry `<text>` (string id), `<next>` edges (several =
player choice), `<precondition>`/`<action>` functor names, `<give_info>`/`<disable_info>`, optional
`<script_text>` (functor *returns* the display text — e.g. `actor_break_dialog` phrase 0 uses
`dialog_manager.create_bye_phrase` over the static text) and `<is_final>`. The named functors are
plain global Lua functions resolved by the engine (F18): shared library = `dialogs.script`,
per-level/story packs = `dialogs_<level>.script` (~14 files, same pattern: story preconditions like
money/item/PDA gates + actions doing transfers and info flips — `dialogs_zaton` sampled,
*(agent-traced)*). Signature convention: preconditions get
`(first_speaker, second_speaker, dialog_id, parent_phrase_id, phrase_id)`, actions
`(first_speaker, second_speaker, dialog_id, phrase_id)`; either speaker may be the actor —
`dialogs.who_is_npc` returns whichever side isn't `AC_ID`.

## Tier 3 — script-built graphs (`<init_func>`)

A `<dialog>` with `<init_func>some_module.fn</init_func>` hands the freshly loaded dialog object to
Lua, which builds the phrase tree imperatively. The bridge is `dialog_manager.add_dialog(parent_id,
id, phrase_id, cond, act)`: stores the dialog in a module-local `saved_dialog`, calls
`saved_dialog:AddPhrase(phrase_id, id, parent_id, -10000)` then wires `AddPrecondition`/`AddAction`
on the phrase's `GetPhraseScript()`; `cond`/`act` accept a string or a table of strings, each
validated with `utils_data.findfunction` — a missing functor is `printe`-logged and **skipped**
(unlike XML-named functors, which the engine hard-asserts on, F18). `add_script_dialog` is the same
plus `SetScriptText(phrase_id)` — the phrase text comes from calling that functor.

Script-built consumers (all in `dialog_manager.script` unless noted): `init_hello_dialogs`
(hello), `init_new_dialog` (`dm_universal_dialog`), `dm_init_dynamic_task_dialog` /
`_sim_` / `_completed_` / `_cancel_` variants (a 10-iteration loop of offer→accept/skip nodes —
the task-giver UI over `axr_task_manager`), `dm_init_dynamic_broker_dialog` (PDA fence),
`dm_init_dynamic_repair_dialog` (delegates functors to `inventory_upgrades`),
`dm_init_surrender_dialog`, the DRX storyline set (`drx_sl_init_honcho_meet_dialog` etc.), the
`dm_agr_u_bandit_boss_*` story set (companion recruit via `dialogs_agr_u`), plus
`travel_manager.init_traveler_dialog` (guide travel) and the companion command menus in
`dialogs_axr_companion` *(agent-traced)*.

## The dynamic-chatter engine (`dialog_manager`)

1. **Boot:** `_g.start_game_callback` (F65) calls `dialog_manager.fill_phrase_table()` — parses
   `misc\dialog_manager.ltx` (854 sections) into `phrase_table[category][id]`. Live categories &
   counts at writing: `information` 715, `anomalies` 58, `tips` 39, `hello` 31, `job` 11; `place`
   is declared but has **zero sections** (dormant — its fill functor is commented out). Per
   section: `npc_community`/`actor_community`/`level` (name lists or `all`/unset), `once`,
   `wounded`, `info` gates (`xr_logic.parse_infop1` +/- info names), Tronex `condition` (condlist)
   + `effect` (condlist) + `important`, and `smart` (unused in the shipped LTX — see Failure
   cases). The phrase id doubles as the localization string id (`dm_job_0` …).
2. **Tree build:** `hello_dialog` (`<start_dialog>` of every NPC) → `init_hello_dialogs` →
   `init_start_dialogs`: root chain whose first action is `fill_priority_hello_table`, then one
   phrase node per LTX entry gated by `dialog_manager.precondition_hello_dialogs` + (for
   `wounded = true` entries) `dialogs.is_wounded`, with medkit-offer (`dialogs.actor_have_medkit` →
   `transfer_medkit` + `break_dialog`) / sorry sub-phrases. `dm_universal_dialog` →
   `init_new_dialog`: four submenus (job/anomalies/information/tips), each with a
   `fill_priority_X_table` action on entry plus `dm_no_more_*`/`dm_do_not_know_*` fallback phrases
   gated by `precondition_X_dialogs_no_more`/`_do_not_know`.
3. **Priority pick:** `fill_priority_table` → `calculate_priority` per phrase per NPC into
   `priority_table[category][npc_id][phrase_id]`: `important` +255; `npc_community` exact match
   +2 / `all` +1 / mismatch −1 *and* disqualify; `level` match +2 / `all` +1 / mismatch
   disqualify; `actor_community` via `get_actor_true_community` (disguise-aware) +1 / mismatch
   disqualify; wounded is an XOR gate (wounded-phrases only for wounded NPCs, normal phrases only
   for healthy ones); info gates and a `condition` condlist evaluating to `false` force −1; a
   stored 255 overrides everything. `precondition_X_dialogs` recalculates the phrase then shows it
   only if `is_highest_priority_phrase` (ties: first iterated wins).
4. **After a phrase:** `action_X_dialogs` → `action`: on the NPC's *first* phrase pick, a
   `once = true` phrase is pinned to 255 (`set_phrase_highest_priority`) — meaning the NPC commits
   to that line **forever** (the same greeting repeats); the Tronex `effect` condlist fires; then
   `ignore_once` is set, which makes `calculate_priority` disqualify all *other* once-phrases for
   that NPC. The four `dm_universal_dialog` categories additionally call `told` — `precondition`
   returns false for the whole category once told, so each NPC answers job/anomalies/information/
   tips **once per session** and falls back to `dm_no_more_*` after. Hello never sets told.
5. **State is runtime-only:** `phrase_table` is rebuilt each game start; `priority_table`,
   `disabled_phrases`, `quest_disabled_phrases` live in Lua tables with **no save/load** — chatter
   memory resets every session. `disabled_phrases[npc_id]` (fed by `action_disable_phrase` /
   `action_disable_quest_phrase`, read by `precondition_is_phrase_disabled`) is additionally
   cleared per NPC each time the NPC is used: `xr_motivator`'s use callback nils it and rerolls
   `RandomVal` (the bye-phrase variant `create_bye_phrase` returns).

## `dialogs.script` — the shared functor library (census)

~250 global functions in families (key bodies read; tail *(agent-traced)*): **plumbing**
(`who_is_npc`, `break_dialog` = `stop_talk` both sides, `update_npc_dialog` = re-tick
`meet_manager` + `xr_meet.process_npc_usability` + `xr_logic.try_switch_to_another_section` —
refreshes the F79 meet state mid-conversation); **transfers** (`relocate_money(victim, num,
in|out)` + `relocate_item_section*` → `itms_manager` + `news_manager` HUD lines); **predicates**
(actor/NPC community per faction, `is_friend`/goodwill tiers, `level_*`, `actor_has_item`,
`is_wounded` → `xr_wounded`, `is_npc_in_current_smart` → `xr_gulag.get_npc_smart`); **task-offer
glue** (`generate_available_tasks`/`npc_has_task`/`text_npc_has_task`/`npc_give_task`/
`npc_skip_task`/cancel/turn-in twins + `sim_*` variants — read `axr_task_manager.available_tasks`/
`finished_tasks`, give via `task_manager.get_task_manager():give_task` (F97), task-count cap from
`ui_options` max_tasks); **DRX storyline** (`drx_sl_*` honcho/change-faction/mechanic task text +
give + turn-in); **trade/upgrade entries** (`npc_is_trader` = `npc:start_trade(db.actor)`,
`npc_is_tech` = `npc:start_upgrade(db.actor)` — the dialog→trade-UI seam of F18;
`free_trade_with_npc` = `npc:use`); **broker** (`text_trade_npc_pda` prices/classifies PDAs via
`se_load_var` state, transfers + pays, warfare traders get items `alife_release`d instead
*(agent-traced)*); **surrender** (`victim_is_bounty`/`set_victim_surrendered`/ransom answers
*(agent-traced)*); **hostage rescue**, medkit/bandage/pistol transfers.

## Isolation ledger (the seams)

- **Engine phrase graph (F18):** XML `<phrase>` tags + `AddPhrase`/`AddPrecondition`/`AddAction`/
  `SetScriptText` on the dialog object; engine evaluates preconditions, fires actions, manages
  `give_info`/`disable_info`. This flow is everything *behind* the functor names.
- **Condlists (F67/F68):** `xr_logic.parse_infop1` (LTX `info` gates), `parse_condlist` +
  `pick_section_from_condlist` (Tronex `condition`/`effect` fields, task `on_job_descr`).
- **Info portions (F40):** `has_alife_info` in `calculate_priority`; phrase-level info transfer is
  engine-side.
- **Meet scheme (F79):** `xr_meet` decides *whether* dialog is allowed (`process_npc_usability`);
  `dialogs.update_npc_dialog` re-runs it mid-dialog; `allow_wounded_dialog` exists for the wounded
  path.
- **Motivator/use (F66/F88):** `xr_motivator`'s use callback clears `disabled_phrases[npc_id]` and
  rerolls `RandomVal` per conversation.
- **Tasks (F97/F98/F114):** the offer/turn-in/cancel dialogs are the *UI* of `axr_task_manager`
  (F114 ⬜) + `task_manager` (F97); dynamic-task setup effects fire from these phrases (F98).
- **Trade/upgrade (F18/F26):** `npc:start_trade`/`start_upgrade` jump out of dialog into the trade
  and upgrade windows.
- **World/sim:** `xr_gulag.get_npc_smart` (anomalies smart gate), `xr_wounded` (wounded gating),
  `news_manager`/`itms_manager` (transfer side effects), `game_relations`/`warfare`
  *(agent-traced)* in the long tail.
- **Data:** `misc\dialog_manager.ltx` (chatter), `gameplay\dialogs*.xml` (graphs),
  `gameplay\character_dialogs(_no_guide).xml` + `character_desc_*.xml` (attachment),
  localization string ids = phrase ids.

## Threading

All on the main thread inside the engine's talk-window update (F18). No timers, no scheduled work;
the only cross-frame state is the module-local tables.

## Failure & edge cases (verified unless noted)

- **`precondition_no_more` latent bug:** after the early-outs it indexes `PRT_subtable` — a **nil
  global** (it's only ever a parameter elsewhere in the file). Whenever a category *does* have an
  available phrase (`pr >= 0`), the expression raises a Lua error. Net effect: the "do not know"
  phrase only works via the early-return paths; the error path's engine-side handling — presumably
  precondition treated as failed, which coincidentally is the wanted outcome — is *(INFERRED)*.
- **Anomalies `smart` gate is inverted and dormant:** `precondition_anomalies_dialogs` *disables*
  the phrase when the NPC's gulag name **equals** the phrase's `smart` field (an NPC won't recite
  the entry tagged with its own smart). The shipped LTX contains **no `smart` fields at all**, so
  the branch never fires. (Corrects the agent draft, which claimed "shown only at that smart".)
- **`once` ≠ "never again":** the 255 pin means the chosen once-phrase becomes permanently
  highest-priority — the NPC *repeats* it (stable greeting), while `ignore_once` suppresses all
  other once-phrases for that NPC. (Corrects the agent draft.)
- **`init_new_dialog` `ph` flag never resets** across the four category iterations — the
  per-category "do not know" fallback for a later empty category can be skipped if an earlier one
  had phrases. Harmless with the shipped LTX (no empty live category).
- **Missing functor severity differs by tier:** script-built phrases skip the functor with a
  `printe` log; XML-referenced functors are fatal in the engine (F18 `THROW3`/`R_ASSERT`).
- **Zombied/monolith gate:** `uni_dialog_precond` (referenced from `dialogs.xml`) blocks the
  universal dialog for `zombied`/`monolith` NPCs.
- **No persistence:** all chatter/priority/told state resets on session restart; only info
  portions and task state (engine/F97 side) survive saves.

## Deeper

- Dynamic chatter + script-built trees: `unpacked/scripts/dialog_manager.script` (whole file);
  data `unpacked/configs/misc/dialog_manager.ltx`
- Shared functors: `unpacked/scripts/dialogs.script`; story packs `unpacked/scripts/dialogs_*.script`
- Graphs/attachment: `unpacked/configs/gameplay/dialogs*.xml`,
  `character_dialogs(_no_guide).xml`, `character_desc_*.xml`
- Engine half: `flows/dialog-trade.md` (F18); meet gating: `flows/scheme-meet-abuse-bribe.md` (F79)
- Map: `modlayer/gameplay/dialogs-info.md` (M14)

## Maintenance

- Level-pack bodies (`dialogs_<level>`), `dialogs_axr_companion`, broker/surrender internals are
  agent-traced — promote when a story/companion task needs them.
- Re-check `gamedata/scripts/dialog*` for overrides per session (none at writing).
- If `dialog_manager.ltx` gains `smart =` or `place` sections, the dormant branches go live —
  re-verify the inverted smart gate then.
