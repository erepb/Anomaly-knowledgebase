---
domain: flows/gameplay-boot
tags: [modlayer, gameplay, unpacked, lua, boot, callbacks, class-registration, flow]
sources: [unpacked/scripts/_g.script, unpacked/scripts/axr_main.script, gamedata/scripts/axr_main.script, unpacked/scripts/class_registrator.script, gamedata/scripts/class_registrator_modded_exes.script, unpacked/scripts/game_registrator.script, unpacked/scripts/game_setup.script]
related: [flows/lua-callback-lifecycle.md, flows/object-lifecycle.md, flows/db-registry.md, flows/scheme-engine-core.md, modlayer/gameplay/_overview.md, modlayer/scripting.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: gameplay boot & load order (F65)

How the Lua gameplay layer comes up: the engine calls `_g.script:start_game_callback`, which kicks
`axr_main.on_game_start` (the module orchestrator) and then directly seeds the core managers. Separately,
at object-factory init the engine calls `class_registrator` to bind every C++ class to its Lua server/
client classes. This is the entry that everything else in the gameplay corpus hangs off. (Deeper
callback-bus detail is F1 `flows/lua-callback-lifecycle.md`; this flow is the boot spine + load order.)

**Provenance:** `_g.script` (`start_game_callback`, the `*ScriptCallback` API, `LoadScheme`) read and
verified. `class_registrator`/`game_registrator` structure and `axr_main.on_game_start` behavior are
*(agent-traced — not personally re-read)* except where noted; the `axr_main` callback functions
(`callback_set/unset/make_callback`) are confirmed by their callers in `_g.script` but their bodies are
agent-traced. **`gamedata/scripts/axr_main.script` overrides** the base `axr_main` (the override is what
runs).

## Sequence (verified spine)

### Engine → Lua game start — `_g.script:start_game_callback`
Verified body, in order:
1. `math.randomseed(device():time_global())` + version printf.
2. `if axr_main then axr_main.on_game_start() end` — run the module orchestrator (see below).
3. `sim_board.get_sim_board()` — create/fetch the simulation board (squad/smart registry; roadmap F86).
4. `task_manager.clear_task_manager()` — reset the task system (`flows/tasks.md`, roadmap F97).
5. `dialog_manager.fill_phrase_table()` — build the dialog phrase tables (`flows/dialog-trade.md`, F99).
6. `sr_light.clean_up()` — reset dynamic-light restrictors.
7. `pda.add_quick_slot_items_on_game_start()` — PDA quick-slot init (`flows/pda-map-news.md`).

So `start_game_callback` is both the hand-off to `axr_main` *and* the explicit boot of the core
managers; it is the canonical "game session start" hook.

### Module orchestrator — `axr_main.on_game_start` *(agent-traced; gamedata override)*
Scans `$game_scripts$` for every `*.script`, skips an ignore list (the registrars, factory glue, etc.),
and calls `on_game_start()` on each module that defines one — the open-ended plugin init point. The
`gamedata/` override adds ordered dispatch (priority via sorted iteration) on top of the base.

### Scripted-callback API — `_g.script` (verified)
The bus every subsystem uses to hook engine events, all thin wrappers over `axr_main`:
- `RegisterScriptCallback(name, fn)` → `axr_main.callback_set(name, fn)`
- `UnregisterScriptCallback(name, fn)` → `axr_main.callback_unset(name, fn)`
- `SendScriptCallback(name, ...)` → `axr_main.make_callback(name, ...)`, then if `axr_main[name]`
  exists it is also called (so `axr_main` can define a handler by naming convention).
- `AddScriptCallback(name)` → `axr_main.callback_add(name)`.
Full bus semantics: `flows/lua-callback-lifecycle.md`.

### Scheme registry hook — `_g.script:LoadScheme` (verified)
`LoadScheme(filename, scheme, ...stypes)` registers a behavior scheme: sets `schemes[scheme] = filename`
and, for each stype passed, `schemes_by_stype[stype][scheme] = true`. `schemes`/`schemes_by_stype` are
the globals `xr_logic`/`modules` use to find and enable schemes (`flows/scheme-engine-core.md`). This is
how e.g. `xr_walker` becomes a usable scheme name.

### Class registration — `class_registrator` *(agent-traced)*
At object-factory init the engine calls `class_registrator.register(object_factory)`, which calls
`object_factory:register(client, server, clsid, script_clsid)` for every entity type (actor → `se_actor`,
stalker → `se_stalker`, monsters → `se_monster`, zones → `se_zones`, items/weapons → `se_item` variants,
…). `gamedata/scripts/class_registrator_modded_exes.register` supplements it with the modded-exe classes
(SSRS weapon, stat-mgun, projector). `game_registrator.get_game_clsid` maps the game mode to its clsid
(`register` is a stub). This is the Lua↔engine factory binding that makes `db`'s `clsid.script_stalker`
checks meaningful.

## Function digests (spine)

- **`_g.script:start_game_callback`** — *Does:* the verified 7-step session boot above. *Calls:*
  `axr_main.on_game_start`, `sim_board.get_sim_board`, `task_manager.clear_task_manager`,
  `dialog_manager.fill_phrase_table`, `sr_light.clean_up`, `pda.add_quick_slot_items_on_game_start`.
  *Fired by:* the engine at game start. *(verified.)*
- **`_g.script:RegisterScriptCallback`/`SendScriptCallback`/`UnregisterScriptCallback`/`AddScriptCallback`**
  — *Does:* the callback-bus facade over `axr_main`. *(verified; `axr_main` bodies agent-traced.)*
- **`_g.script:LoadScheme`** — *Does:* register a scheme name→file and its stypes into `schemes`/
  `schemes_by_stype`. *Writes:* those globals. *(verified.)*
- **`axr_main.on_game_start`** — *Does:* discover + dispatch every module's `on_game_start`. *(agent-traced;
  gamedata override.)*
