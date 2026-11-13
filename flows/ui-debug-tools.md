---
domain: flows/ui-debug-tools
tags: [modlayer, gameplay, unpacked, gamedata, lua, ui, debug, hud, dev-tools, flow]
sources: [src/xrEngine/renderdoc_integration.cpp, src/xrEngine/xr_ioc_cmd.cpp, src/3rd party/luajit-2/src/lj_profile.c, src/3rd party/luajit-2/src/lj_allocprof.c, gamedata/scripts/imgui_shader_bus.script, unpacked/scripts/ui_debug_launcher.script, unpacked/scripts/ui_debug_main.script, unpacked/scripts/xrs_debug_tools.script, unpacked/scripts/debug_cmd_list.script, unpacked/scripts/ui_debug_item.script, unpacked/scripts/ui_debug_weather.script, unpacked/scripts/ui_debug_lighting.script, unpacked/scripts/ui_debug_wpn_hud.script, unpacked/scripts/_g.script, gamedata/scripts/zz_ui_debug_inputs.script, gamedata/scripts/imgui_lua_debug.script, gamedata/scripts/options_modded_exes_debug.script]
related: [INDEX.md, modlayer/gameplay/ui-scripts.md, flows/gameplay-ui-screens.md, flows/pda-tabs.md, flows/hud-ui-load.md, flows/gameplay-boot.md, flows/ai-squad-logic.md, flows/options-runtime.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-11-13
verified: code
---

# Flow: UI debug & dev tools

The developer tooling layer: an in-game **launcher** that hosts spawners, target actions and editors;
a frame-driven **HUD inspector** that overlays live AI/world state; a named-**command registry**; and
a small **gamedata** delta (Lua remote-debugger menu, input-history recall, a debug options page).
Everything here is gated behind the global `DEV_DEBUG` flag — in a normal (non-dev) build none of it
registers. This is the gameplay UI tier (cluster of `modlayer/gameplay/ui-scripts.md`, M20); the
engine widget/XML machinery it stands on is F19 (`flows/hud-ui-load.md`).

**Provenance:** spine personally code-verified — the `DEV_DEBUG` gate (`_g.script`), the launcher's
`main_menu_on_keyboard` open path + `on_game_start` registration + `inject` API + editor-launch
functions (`ui_debug_launcher`), the `xrs_debug_tools` `main_loop` render loop + HUD-slot toggle keys +
`activate_feature` + `on_game_start` gate, the `debug_cmd_list` command-registry API, the old
`ui_debug_main` `debug_ui` function surface and its commented-out opener, and all three gamedata files
(`zz_ui_debug_inputs`, `imgui_lua_debug`, `options_modded_exes_debug`) were read directly. The editor
interiors (`ui_debug_item` / `ui_debug_weather` / `ui_debug_lighting` / `ui_debug_wpn_hud`) were
**structure-surveyed** (function lists, entry points, the shared `ui_debug_launcher.xml` parse) but
their bodies were not line-read — flagged inline as *(structure-surveyed)*.

## The master gate: `DEV_DEBUG`

`DEV_DEBUG` is set `true` in `_g.script` (with `DEV_DEBUG_DEV` for an even deeper tier). It is the
on/off switch for the whole subsystem: `ui_debug_launcher.on_game_start` and
`xrs_debug_tools.on_game_start` both **return early / skip all callback registration unless
`DEV_DEBUG`**, so the launcher key and HUD toggles are inert in a shipping build. The same flag also
gates `_g`'s on-screen error reporter (`ui_options.get("other/debug_error")`) and the `printf` debug
log. Shipping a build with the tools off is a one-line change.

## Three generations of debug menu

The tree has accreted three layers; only the launcher and the HUD inspector are live.

- **`ui_debug_launcher` — the live launcher** (Tronex, "Anomalous Debugger", 2019). Opened from the
  main menu or in-game by the bound debug key. Hosts target actions (get-info / kill / delete / trade /
  set goodwill-relation between two cursor targets), an inventory-cell **item spawner**, an **object /
  squad spawner**, a **Lua/console executer**, a **faction switcher**, and launches the editors. Other
  feature scripts append their own entries via `inject` (verified callers: `bind_anomaly_zone` "Spawn
  artefacts", `txr_routes` "Unlock all routes").
- **`ui_debug_main` — the legacy menu** (the `debug_ui` class: tabbed console, teleport, spawn
  section/squad, set/revert logic, find-nearest, level jump, item editors). It is **dormant**: its
  historical opener — the `keybind_debug_ui` ("S") branch inside `xrs_debug_tools` — is **commented
  out**, superseded by the launcher. The class still loads but nothing routes to it in the stock tree.
- **`xrs_debug_tools` — the HUD inspector** (Alundaio, 2012). Not a dialog; a frame loop that paints
  live state onto three HUD overlays. Covered in its own section below.

### Launcher open path & registration (verified)

`ui_debug_launcher.on_game_start` (only if `DEV_DEBUG`) registers `main_menu_on_keyboard`,
`on_key_release` (in-game injected actions), and `on_game_load` → `check_respawn`; it always registers
`on_localization_change` (which nils the cached `GUI`/`GUI2..5` window refs so they re-init in the new
locale). The open handler fires on `WINDOW_KEY_RELEASED`, maps the key with `dik_to_bind`, and when the
bind is `key_bindings.kCUSTOM17` (the debug key, **F7** by the header docstring) **and** `level.present()`
**and** the actor is alive, calls `start_debug_main(main_menu)` → `start_ui(GUI, UIDebugMain, owner)`.
Each sub-window is a `CUIScriptWnd` (`UIDebugMain`, `UIDebug_ItemSpawn`, `UIDebug_ObjSpawn`,
`UIDebug_Executer`, `UIDebug_FactionSwitch`), registered/unregistered through the F19 `start_ui` /
`Unregister_UI` pattern. The `hide_ui` field on each content entry controls whether executing an action
closes the launcher and unpauses (important when launched from the main menu, which leaves the game
paused).

### The `inject` extensibility API (verified)

`inject(key, v)` lets any script add an entry to the launcher's `content` table under a category
(`"target"`, `"action"`, …). An entry can carry a `functor` (a Lua function + params), a `cmd` (a
`debug_cmd_list` command name), a `console` (a console command with on/off state), an optional `key`
(DIK keybind), and the `hide_ui` policy. This is why feature code (anomaly zones, routes) can surface
its own dev shortcuts without editing the launcher.

## The HUD inspector (`xrs_debug_tools`) — verified

A frame-throttled overlay system, **not** a dialog. `on_game_start` (only if `DEV_DEBUG`) reads
`plugins\xrs_debug_tools.ltx` for keybinds, registers `main_menu_on_keyboard` and
`ActorMenu_on_item_focus_receive` (for the inventory-hover panel), and — if `other/debug_hud` is on and
a level is present — calls `activate_feature`.

- **Render loop.** `main_loop` runs at a 1 s throttle, lazily builds three `hud_tool` custom-statics
  (`cs_debug` top, `cop_debug` left, `cop2_debug` right — engine `AddCustomStatic` widgets), computes
  the nearest stalker, and **dispatches by name**: for each slot it calls
  `_G["xrs_debug_tools"][slot_hud .. "_show"](hud)`. So each panel is just a `<name>_show(hud)` function
  that pushes rows via the `hud_tool:add_msg` / `:display` / `:set_header` helpers. Panels cover squad
  info, the GOAP **planner** (current action/goal), heli, monster, space-restrictor, surge/arena, actor
  slots, feel-touch, zone membership, wounded, and stalker data.
- **Toggle keys.** `main_menu_on_keyboard` cycles each slot through its rotation list with three keys
  (defaults 1 / 2 / 3 → top / left / right), persisting the chosen index to `axr_main.config`
  (`xrs_debug_tools/toggle_*_key`) so it survives a reload. The `print_position` ("P") and legacy
  `debug_ui` ("S") branches are both **commented out**.
- **`activate_feature` side effects.** Sets `smart_terrain.dbg_hud` and `smart_terrain_warfare.dbg_hud`,
  flips `_G._ALIFE_CACHE_RECORD = true`, restores the saved slot indices, and builds the GOAP
  **action-id → name** reverse map (`actions_by_stalker_ids`, from `_G.xr_actions_id` plus a block of
  hardcoded planner ids) that the planner panel reads. It also registers `print_debug_hud` into the
  shared command table (`CMD = debug_cmd_list.command_get_list()`).

## The command registry (`debug_cmd_list`) — verified surface

A flat dictionary of named debug commands with a thin dispatcher: `command_get_list`,
`command_get_count`, `command_exists`, `command_give(name, ...)`, plus arg-splitting helpers
(`split` / `first_word` / `trim`). The launcher's `cmd=` entries and `xrs_debug_tools`'
`print_debug_hud` route through this table, so it is the shared bus between the typed/console surface
and the launcher buttons. The command bodies themselves *(structure-surveyed — large file, not line-read)*.

## The editors (launched from the launcher) — structure-surveyed

Each is a `CUIScriptWnd` started by a `start_*_editor` function on the launcher:

- **`ui_debug_item`** — weapon / outfit / artefact editors (`start_itm_weapon_editor` /
  `_outfit_editor` / `_artfact_editor`), with an inventory-cell preview; it shares the launcher's XML
  (`ParseFile "ui_debug_launcher.xml"`) and calls `ui_debug_launcher.resume()` to hand control back.
- **`ui_debug_weather`** — weather editor (`ui_debug_weather.start(GUI)`).
- **`ui_debug_lighting`** (+ helper **`ui_ctrl_lighting`**) — lighting / r2 console editor
  (`ui_debug_lighting.start(GUI)`).
- **`ui_debug_wpn_hud`** — weapon-HUD position editor (`ui_debug_wpn_hud.start(GUI)`); the launcher also
  has `reload_wpn_hud` / `toggle_debug_hud` helpers.
- **`ui_map_debug_ex`** — map-debug overlay (extras), and `start_workshop` hands off to the `ui_workshop`
  dialog.

Their entry points, launcher hand-back and shared XML are verified; the field/control bodies were not
line-read.

## The gamedata delta (runtime-winning) — verified

Three files in `gamedata/scripts/` change the picture at runtime (per the `gamedata/` override rule):

- **`zz_ui_debug_inputs` (new) — input-history recall.** A "better_debug_inputs" addon that
  **monkeypatches** the launcher's `CUIScriptWnd` methods (`UIDebug_Executer.OnKeyboard` /
  `OnExecute`, `UIDebug_ItemSpawn.OnKeyboard` / `OnButton_Name` / `OnItemList` / `On_CC_Mouse1`,
  `UIDebug_ObjSpawn.*`) by capturing the originals and wrapping them. It keeps a 3-level cache
  (file → class → input field) of the last typed strings/picks, persisted in `axr_main.config.ini`
  under `[better_debug_inputs]`, so up/down recalls previous entries in the executer and spawner fields.
  The `zz_` prefix sequences it to load after the launcher it patches.
- **`imgui_lua_debug` (new) — Lua remote debugger.** Registers an ImGui menu (the modded-exe ImGui
  surface) "Debug → LuaDebug" with Start / Stop / Reconnect that drive `debugger_attach` /
  `debugger_stop` / `debugger_reconnect` (the EmmyLua-style external Lua debugger), gated on a
  `DebuggerMode` flag. Independent of the in-game launcher.
- **`options_modded_exes_debug` (override) — debug options page.** Builds the modded-exe options-tree
  "debug" group with **logging** and **errors** subpages (it imports `options_builder` and the
  `options_modded_exes_logging` / `_errors` pages). This is the F17 options domain
  (`flows/options-runtime.md`), surfaced here only because it is the settings home for the debug flags
  (`other/debug_hud`, `other/debug_error`) that gate this subsystem.

## Engine-side debug & profiling tools (merged 2026.09–2026.11)

Not gated by `DEV_DEBUG`; these are engine features.

- **RenderDoc integration** (`xrEngine/renderdoc_integration.cpp`) — active when `renderdoc.dll` is
  injected or the game runs with `-renderdoc`. Console `rdoc_capture [frames 1..60]`, `rdoc_open`,
  `rdoc_overlay` (`xr_ioc_cmd.cpp`, `CCC_RenderDoc*`). Captures are annotated with engine state
  (`renderdoc_annotate_frame`); render targets carry debug names (`dx10DebugName.h`); R4 frame markers
  are coloured and skipped when nothing listens (`dx10EventWrapper.cpp`).
- **LuaJIT profilers** (`src/3rd party/luajit-2/src/lj_profile.c`, `lj_allocprof.c`, `lib_jit.c`) —
  `jit.profile` (backported 2.1 timer sampler, jittered interval), `jit.allocprof` (byte-sampled
  allocation attribution, optional stacks), `jit.util.gcstat` (GC totals + completed cycles). Mutually
  exclusive; a profile callback error no longer kills the game. API in `lua_help_ex.script`.
- **Shader bus monitor** — ImGui Debug > Shader bus (`imgui_shader_bus.script`; see
  `features/shader-bus.md`).
- **ImGui visibility callbacks** — `imgui_on_show` / `imgui_on_hide` (`callbacks_gameobject.script`).
- **`dump_cvar_enable`** (default on) — gates the cvar dump at start and on crash (`x_ray.cpp`,
  `XR_IOConsole.cpp`).
- **Sound event profiling** — sound events carry profiler zones (`PROF_EVENT`).

## Isolation ledger (the seams)

- **Boot/callbacks (F65):** all entry points are `main_menu_on_keyboard` / `on_key_release` /
  `on_game_load` / `on_localization_change` / `ActorMenu_on_item_focus_receive` registrations, each
  guarded by `DEV_DEBUG`. Nothing here runs without the dev flag.
- **Engine UI (F19):** sub-windows are `CUIScriptWnd` via `start_ui` / `Unregister_UI`; HUD panels are
  `AddCustomStatic` custom-statics; XML comes from `ui_debug_launcher.xml` through `CScriptXmlInit`
  (`flows/hud-ui-load.md`).
- **AI introspection (F12):** the HUD panels read live GOAP/squad/smart-terrain state
  (`actions_by_stalker_ids`, `smart_terrain.dbg_hud`, planner action ids) — the inspector is a read
  view onto `flows/ai-squad-logic.md`.
- **Command bus:** `debug_cmd_list` is the shared registry between launcher `cmd=` entries, the legacy
  console, and `xrs_debug_tools.print_debug_hud`.
- **Options (F17):** `other/debug_hud` toggles the HUD inspector, `other/debug_error` toggles the `_g`
  error reporter; the debug options page is `options_modded_exes_debug`.
- **Config:** `plugins\xrs_debug_tools.ltx` (HUD keybinds + position-file), `cache_dbg.ltx`
  (launcher cache), `axr_main.config` (`xrs_debug_tools/toggle_*_key`, `[better_debug_inputs]`).

## Threading

Main thread. The HUD inspector throttles its `main_loop` to 1 s; everything else is event-driven
(keyboard callbacks, dialog `Update`). Single-threaded with gameplay.

## Failure & edge cases

- **Nothing appears in a shipping build:** that's by design — `DEV_DEBUG` is the gate; the launcher key
  and HUD toggles don't register without it.
- **Launcher opened from the main menu leaves the game paused:** the `hide_ui` policy per content entry
  governs whether an action unpauses; the header warns to keep editor entries at `hide_ui = 2` so the
  main-menu/paused state is force-cleared on launch.
- **Legacy `ui_debug_main` looks wired but isn't:** its only stock opener (the `xrs_debug_tools` "S"
  branch) is commented out — use the launcher.
