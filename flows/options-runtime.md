---
domain: flows/options-runtime
tags: [modlayer, lua, options, ui, dxml, console, config, flow]
sources: [gamedata/scripts/options_builder.script, gamedata/scripts/options_modded_exes.script, gamedata/scripts/ui_options_modded_exes.script, gamedata/scripts/dxml_core.script, gamedata/configs/text/eng/ui_mm_modded_exes.xml, gamedata/scripts/lua_help_ex.script, src/xrGame/console_commands.cpp, src/Layers/xrRender/xrRender_console.cpp, unpacked/scripts/ui_options.script]
related: [flows/dxml-xml-interception.md, flows/lua-callback-lifecycle.md, flows/sound-music.md, flows/plugins-tweaks-presets-configs.md, flows/renderer-resources-shaders.md, modlayer/options-ui.md, modlayer/scripting.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# Flow: options framework runtime

The end-to-end life of a modded-exes game option: **defined** in Lua via a small DSL, **aggregated**
into one tree, **injected** into the base game's options UI through the DXML `on_xml_read` seam,
**rendered** by the base `UIOptions` class, and on OK **applied** to a console variable and/or the
`axr_options.ltx` config, then read back to affect behavior. Extends `modlayer/options-ui.md` and the
`ui_options` API quirks.

**Provenance:** spine personally code-verified — `options_builder.script` (`check`, `track`, `page`,
`group`, `import_into`), `options_modded_exes.script` (root `GROUP`), `ui_options_modded_exes.on_xml_read`,
the `dxml_core.script` force-load do-block, and `unpacked/scripts/ui_options.script` `UIOptions:On_Accept`.
The base `UIOptions` render/cache methods (`GetCurrentValue`, `CacheValue`) and the C++ console binding
(`exec_console_cmd`/`get_console_cmd`) are *(agent-traced / engine boundary — not re-read here)*.

## 1. Definition — the DSL (verified)

`gamedata/scripts/options_builder.script` exposes constructor functions that just stamp a descriptor
table with type info:
- **`check(args)`** → `type="check"`, `val=1`, `def` defaults false, `cmd` defaults to `id`.
- **`track(args)`** (numeric slider) → `type="track"`, `val=2`; if `cmd` is a string it reads
  `get_console():get_variable_bounds(cmd)` to fill `min`/`max`; swaps min/max if inverted.
- **`page(args)` / `group(args)`** — pop the first element as the header table; the rest become `gr`
  (children). `page` also inserts a `slide` header row.
- helpers: `list`/`list_enum`/`list_bool`, `crosshair_color`/`vector` (composite tracks), `desc`,
  `line`, `slide`, `button`, `filter_false`, `get_item_if_exists`.
- **`import_into(dest)`** — patches `dest`'s metatable `__index` so a module can call `check{...}`,
  `track{...}`, `group{...}` etc. as if local, chaining to the old `__index`.

## 2. Aggregation (verified)

Each feature module (`options_modded_exes_<area>.script`) calls `options_builder.import_into(this)` and
builds a `GROUP` or `PAGE`. **`gamedata/scripts/options_modded_exes.script`** imports the builder and
assembles the root: `GROUP = group { { id="modded_exes" }, group_visual, group_sound, group_control,
group_gameplay, group_saves, page_optimizations, group_debug }`.

## 3. Injection via the DXML seam (verified)

A top-level `do ... end` block in **`gamedata/scripts/dxml_core.script`** runs at script-load: it scans
`modxml_*.script` files for an `on_xml_read` and calls each, then **force-loads** a fixed list — which
contains `["ui_options_modded_exes"] = true` — calling its `on_xml_read` even though it is not a
`modxml_` file. (DXML interception generally: `flows/dxml-xml-interception.md`.)

**`ui_options_modded_exes.on_xml_read`** wraps the base initializer: it saves
`ui_options.init_opt_base`, replaces it with a function that calls the original and then
`table.insert(ui_options.options, options_modded_exes.GROUP)` (splicing the whole modded tree into the
menu), and finally invokes `ui_options.init_opt_base()`. The same wrapper patches a few **vanilla**
rows:
- `sound_device` content via `ui_options.get_opt_table("sound/general/sound_device")` — works because
  that path resolves a **control row** (the documented `get_opt_table` quirk).
- a **monitor** list and a **persistent_weather** check are inserted by manually **walking
  `ui_options.options` by `id`** (`video`→`basic`/`weather`) — because those are group insertions, not
  control-row lookups; on insert it sets `need_init_opt_coder` and calls `ui_options.init_opt_coder()`.

## 4. Render & pending edits (base game; agent-traced)

The base `unpacked/scripts/ui_options.script` `UIOptions` class builds rows from the descriptor tree.
`GetCurrentValue` resolves a row's current value from (in order) the pending `opt_temp` cache, a `curr`
callback, a `cmd` console var (`get_console_cmd`), or `axr_main.config`. Editing a control stores the
pending value in `opt_temp[id]` (and marks the row changed). *(These two methods are agent-traced.)*

## 5. Apply on OK (verified)

**`UIOptions:On_Accept`** iterates `opt_temp`; per entry (`v = self:GetOption(id)`):
- if no `v.curr` callback → write to config: `axr_main.config:w_value(opt_section, id, val)`, mark
  `Save_AXR`.
- `Change_Done = true`; run `v.functor` (guarded by `v.postcondition` if present).
- set `Need_Restart` (`v.restart`) / `Need_VidRestart` (`v.vid`) flags.
- if `v.cmd` → coerce booleans (`bool_to_num` → `"1"/"0"` else `"on"/"off"`) and
  **`exec_console_cmd(v.cmd .. " " .. cmd_value)`**, mark `Save_CFG`.

Then: if `Save_AXR` → `axr_main.config:save()`; keybinds saved via `COptionsManager` if shown; if
`Save_CFG` → `exec_console_cmd("cfg_save")` (persists engine console vars to user.ltx); if
`level.present() and Change_Done` → **`SendScriptCallback("on_option_change")`**; clear
`opt_temp`/`opt_backup`; close.

## 6. Read-back

Behavior reads the value back through the same two stores: console vars via `get_console_cmd` / engine
queries of the cvar (engine boundary), and script-only options via `axr_main.config:r_value(...)`. The
`on_option_change` callback lets in-level systems react immediately to a change.

## Isolation ledger (the seams)

- **DXML callback seam:** `RegisterScriptCallback("on_xml_read", ...)` (patched by `dxml_core`) +
  the explicit force-load of `ui_options_modded_exes` — this is *how* the options tree reaches the menu
  (`flows/dxml-xml-interception.md`).
- **Script callback fired:** `on_option_change` (only in-level, only when something changed).
- **Console var API (Lua→engine):** `get_console():get_variable_bounds(cmd)` and `get_token_list(cmd)`
  (slider bounds / enum choices), `get_console_cmd(type, cmd)` (read), `exec_console_cmd(str)` (write),
  `cfg_save` (persist cvars to user.ltx). The cvar registry itself is C++ (engine boundary).
- **Config store (Lua):** `axr_main.config` = `ini_file_ex` over `axr_options.ltx` (script-only
  options, in user appdata); `w_value`/`r_value`/`save`.
- **UI framework:** `ui_options.options` (descriptor tree), `opt_temp` (pending edits), `opt_backup`,
  `get_opt_table` (control-row lookup only), `init_opt_base`/`init_opt_coder`.
- **Two persistence targets:** options with a `cmd` go to a **console var** (cfg_save → user.ltx);
  options without a `curr` callback go to **axr_options.ltx**. Some do both.

## Live overrides vs reference

- **LIVE (`gamedata/scripts/`, the mod layer):** `options_builder.script`, `options_modded_exes.script`
  + the per-area `options_modded_exes_*.script` modules, `ui_options_modded_exes.script`,
  `dxml_core.script`.
- **REFERENCE (`unpacked/scripts/`, base game, not overridden):** `ui_options.script` (the `UIOptions`
  render/cache/apply class). The mod layer extends it via the `on_xml_read` wrapper rather than
  replacing it.

## Adding a row: the four-file checklist (worked from recent additions)

Every option added recently touched exactly these places, and missing one is the usual "my option does
not show up / shows a raw id" bug:

1. the **cvar** — `CMD4(CCC_Integer, "<name>", &<global>, lo, hi)` in `console_commands.cpp`
   (gameplay) or `xrRender_console.cpp` (`r__*` renderer vars);
2. the **row** — `list_bool { id = "<name>" }` or `track { id = "<name>", def, step }` in the right
   `options_modded_exes_*.script` page. Rows for cvars that may not exist in every exe are wrapped in
   `get_item_if_exists` (that is how the renderer eviction rows are written);
3. the **label** — `ui_mm_modded_exes_<page-path>_<name>` (plus optional `_desc`) in
   `gamedata/configs/text/eng/ui_mm_modded_exes.xml` **and** the `rus/` copy; the label text
   conventionally repeats the cvar name in parentheses;
4. the **doc line** — the cvar list at the top of `gamedata/scripts/lua_help_ex.script`.

Recent examples to copy: `allow_accel_during_lookout` (gameplay/general),
`scale_hud_motion_marks_by_speed` (gameplay/weapons), `pseudogiant_dodge_stomp_while_falling`
(gameplay/monsters), `r__tex_evict_enabled` / `_age_frames` / `_batch_size` (optimizations,
`flows/renderer-resources-shaders.md`).

## Failure & edge cases

- **`get_opt_table` on a group returns nothing** — the monitor/weather insertions deliberately walk the
  tree by `id` instead (this is the recurring `ui_options` quirk; see the memory).
- **A `track` row's `def`/`step` are not validated against the cvar's range.** The shipped
  `r__tex_evict_age_frames` / `r__tex_evict_batch_size` rows both declare `def = 1024, step = 32`,
  which is outside the batch-size cvar's 10–1000 clamp — the engine clamps, the row lies. Read the
  `CMD4` bounds, not the row, when documenting a value.
- **`r__tex_evict_interval` has a cvar but no row**, so it is console/user.ltx only.
- **Slider bounds depend on the cvar existing** — `track` reads `get_variable_bounds`; a non-string or
  unknown `cmd` nils the command and the bounds are skipped.
- **Restart-required options** only set flags (`Need_Restart`/`Need_VidRestart`); they don't force a
  restart — the UI surfaces that separately.
- **`on_option_change` is in-level only** — changes made from the main menu don't fire it.

## Deeper

- DSL: `gamedata/scripts/options_builder.script` (`check`, `track`, `page`, `group`, `import_into`)
- Tree: `gamedata/scripts/options_modded_exes.script` + `options_modded_exes_*.script`
- Injection: `gamedata/scripts/ui_options_modded_exes.script` (`on_xml_read`),
  `gamedata/scripts/dxml_core.script` (the force-load `do` block)
- Render/apply: `unpacked/scripts/ui_options.script` (`UIOptions:On_Accept`, `GetCurrentValue`)
- Concepts + quirks: `modlayer/options-ui.md`, `modlayer/config-ltx.md`

## Maintenance

- Re-grep the `force_load` table in `dxml_core.script` if the injected script name changes; re-grep
  `ui_options_modded_exes.on_xml_read` if the vanilla-row patches drift.
- Keep the "two persistence targets" (console var vs `axr_options.ltx`) and the `get_opt_table`-only-
  for-control-rows quirk documented — both are easy to get wrong.
- Promote the *(agent-traced)* `GetCurrentValue`/`CacheValue` base-game detail to verified if a task
  needs the exact precedence of pending vs cvar vs config.
