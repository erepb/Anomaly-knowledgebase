---
domain: flows/ui-news-tips
tags: [modlayer, gameplay, unpacked, lua, ui, news, pda, tips, dynamic-news, flow]
sources: [unpacked/scripts/news_manager.script, unpacked/scripts/dynamic_news_manager.script, unpacked/scripts/dynamic_news_helper.script, unpacked/scripts/faction_expansions.script]
related: [INDEX.md, modlayer/gameplay/ui-scripts.md, flows/pda-map-news.md, flows/pda-tabs.md, flows/gameplay-ui-screens.md, flows/ui-debug-tools.md, flows/options-runtime.md, flows/item-behaviors-lua.md, flows/game-services.md, flows/faction-relations-lua.md, flows/offline-combat.md, flows/sound-music.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-12
verified: code
---

# Flow: UI news & tips

The PDA notification layer: the small popups in the top-right ("tip from Duty: …"), the bottom-left
item/money/treasure showcase icons, and the ambient **Dynamic News** radio chatter that invents a
living Zone out of procedurally composed messages. Two tiers sit here — a universal **dispatcher**
(`news_manager`) that ~50 scripts call to push a notification, and a self-contained **generator**
(`dynamic_news_manager` + `dynamic_news_helper`) that schedules and composes ambient news on its own.
Both ultimately call the engine's `give_game_news` queue (the data layer is F39,
`flows/pda-map-news.md`). This is the gameplay UI tier (cluster of `modlayer/gameplay/ui-scripts.md`,
M20); the engine news queue and HUD widget are F39.

**Provenance:** code-verified — `news_manager` (263 lines) and `dynamic_news_helper` (749 lines) were
**READ-IN-FULL**; `dynamic_news_manager` (2857 lines) had its **spine read in full** (control/options
`update_settings`, singleton lifecycle `get_dynamic_news`/`actor_on_update`/`on_game_start`, the
`DynamicNewsManager:__init`/`destroy` channel+queue+tick setup, `TickNews`/`TickQuick` drain,
`PushToChannel` enqueue, the `NewsToggle` gate) with the ~90 message-composer methods
**structure-surveyed** (named function list + signatures, not each body) — flagged *(composer
surveyed)* where relevant. `faction_expansions` (the profile data tier) was sampled.

## Tier 1 — `news_manager`: the universal dispatcher (verified, READ-IN-FULL)

The thin Lua front over the engine notification calls (`give_game_news` for HUD popups,
`give_talk_message2` for the in-dialog iconed message). ~50 scripts across dialogs, tasks, trade,
items, statistics and combat call it; it is the one place that knows the icon/sound/caption
conventions. The pieces:

