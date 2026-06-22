---
domain: flows/monster-schemes
tags: [modlayer, gameplay, unpacked, lua, schemes, ai, flow]
sources: [unpacked/scripts/mob_walker.script, unpacked/scripts/mob_home.script, unpacked/scripts/mob_remark.script, unpacked/scripts/mob_jump.script, unpacked/scripts/mob_camp.script, unpacked/scripts/mob_combat.script, unpacked/scripts/mob_death.script, unpacked/scripts/mob_trade.script, unpacked/scripts/mob_trader.script, unpacked/scripts/mob_state_mgr.script, unpacked/scripts/mob_sound.script, gamedata/scripts/bind_monster.script, unpacked/scripts/bind_trader.script]
related: [flows/scheme-engine-core.md, flows/logic-ltx-switching.md, flows/binder-architecture.md, flows/script-npc-actions.md, flows/monster-ai-control.md, flows/special-actors.md, flows/scheme-walker-patrol.md, flows/smart-terrain-core.md, flows/db-registry.md, flows/lua-callback-lifecycle.md, flows/creatures-actor-configs.md, modlayer/gameplay/monster-schemes.md, modlayer/gameplay/binders.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: monster schemes (`mob_*`) (F94)

The **`mob_*` schemes** — logic schemes for monsters (and the scripted trader): patrol, home leash,
scripted performances, jumps, plus the generic combat/death/trade listeners. Control model differs from
stalker `xr_*` schemes: monsters have no GOAP/state_mgr — a `mob_*` scheme either **captures** the monster
and drives the **script-entity action queue** (F37: `xr_logic.mob_capture` + `action(move/anim/look/
sound, cond)`) or leaves the engine monster FSM (F34) running inside constraints (`set_home`).

**Provenance:** **verified, no agent** — all 11 `mob_*` files READ-IN-FULL this session. Run by
**`gamedata/bind_monster.script` (live override, F88)** for `stype_mobile` and `bind_trader.script` for
`stype_trader` (both checked).

## Registration & usage (verified in `modules.script` + config grep)

`LoadScheme(file, scheme, stype)`: **stype_mobile** = `mob_remark`/`mob_walker`/`mob_combat`/`mob_death`/
`mob_jump`/`mob_home`; **stype_trader** = `mob_trader`/`mob_trade`. **`mob_camp` is NOT registered** —
a complete home-camp FSM (camp/alife-release/move-home states, `db.camp_storage` point claiming,
`get_monster_hit_info` retaliation) that is dead code in the base corpus, like `ph_appforce` (F93).
`mob_state_mgr` and `mob_sound` are helper modules, not schemes. Base-config usage: **`mob_home` 230**,
`mob_walker` 99, `mob_trader` 3, `mob_jump` 1, others 0.

## How callbacks arrive (verified in both binders)

`bind_monster` (gamedata) registers `patrol_path_in_point` → routes `waypoint_callback` to the active
scheme, `hit` → `st.hit` `hit_callback`, `death` → **explicitly issues `death_callback` to
`st.mob_death`** then to the active scheme, `sound` → hear; `use_object` per meet/companion state. No
combat callback is registered (see `mob_combat` below). `bind_trader` runs `stype_trader` via
`smart_terrain.setup_gulag_and_logic_on_spawn(…, modules.stype_trader, …)` + `issue_event("update")`.

## The schemes (all read in full)

### `mob_walker` — monster patrol
`reset_scheme`: `mob_capture` + `action(move(walk_fwd, patrol(path_walk, next, continue)), cond(move_end))`.
The engine fires `patrol_path_in_point` → `waypoint_callback`, which applies the waypoint flags parsed by
`utils_obj.path_parse_waypoints` — `r` run / `c` crouch (`move.steal`) / `s` sound / `sig` signal (into the
active scheme's `signals`) / `b` per-point behavior state / `t` wait time / `a` idle-anim condlist — and
picks a look point via `move_mgr.choose_look_point` (the same waypoint engine as the stalker walker, F74).
Alternates `state_moving`/`state_standing` (`anim` + `look` actions with `time_end` cond); **re-runs
`reset_scheme` if something released the capture** (`mob_captured` check each update). `path_walk ==
path_look` aborts at `set_scheme`.

### `mob_home` — territory leash (the workhorse, 230 refs)
No capture at all: `reset_scheme` calls the engine `set_home(vertex, min_r, max_r, aggressive, mid_r)` —
either the **gulag smart's vertex** (`gulag_point=true`, resolved via `alife_object(id).m_smart_terrain_id`,
the F72 job pattern) or a `path_home` patrol point; radii are sanity-bumped (`mid>min`, `max>mid`).
The engine FSM (F34) then roams/fights freely inside the leash. `deactivate` → `remove_home`.

### `mob_remark` — scripted performance
`mob_capture` + a queued sequence of `action(anim(…)[, sound(…, "bip01_head")], cond(anim_end|sound_end|
time_end))` from parallel `anim`/`snd`/`time` config lists; `dialog_cond` condlist toggles
`enable_talk/disable_talk` per update; `tip` → `news_manager.send_tip`; signals `action_end` when the
queue drains. **Latent break:** its sound path calls `mob_sound.pick_sound_from_set(self.object, sndset,
{})` but that function takes `(set)` — the object lands in the set slot → abort for any `snd=` config;
the anim-only path is the live one.

### `mob_jump` — scripted jump
Three states: capture → `action(look(look.point, jump-point), cond(look_end))` → engine `jump(point,
ph_jump_factor)` → signal `"jumped"` + `mob_release`. Jump point = `path_jump` patrol point 0 + `offset`
vector. Aborts at `set_scheme` if no `on_signal` line (the scheme is useless without an exit).

### `mob_camp` *(UNREGISTERED — dead code)*
FSM: STATE_CAMP (look at `path_look` point, idle anim) / STATE_ALIFE (`mob_release` to engine AI when an
enemy is inside the radius bands, with hysteresis on `home_min/max_radius`) / STATE_MOVE_HOME (run back).
Claims look/home points in the shared `db.camp_storage[path]` table (aborts if more campers than points);
returns to ALIFE on being hit near home (`get_monster_hit_info`); `skip_transfer_enemy` option. Would
need a `LoadScheme` row to be usable.

### `mob_combat` — generic combat listener *(dormant)*
`setup_generic_scheme` binds on an `on_combat` field in stype_mobile logic (18 base-config uses) and
`combat_callback` would `try_switch` while an enemy exists — but **no caller found**: neither binder
registers a combat callback and there is no `callback.combat` in the engine luabind enum (grepped). The
storage gets set up; the event never fires. Monster combat reaction in practice = the engine FSM +
switch conditions.

### `mob_death` — generic death listener
`bind_monster:death_callback` issues straight to `st.mob_death`: records
`db.storage[victim].death.killer`/`killer_name` (the monster analog of the stalker death record, F78 —
read by `xr_conditions` killer checks) then `try_switch`. **Latent bug:** the unknown-killer branch
references the block-local `death` out of scope (nil index error if `who == nil`).

### `mob_trader` — the scripted trader's performance (stype_trader)
The scheme behind the Sidorovich-style trader (`CAI_Trader`, F36; clsid `script_trader` →
`se_stalker.se_trader`). Registers the engine trader callbacks `trader_global_anim_request`/
`trader_head_anim_request`/`trader_sound_end` and answers them with random picks from the hardcoded
`global_animations`/`head_animations` tables (desk/chair/diagonal/forester sets) via
`set_trader_global_anim`/`set_trader_head_anim`; `sound_phrase` plays via
`mob_sound.pick_sound_from_set(set)` (the correct 1-arg call) + `set_trader_sound`; `can_talk` gates
dialog; `play_once` inferred from `on_signal = animation_end`; registers `db.trader`.

### `mob_trade` — generic trade listener (stype_trader)
Binds on `on_trade`; engine `callback.trade_perform_operation` → `on_trade(buy_coast, sell_coast)`
accumulates `st.all_money`, sets `exchanged`, `try_switch`. Module-level `ActorMenu_on_trade_started/
closed` script callbacks (F1 bus, registered `on_game_start`) set `trading` on the talking
`script_trader`'s `mob_trade` storage + `try_switch` — so logic can switch sections on trade
open/close/deal. These storage fields are the condition surface for story logic.

### Helpers
- **`mob_state_mgr`** — `state =` config field applier: only the bloodsucker `invis`/`vis`
  (`set_invisible`) is implemented; any other non-empty state **aborts**. Called by walker/home/remark/camp.
- **`mob_sound`** — a legacy SoC trader voice bank (`trader_*`/`red_forester_*` sets) +
  `pick_sound_from_set(set)` reservoir-random pick. Live consumer: `mob_trader`.

## Isolation ledger (the seams)

- **Scheme engine (F66/F68):** standard template; generics via `setup_generic_scheme`
  (`enable_generic_schemes`, same mechanism as F93's `ph_on_hit`).
- **Binders (F88):** `gamedata/bind_monster.script` (**live override**) runs stype_mobile + routes
  waypoint/hit/death/sound callbacks; `bind_trader.script` runs stype_trader.
- **Script-entity queue (F37):** `mob_capture`/`mob_release` + `action(move/anim/look/sound, cond)` —
  the actual motion driver for walker/remark/jump/camp.
- **Engine monster FSM (F34):** `set_home`/`remove_home`, `get_enemy`, `jump`, `set_invisible`,
  `skip_transfer_enemy`, `get_monster_hit_info` — `mob_home` constrains it instead of capturing.
- **Trader engine (F36):** `CAI_Trader` anim/sound callbacks + `set_trader_*`; `db.trader` (F69).
- **Waypoint engine (F74):** `utils_obj.path_parse_waypoints` + `move_mgr.choose_look_point` shared
  with the stalker walker.
- **Smart terrain (F72/F73):** `gulag_point` home at the job smart's vertex; gulag-prefixed path names
  (`utils_data.prefix_r_string`).
- **Callback bus (F1):** `ActorMenu_on_trade_started/closed` for `mob_trade`.

## Threading / failure / edge cases

- **`mob_camp` is unregistered** and `mob_combat` never fires — don't build logic on `on_combat` for
  monsters; use switch conditions / `mob_home` aggressive instead.
- **`mob_remark` + `snd` aborts** (arg-mismatched `mob_sound` call); use anim-only remarks.
- **`mob_death` with an unknown killer nil-crashes** (out-of-scope `death` in the else branch).
- **`mob_state_mgr` aborts on any state except bloodsucker `invis`/`vis`** (or empty) — a stray
  `state =` in monster logic is a hard error, not a no-op.
- **Capture tug-of-war:** walker re-captures every update if released; anything else releasing the
  monster (combat, other schemes) restarts the patrol from `reset_scheme`.
- **`mob_home` radii are auto-fixed, `mob_camp`'s abort** (`min > max`) and `mob_walker`'s
  (`path_walk == path_look`) are hard.
- **No `gamedata/` override of any `mob_*` file** at writing (the override lives in `bind_monster`).

## Deeper

- Patrol/leash: `unpacked/scripts/mob_walker.script` (`waypoint_callback`), `mob_home.script`
  (`set_home` gulag fork)
- Performances: `mob_remark.script`, `mob_jump.script`, `mob_camp.script` (dead), `mob_state_mgr.script`
- Generics: `mob_combat.script` (dormant), `mob_death.script` (killer record)
- Trader: `mob_trader.script` (`global_animations`/callbacks), `mob_trade.script`
  (`trade_perform_operation`), `mob_sound.script` (voice bank); engine side `flows/special-actors.md` (F36)
- Binders: `gamedata/scripts/bind_monster.script`, `unpacked/scripts/bind_trader.script` →
  `flows/binder-architecture.md` (F88)
- Engine FSM: `flows/monster-ai-control.md` (F34); action queue: `flows/script-npc-actions.md` (F37)
- Map: `modlayer/gameplay/monster-schemes.md` (M9)

## Maintenance

- All bodies read in full — stable. Re-check `gamedata/scripts/mob_*` for overrides per session (none
  at writing; `bind_monster` is the live override that drives these).
- If a mod registers `mob_camp`, wires a combat callback, or fixes the `mob_remark`/`mob_death` latent
  bugs, update the dead/dormant notes.