- **`class_registrator.register`** — *Does:* bind C++ clsids to Lua server/client classes via
  `object_factory:register`. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** `schemes`/`schemes_by_stype` (in `_g`), `axr_main`'s `intercepts` callback registry,
  the new-game flags in `game_setup`.
- **Engine seam (in):** the engine fires `start_game_callback` and `class_registrator.register`; both are
  Lua functions the C++ side looks up by name.
- **Out:** `start_game_callback` boots the core managers (sim_board/tasks/dialogs/pda/sr_light); the
  callback bus fans engine events to all subscribers.
- **Override seam:** `axr_main` and `class_registrator_modded_exes` live in `gamedata/` — the modded-exe
  layer customizes boot here (`METHODOLOGY.md` dual-cite).

## Threading / failure / edge cases

- **Load order matters:** schemes must be `LoadScheme`-registered (via their module's load) before an NPC
  activates them; `class_registrator` must run before any Lua entity is created.
- **`axr_main` is required:** `start_game_callback` guards `if axr_main`, but the rest of the corpus
  assumes the callback bus exists.
- **New-game vs load:** `game_setup.on_game_start` only does first-time world setup (guarded by a saved
  flag); don't assume its item-spawn work runs on every load.

## Deeper

- Boot + callback API + scheme registry: `unpacked/scripts/_g.script`
  (`start_game_callback`, `RegisterScriptCallback`, `LoadScheme`)
- Orchestrator: `gamedata/scripts/axr_main.script` (override) / `unpacked/scripts/axr_main.script`
  (`on_game_start`, `callback_set`/`callback_unset`/`make_callback`)
- Class binding: `unpacked/scripts/class_registrator.script`,
  `gamedata/scripts/class_registrator_modded_exes.script`, `unpacked/scripts/game_registrator.script`
- New-game setup: `unpacked/scripts/game_setup.script`
- Bus internals: `flows/lua-callback-lifecycle.md`; scheme engine: `flows/scheme-engine-core.md`

## Maintenance

- `start_game_callback`'s body is the authoritative boot order — re-read it if managers are added/removed.
- Promote the agent-traced `axr_main`/`class_registrator` parts to verified when their bodies are read
  (good companions: F65 deepening or the F1 bus flow).
- Always check the `gamedata/` overrides of `axr_main`/`class_registrator_modded_exes` — they are live.