- **`send_tip(actor, news_id, timeout, sender, showtime, sender_id)`** — the generic PDA tip. Guards
  on the sender being heavy-wounded or dead, plays the `pda_tips` sound (and, if a matching
  `sound_theme` entry exists for the `news_id`, a spoken script sound whose length extends the showtime),
  resolves the icon (a faction icon from `tips_icons`, or the NPC's `character_icon()`), translates
  `st_tip` + the `news_id` string, and pushes `give_game_news(…, type 0)`.
- **`send_sound(npc, faction, point, str, str2, delay_sound)`** — the voice/faction news path
  (`give_game_news` type 1): converts a `characters_voice\…` path into a localizable key, builds a
  "tip <faction>. <smart name>:" caption, picks the icon from npc/faction/point.
- **`send_task(actor, type, tsk)`** — task-state news (`new`/`complete`/`fail`/`reversed`/`updated`):
  plays `pda_task`, uses the task title + icon, routes to `give_talk_message2` when the actor is in a
  dialog else `give_game_news`.
- **`send_treasure(param)`** — the stash-found showcase (new / got / old).
- **`relocate_item` / `relocate_money`** — the bottom-left in/out/docs showcase icons (item name + `xN`,
  or a money amount), also dialog-aware.
- **`on_game_start`** loads `plugins\news_tips_icons.ltx` into the `tips_icons` faction→texture map.

Everything routes through the F39 engine queue; `news_manager` adds no scheduling — it fires
immediately when called.

## Tier 2 — Dynamic News: the ambient generator

`dynamic_news_manager` (ARS Team → Alundaio → MLR → Tronex lineage) is a **singleton simulation** that
fabricates a living radio network: stalkers reporting deaths, loot, artefacts, stashes, trades and
upgrades; companions chitchatting; faction/zone/weather bulletins; surge casualty reports; task
gossip. `dynamic_news_helper` is its **composition toolkit** (landmarks, time-phrasing, faction/mutant
pickers, localized descriptors, and the actual tip sender). Much of the chatter is **fabricated
ambiance**, not real events — note the `*_Fake` death-report variants and the "fake" casualty reports.

### Singleton lifecycle (verified)

`on_game_start` registers `on_game_load` (caches `TimeFactor`, re-registers the delay callbacks) and
`on_option_change` → `update_settings`. After a load the manager **delays startup ~27.5 s**
(`actor_on_first_update` arms a 7.5 s delay, then `actor_on_update` waits a further 20 s) before
`get_dynamic_news()` lazily constructs the `DynamicNewsManager` and calls `update_settings`; on the
very first run it fires the DRX-questline gossip once (`drx_sl_start_news` info-portion guard). The
delay avoids playing news before the level is fully loaded. `destroy_dynamic_news` tears the singleton
down on a new game.

### Channels & the throttled queue (verified)

The manager keeps a per-faction **`queue[community]`** of pending messages plus a `channel_status` map
(some channels open only when the actor *is* that community — `monolith`/`army`/`greh`/`isg`).
**`PushToChannel(name, t)`** enqueues a message table (`Mg` text / `Se` sender / `Dl` delay / `Ic` icon
/ `Snd` sound / `It` icon-by / `Ty` reaction-type / `Id` sender-id), inserting at the head (LIFO) and
pre-translating any `st_dyn_news_ch_*` channel tokens. It is **gated**: it drops the message if news is
disabled, `NewsToggle` says suppress, the PDA isn't charged, or the in-flight `counter` exceeds
`max_cnt` (3) — so no more than a few messages ever stack.

### The tick scheduler (verified spine; composers surveyed)

`__init` arms six `CreateTimeEvent` ticks on the `"DynamicNewsManager"` group and three callbacks
(`monster_on_death_callback`, `npc_on_death_callback`, `npc_on_get_all_from_corpse`):

- **`TickNews`** (re-arms every 5–20 s) is the **drain**: if `NewsToggle` passes and the PDA is
  charged, it pops one message from a channel queue and sends it via `dynamic_news_helper.send_tip`,
  then (if the message carried a reaction `Ty`) arms `TickQuick` to fire a follow-up reaction.
- **`TickQuick`** emits reactions to the last news (e.g. someone else commenting on a found artefact).
- **`TickSpecial` / `TickTask` / `TickRandom` / `TickCompanion`** are the **generators**, re-armed on
  config-driven cycles (`cycle_of_*_news`, randomized to value+1 … value·2+1), each calling a family of
  composer methods *(surveyed)* — faction reports, zone/weather/time bulletins, bounty/alpha-squad
  gossip, task gossip (`GossipTaskLL/MS/OA/DRX`), and companion chatter (`CompanionAboutLife/Level/Actor`).
- **Event callbacks** turn real world events into reports: `npc_on_death_callback` →
  `DeathByStalker`/`DeathByMutant`/`KillWounded`/`SOS*` (online), `monster_on_death_callback`,
  `npc_on_get_all_from_corpse` → loot gossip. Offline/warfare variants (`SOSBattleOffline`,
  `SOSWarfareCapture`) cover other maps. The composers pick a plausible speaker
  (`FindSpeaker*` family) and build the sentence with the helper's descriptors *(surveyed)*.

### The `NewsToggle` gate (verified — the key runtime rule)

`NewsToggle()` returns **true to suppress** and is checked by every send/enqueue path. It suppresses
when the game is paused (prevents a crash from playing news mid-load), on an **invalid map**
(underground/lab levels from the helper's `invalid_maps`), and **during a surge or psi-storm** — and on
surge start it **erases all queued messages** to stop a flood the moment the blowout ends. It also
drives the **comm-centre transitions**: a one-time "Welcome to the network" on a new game
(`trx_dynamic_news_welcome_to_network` guard), and a "communication established/lost" tip
(`pda_welcome` / `pda_communication_lost` sound) on each surge edge. Dynamic news additionally requires
`item_device.is_pda_charged` — a dead PDA battery silences it (shared with the F110 radio / F102 device
battery).

## The helper toolkit (`dynamic_news_helper`) — verified, READ-IN-FULL

The library the composers lean on:

- **`level_points` + `GetPointDescription(obj)`** — a hand-authored table of named landmark coordinates
  per level; the picker finds the nearest landmark to an object and computes an 8-way compass bearing,
  yielding strings like "near the cement factory to the east" (`st_dyn_*_blizko/daleko_N` +
  `st_dyn_news_<dir>`).
- **`GetTimeString` / `GetTimeScope` / `IsDateCloseBy`** — a six-tier time-phrasing engine that turns an
  hour-delta into "at 8 PM" (tier 6) … "between 3:00 and 5:00" … "in three hours" … "this morning" …
  "today" … "soon" (tier 1), falling back to the widest range if a tier can't render.
- **Pickers over `faction_expansions`** — `PickFaction(only_natural)` (skips factions hostile to the
  actor via `game_relations`, F108), `PickMap`, `PickMutant(tier)`, `GetFaction`, `GetMutant`,
  `PickDirection`.
- **Descriptors** — `GetCommunityDescription`, `GetMonsterDescription` (clsid/kind → localized name),
  `GetWeaponDescription` (classifies the speaker's active weapon), `GetColor`.
- **`send_tip(msg, header, timeout, showtime, sender, sound, icon_by)`** — the richer sender the manager
  actually calls: picks the PDA sound by news type
  (`pda_tips`/`pda_alarm`/`pda_communication_lost`/`pda_beep_*`/`pda_welcome`), optionally delays it via
  a `CreateTimeEvent`, resolves the icon from `news_manager.tips_icons` (or the NPC), **drains the PDA
  battery** (`item_device.drain_device_on_event` on slot 8), and pushes `give_game_news` with a
  `%c[default]` color prefix.
- **`list_actor_squad_by_id`** — the companion-roster helper (shared with `axr_companions`).

`faction_expansions` (werejew/Tronex) is the data tier behind the pickers: `faction` / `level` /
`mutant` profile tables + `mutant_tier_by_clsid`, populated from config.

## Isolation ledger (the seams)

- **Engine news queue (F39):** both tiers terminate at `give_game_news` / `give_talk_message2`
  (`flows/pda-map-news.md`); nothing here draws UI itself.
- **Options (F17):** `update_settings` reads ~22 `alife/dynamic_news/*` toggles + four `cycle_of_*_news`
  timers + `message_duration` via `ui_options.get`, re-armed on `on_option_change`
  (`flows/options-runtime.md`).
- **PDA battery (F102/F110):** `is_pda_charged` gates dynamic news; the helper's `send_tip` drains the
  device — same battery the F110 radio and F102 `item_device` use.
- **Combat/world events (F87/F105/F94/F34):** death/loot callbacks and the offline/warfare SOS reports
  read `sim_offline_combat` (F87) and warfare capture (F105); monster descriptors use clsid tables
  shared with F94/F34.
- **Stats & relations (F107/F108/F40):** `CompanionAboutActor` reads rank/reputation
  (`flows/game-services.md`); `PickFaction(natural)` respects `game_relations` enmity
  (`flows/faction-relations-lua.md`).
- **Surge (engine + Lua):** `xr_conditions.surge_started/complete`, `surge_manager`,
  `psi_storm_manager` drive the comm-centre on/off transitions.
- **Config & localization:** `plugins\news_tips_icons.ltx` (Tier-1 icons), `faction_expansions` data,
  and a large `st_dyn_news_*` / `st_dyn_*` localization surface (every composed sentence is a translated
  key).

## Threading

Main thread. Dynamic News is entirely `CreateTimeEvent`-driven (the six ticks) plus the three event
callbacks; Tier-1 dispatch is synchronous with the caller. Single-threaded with gameplay.

## Failure & edge cases

- **No dynamic news right after a load:** by design — the ~27.5 s startup delay; also nothing fires
  while paused, underground/in a lab, during a surge, or with a dead PDA battery.
- **News goes silent after a blowout:** `NewsToggle` erases the queue on surge start to prevent a
  post-surge flood, and re-opens with the "communication established" tip — not a bug.
- **`shw_stash` is hardcoded `false`:** the angry-stash-owner news is disabled regardless of the
  option.
- **`ResponseOnDeathBySurges` is `[NOT USED]` and `ReportDeathBySurge` is commented out:** surge
  casualty reports are dormant; the live surge-death chatter is the `*_Fake` (fabricated) variants.
- **`dynamic_news_helper.PickRandomKey` is dead and broken:** it iterates `pairs(v)` over an undefined
  global `v` (should be `t`) and would error if called — but it has no callers (the pickers use their
  own loops). Latent, harmless until someone wires it up.
- **`GetTimeString` tier-5 copy-paste bug:** the `else` branch of the right-bound pick reassigns
  `hrs_L` instead of `hrs_R`, so `hrs_R` can stay nil and the "between X and Y" phrasing silently
  drops to the wider tier-4 fallback. Cosmetic.

## Deeper

- Dispatcher: `unpacked/scripts/news_manager.script` (`send_tip`, `send_sound`, `send_task`,
  `relocate_item`/`relocate_money`, `tips_icons`)
- Generator: `unpacked/scripts/dynamic_news_manager.script` (`DynamicNewsManager` — `__init`/`destroy`,
  `TickNews`/`TickQuick`/`Tick{Special,Task,Random,Companion}`, `PushToChannel`, `NewsToggle`, the
  `Death*`/`SOS*`/`Gossip*`/`Companion*`/`Response*` composers)
- Toolkit: `unpacked/scripts/dynamic_news_helper.script` (`GetPointDescription`, `GetTimeString`,
  `Pick*`/`Get*` descriptors, `send_tip`, `list_actor_squad_by_id`, `invalid_maps`)
- Data: `unpacked/scripts/faction_expansions.script` (`faction`/`level`/`mutant` profiles)
- Engine queue: `flows/pda-map-news.md` (F39); options: `flows/options-runtime.md` (F17); PDA battery:
  `flows/item-behaviors-lua.md` (F102); map: `modlayer/gameplay/ui-scripts.md` (M20)

## Maintenance

- Keep the two-tier split clear: `news_manager` = the synchronous dispatcher ~50 scripts call;
  Dynamic News = the self-scheduling generator. They meet only at `give_game_news`.
- Keep the `NewsToggle` rules front (paused / invalid map / surge-erases-queue / PDA charge + the
  comm-centre transitions) — it answers most "why is dynamic news silent / spammy" questions.
- Promote the *(composer surveyed)* `Death*`/`Gossip*`/`SOS*` bodies to verified if a task needs a
  specific report's exact trigger or sentence assembly.
- No `gamedata/` override of any of the three scripts as of this writing — re-check on pickup.
