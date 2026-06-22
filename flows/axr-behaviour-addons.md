---
domain: flows/axr-behaviour-addons
tags: [modlayer, gameplay, unpacked, lua, axr, scheme, goap, ai, combat, keybind, flow]
sources: [unpacked/scripts/axr_beh.script, unpacked/scripts/axr_beh_patches.script, unpacked/scripts/axr_fight_from_cover.script, unpacked/scripts/axr_stalker_panic.script, unpacked/scripts/axr_npc_vs_box.script, unpacked/scripts/axr_npc_vs_heli.script, unpacked/scripts/axr_radio_in_heli.script, unpacked/scripts/axr_turn_on_campfire.script, unpacked/scripts/axr_keybind.script]
related: [INDEX.md, modlayer/gameplay/axr-addon.md, flows/script-npc-actions.md, flows/scheme-engine-core.md, flows/ai-squad-logic.md, flows/gameplay-boot.md, flows/helicopter-ai.md, flows/ui-news-tips.md, flows/relations-info.md, flows/axr-managers.md, flows/companion-system.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-13
verified: code
---

# Flow: AXR behaviour add-ons

The `axr_*` behaviour layer — Alundaio's pack of **scripted GOAP schemes** that the modded exe adds on
top of the stock A-Life action set, plus one keybind utility. Eight of these are NPC behaviour schemes
registered through the standard scheme interface (`flows/script-npc-actions.md`, F37): a companion
escort/patrol brain (`axr_beh`) and a family of auto-active combat/utility reactions (fight-from-cover,
panic, hide-from-heli, radio-for-help, attack-boxes, light-campfires). `axr_keybind` is the scriptable
key dispatcher. The bigger `axr_*` **managers** (task/trade/companions) are F114; the `axr_main`
orchestrator that loads all of this is F65 (`flows/gameplay-boot.md`). This domain is the first half of
the M21 axr-addon map.

**Provenance:** code-verified — `axr_beh` (794 lines), `axr_beh_patches` (126) and `axr_keybind` (83)
were **READ-IN-FULL**; the six combat/utility schemes (`axr_fight_from_cover`, `axr_stalker_panic`,
`axr_npc_vs_box`, `axr_npc_vs_heli`, `axr_radio_in_heli`, `axr_turn_on_campfire`) were
**structure-surveyed** — headers, the scheme-registration interface (`setup_generic_scheme` /
`add_to_binder` / `configure_actions` / `disable_generic_scheme` / `npc_add_precondition`), the
evaluator/action class shape, and each unique `evaid`/`actid` were read, but the per-action `execute`
bodies were not line-read (flagged *(scheme surveyed)*). The `gamedata/scripts/axr_beh_patches.script`
override was **diffed byte-for-byte against `unpacked/`** — identical (a redundant packaged copy).

## The shared shape: every add-on is a GOAP scheme (verified pattern)

With one exception (`axr_keybind`), each file follows the Anomaly scheme contract that F37 defines: a
fixed `evaid`/`actid` pair (here the block 188111–188120), an `evaluator_*` (a `property_evaluator`
whose `evaluate()` decides whether the scheme wants control) and an `action_*` (an `action_base` whose
`execute()` drives the NPC), plus the registration quartet `setup_generic_scheme` / `add_to_binder` /
`configure_actions` / `disable_generic_scheme` and a `npc_add_precondition`. They are wired into the
NPC's `motivation_action_manager` at bind time and registered as named schemes at boot via
`_G.LoadScheme` (`flows/gameplay-boot.md`, F65 → `flows/scheme-engine-core.md`, F66). Two `scheme_type`s
appear:

- **`active_section`** — `axr_beh` only: an explicit logic scheme an NPC runs because its logic LTX
  names a `[beh@…]` section.
- **`generic`** — all six combat/utility schemes: auto-active reactions layered onto every eligible NPC
  through `configure_actions`, gated by their evaluator (e.g. "is a hostile heli near?").

The five combat schemes share an almost identical movement skeleton — local `validate(npc,vid)` /
`lmove` / `lmove_ns` cover-vertex helpers and `try_go_cover` / `try_to_strafe` / `try_go_backward`
action methods — they differ mainly in *what* they evaluate against *(scheme surveyed)*.

## `axr_beh` — the companion / escort brain (verified, READ-IN-FULL)

The centerpiece: Alundaio's `beh` scheme (`evaid`/`actid` 188111), an `active_section` scheme that makes
an NPC follow, escort, patrol or take cover. It is the AI that `axr_companions` (F114) drives, and the
source of the `npcx_beh_*` info-portion flags that the PDA relations/companion roster reads (F110).

- **Config defaults (`init_custom_data`).** Everything is a condlist keyed on `npcx_beh_*` info flags:
  `behavior_state` (`beh_wait` / `beh_move` / `beh_patrol`), `target` (`commander` for non-commanders,
  else `actor`), walk/jog distances, `keep_dist` (near/far), per-state animations
  (wait/walk/jog/run/delay), and a stealth substate that swaps the animation set
  (`hide`/`sneak`/`rush`). All are overridable from the `[beh@section]` logic block.
- **`action_beh:execute`.** Each tick it re-picks the behaviour and animations from their condlists,
  resolves the desired target via `set_desired_target`, then dispatches to the matching handler.
- **`set_desired_target`** is the routing core — it understands a rich target vocabulary: story object
  names, `enemy`, `cover_near_actor`, `cover_wait`, `task_target_anomaly`, `task_target_hostage`,
  `waypoint`, `commander` (the squad commander's position), `scan_area`, and literal `lvid:<n>` /
  `id:<n>` forms, caching the result with a `__keep_until` TTL and reserving the level vertex in
  `db.used_level_vertex_ids`.
- **Behaviour handlers.** `beh_wait` (stand and look), `beh_cover` (rush to a cover vertex then idle in
  a wait/hide/random state), `beh_move` (close to within `desired_distance` of the target, choosing an
  offset "assist point" via the `position_node` helper so squadmates don't stack), and `beh_path` (walk
  a waypoint list with per-point look/animpoint/sound/signal directives, delays, and `path_end`
  loop/reverse/switch logic).
- **`add_to_binder`** installs the evaluator + action with the standard combat-exclusion preconditions
  (alive, no enemy, no danger, not in meet/abuse) and makes the stock `alife` and `state_mgr` planner
  actions yield to it when `beh` is active.
- The camp/animpoint integration (`use_camp`, `fill_approved_actions`) is present but the camp branch is
  **commented out** (the file's own `TODO: Fix use camp`) — `axr_beh` does not currently drive camp
  director/listener animations.

## `axr_beh_patches` — the customization hook (verified, READ-IN-FULL)

GhenTuong's monkeypatch extension (guarded by `_G._AXR_BEH_EX_LOADED` against double-loading). At
`on_game_start` it wraps `axr_beh.set_scheme`, `action_beh.execute` and `action_beh.finalize` so a
`[beh@section]` can declare three hooks — `init` / `main` / `finalize` as `file.function` strings — that
run alongside (or, by setting `target = nil` and `behavior_state = nil`, *instead of*) the stock beh
logic. This is how third-party addons graft custom per-NPC behaviour onto the beh scheme without forking
it. The `gamedata/` copy is byte-identical, so it carries no runtime delta.

## The combat / utility schemes (structure-surveyed)

All Alundaio `generic` schemes, auto-applied and gated by their evaluators:

- **`axr_fight_from_cover`** (188118) — forces an NPC to fight cautiously, breaking to cover between
  shots; honors an optional `home_point` to anchor the cover search.
- **`axr_stalker_panic`** (188115) — makes an NPC flee/panic from things that `can_cause_panic` (the
  evaluator's `find_valid_target` picks the threat); flees via cover/strafe/backward moves.
- **`axr_npc_vs_heli`** (188117) — stalkers take cover from a hostile helicopter (`best_heli` selects
  the threat); exposes `is_under_npc_vs_heli(npc)` for other code to test. Pairs with the heli AI (F95).
- **`axr_radio_in_heli`** (188116) — the radio-for-assistance reaction (the Lua side of the "requesting
  chopper assistance" chatter surfaced by Dynamic News `RadioInHeli`, F112).
- **`axr_npc_vs_box`** (188120) — an NPC shoots/destroys breakable wooden boxes (the `is_box` section
  whitelist) in its way.
- **`axr_turn_on_campfire`** (188114) — stalkers holding a gulag job at a smart terrain walk to and
  **light that smart's campfires** (`light_campfire`, evaluator `find_valid_target` over nearby
  campfires); job-driven, ties into the smart-terrain job board (F72/F73) and `is_under_turn_on_campfire`.

Each registers/unregisters cleanly through the F37 generic interface; the exact flee/cover math lives in
their `execute` bodies *(not line-read)*.

## `axr_keybind` — the scriptable key dispatcher (verified, READ-IN-FULL)

Not a scheme — a small utility (ported from CoC 1.5). `bind(dik_or_bind, functor, alias)` registers a
functor against either a raw `DIK_keys` code or a named engine bind (`kCUSTOM1`–`kCUSTOM15` from
`ui_keybinding.xml`); `unbind` removes it. At `on_game_start` it subscribes to
`on_key_press`/`release`/`hold` (F65 callback bus) and, for any alias-bound action not already a stock
`key_bindings` entry, persists a default into `axr_main.config [global_keybinds]` (so the bind is
rebindable via `axr_options.ltx`). `try_key_action(key, typ)` dispatches the functor on each key event
with `typ` = 1 press / 0 release / 2 hold. This is the mechanism feature scripts use for custom hotkeys.

## Isolation ledger (the seams)

- **Scheme registration (F37/F66):** every behaviour add-on plugs into `motivation_action_manager` via
  the generic/active_section interface and is registered at boot by `_G.LoadScheme`
  (`flows/script-npc-actions.md`, `flows/scheme-engine-core.md`).
- **Boot (F65):** `axr_main` loads the schemes and the callback bus; `axr_keybind` and
  `axr_turn_on_campfire` hook `on_game_start` (`flows/gameplay-boot.md`).
- **Companions (F114):** `axr_beh` is the behaviour `axr_companions` assigns; the `npcx_beh_*` info
  flags it reads are set by the companion manager and shown in the PDA roster (F110).
- **Squad/jobs (F12/F72/F73):** `target = commander` reads the squad commander (`get_object_squad`);
  `axr_turn_on_campfire` is gulag-job-driven.
- **Helicopters (F95) & news (F112):** `axr_npc_vs_heli` reacts to the heli AI; `axr_radio_in_heli` is
  the Lua counterpart of the Dynamic-News chopper-assistance report.
- **Config:** `[beh@section]` logic blocks (waypoints `ptN`, `target`, `behavior_state`, the
  `init`/`main`/`finalize` hooks), per-scheme logic sections, `axr_main.config [global_keybinds]`,
  `ui_keybinding.xml` custom binds, the `is_box` whitelist.
- **State machine:** all schemes drive movement/animation through `state_mgr.set_state` and the
  level-vertex reservation table `db.used_level_vertex_ids`.

## Threading

Main thread. Schemes tick inside the NPC's GOAP planner on the AI scheduler; `axr_keybind` is
event-driven on the key callbacks. Single-threaded with gameplay.

## Failure & edge cases

- **`axr_beh` camp animations are disabled:** the `use_camp` director/listener branch is commented out
  (`TODO: Fix use camp`) — beh NPCs won't play camp animpoint sets even with `use_camp` set.
- **`axr_beh_patches` requires valid hook strings:** an `init`/`main`/`finalize` naming a missing
  `file.function` is logged (`function invalid`) and skipped, not fatal.
- **The gamedata `axr_beh_patches` copy is identical:** no override behavior — don't treat it as a
  runtime delta.
- **Generic combat schemes are preconditioned to non-companion combat:** they yield to higher-priority
  planner actions; if an NPC "won't fight from cover / won't panic", the gate is usually its evaluator
  (no valid threat) or a competing scheme, not a bug.
- **`axr_keybind` aliases live in `axr_main.config`:** a stale `[global_keybinds]` entry can shadow a
  new default; clear it there to reset.

## Deeper

- Companion brain: `unpacked/scripts/axr_beh.script` (`action_beh:execute`/`set_desired_target`/
  `beh_move`/`beh_path`/`beh_cover`, `add_to_binder`, `position_node`)
- Hook: `unpacked/scripts/axr_beh_patches.script` (`GHENTUONG_scheme_init`, the `set_scheme`/`execute`/
  `finalize` wrappers)
- Combat/utility schemes: `unpacked/scripts/axr_fight_from_cover.script`, `axr_stalker_panic.script`,
  `axr_npc_vs_heli.script`, `axr_radio_in_heli.script`, `axr_npc_vs_box.script`,
  `axr_turn_on_campfire.script`
- Keybinds: `unpacked/scripts/axr_keybind.script` (`bind`/`unbind`/`try_key_action`)
- Scheme interface: `flows/script-npc-actions.md` (F37), `flows/scheme-engine-core.md` (F66); boot:
  `flows/gameplay-boot.md` (F65); map: `modlayer/gameplay/axr-addon.md` (M21)

## Maintenance

- Keep the "every add-on is a GOAP scheme on the F37 interface (except axr_keybind)" framing — it routes
  any axr behaviour question straight to the scheme engine.
- Promote the *(scheme surveyed)* combat `execute` bodies to verified if a task needs a specific
  flee/cover algorithm.
- `axr_beh` is the F114 companion dependency — keep its target vocabulary and `npcx_beh_*` flag list
  current when F114 (axr_companions) lands.
- Re-diff `gamedata/scripts/axr_beh_patches.script` and re-check for new `gamedata/scripts/axr_*` on
  pickup (the modded-exe addon layer is where overrides accumulate).