- **`zz_ui_debug_inputs` depends on the launcher's exact method names:** because it monkeypatches
  `ui_debug_launcher.UIDebug_*` methods by reference, renaming a launcher handler silently disables the
  input-history wrapper for that field.
- **`other/debug_hud` only activates with a level present:** `xrs_debug_tools.on_game_start` calls
  `activate_feature` only when `level.present()`; toggling it in the main menu does nothing until in a
  level.

## Deeper

- Launcher: `unpacked/scripts/ui_debug_launcher.script` (`prepare`, `inject`, `main_menu_on_keyboard`,
  `start_debug_main`, the `UIDebug_*` classes, `start_*_editor`)
- HUD inspector: `unpacked/scripts/xrs_debug_tools.script` (`main_loop`, `activate_feature`,
  `*_show(hud)` panels, the `hud_tool` class)
- Command registry: `unpacked/scripts/debug_cmd_list.script` (`command_give`, `command_get_list`)
- Legacy menu: `unpacked/scripts/ui_debug_main.script` (the `debug_ui` class)
- Editors: `unpacked/scripts/ui_debug_item.script`, `ui_debug_weather.script`, `ui_debug_lighting.script`
  (+ `ui_ctrl_lighting.script`), `ui_debug_wpn_hud.script`, `ui_map_debug_ex.script`
- Dev flag: `unpacked/scripts/_g.script` (`DEV_DEBUG`)
- Gamedata delta: `gamedata/scripts/zz_ui_debug_inputs.script`, `imgui_lua_debug.script`,
  `options_modded_exes_debug.script`
- Map: `modlayer/gameplay/ui-scripts.md` (M20); engine UI: `flows/hud-ui-load.md` (F19); AI state:
  `flows/ai-squad-logic.md` (F12); options: `flows/options-runtime.md` (F17)

## Maintenance

- Keep the "`DEV_DEBUG` gates everything" fact front — it explains why none of this appears in a normal
  build and is the first thing to check when debug tools "don't work".
- Promote the *(structure-surveyed)* editor interiors and the `debug_cmd_list` command bodies to
  verified if a task needs a specific editor field or command's exact behavior.
- Re-grep `gamedata/scripts/*debug*` and `zz_ui_debug_inputs`' patched method names on pickup — the
  gamedata delta is the runtime-winning layer and `zz_` addons are brittle against launcher renames.
