---
domain: modlayer/options
tags: [modlayer, options, ui, imgui, lua]
sources: [gamedata/scripts/options_builder.script, gamedata/scripts/options_modded_exes.script, gamedata/scripts/ui_options_modded_exes.script, gamedata/scripts/dxml_core.script]
related: [modlayer/scripting.md, flows/dxml-xml-interception.md, flows/options-runtime.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Options framework

The modded-exes settings system: a Lua DSL builds nested option-descriptor tables, which are injected
into the base-game options menu (`ui_options`). Verified by reading the three scripts below. For the
full runtime lifecycle (definition → DXML force-load injection → render → `On_Accept` → console var /
`axr_options.ltx` → read-back) see `flows/options-runtime.md`.

## How it works (verified)

1. **DSL constructors** — `options_builder.script` defines functions that take an `args` table, stamp
   descriptor fields, and return it: `check` (`type="check"`, `val=1`, `cmd=args.id`), `track`
   (numeric slider; reads `get_console():get_variable_bounds(args.cmd)` for min/max), `slide`,
   `button`, `line`, `list`/`list_enum`/`list_bool`, `input`, `vector`, `crosshair_color`, `desc`.
   Containers: `page(args)` and `group(args)` take `{header, child1, child2, …}`, set
   `header.gr = children` (the child list), and return the header. Rows bind to a
   console variable via `cmd` (defaulting to `id`).
2. **DSL import** — `options_builder.import_into(dest)` installs a metatable `__index` on a
   script so it can call the constructors as if local. **28** `options_modded_exes_*.script` feature
   modules call `options_builder.import_into(this)` and build a `GROUP` (or `PAGE`) descriptor.
3. **Aggregation** — `options_modded_exes.script` collects each feature module's `.GROUP`/`.PAGE` and
   wraps them in a root `GROUP = group { { id = "modded_exes" }, group_visual, group_sound, … }`.
4. **Injection into the menu** — `ui_options_modded_exes.script` exposes `on_xml_read()` (force-loaded
   by `dxml_core.script`, see `flows/dxml-xml-interception.md`). It **wraps**
   `ui_options.init_opt_base`: the wrapper calls the original, then
   `table.insert(ui_options.options, options_modded_exes.GROUP)` — adding the whole modded tree to the
   base options. It also patches some vanilla rows (see Quirk).

## Descriptor shape (verified)

Option entries are plain Lua tables. Common fields seen: `id`, `type` (`"check"`/`"track"`/`"list"`/
`"slide"`/`"desc"`/…), `cmd` (console var), `val`, `def`, `content` (for lists; often a function),
`restart`, `no_str`, and `gr` (child list for pages/groups). A group/page is just a row with a `gr`.

## Adding an option

In an `options_modded_exes_<area>.script`: `options_builder.import_into(this)`, then build
`GROUP = group { { id = "<area>" }, check{ id="my_opt", cmd="my_cvar", def=true }, … }`, and add it to
the aggregator list in `options_modded_exes.script`. Bind `cmd`/`id` to the console variable the
engine reads. Use `get_item_if_exists`/`filter_false` for conditional rows.

## Quirk: get_opt_table vs id-walk (verified in `ui_options_modded_exes.script`)

- `ui_options.get_opt_table("sound/general/sound_device")` fetches an existing **control row**
  by slash path — used to tweak `restart`/`content` on the vanilla sound-device row.
- For inserting **new rows into a group**, the script does **not** use `get_opt_table`; it **walks
  `ui_options.options` by `id`** and `table.insert`s the new descriptor at a computed index. Two
  verified cases: a `monitor` dropdown (`vid_monitor`) is inserted just after the `renderer` row under
  `video`→`basic`; a `persistent_weather` switch is inserted just before the `clear_slide` row under
  `video`→`weather`. After such inserts it calls `ui_options.init_opt_coder()` if
  `need_init_opt_coder` (when present).
- Takeaway (confirmed by code): `get_opt_table` resolves control rows, not group containers; to add
  rows under a group, traverse `ui_options.options` by `id`. (`ui_options` is a base-game module in
  `unpacked/scripts/`; not in this repo.)

## ImGui

The repo ships ImGui helper scripts (`imgui_helper.script`, `imgui_lua_debug.script`,
`lua_help_imgui.script`, `_imgui_groups.script`, `zz_ui_debug_inputs.script`). The core `ImGui.*`
binding is provided by the **engine** (`xrEngine`, `engine/engine-core.md`); these scripts extend it
in Lua. Verified (`imgui_helper.script`): defines `_G.ImGui.HelpMarker(text, sameline, …)` (a "(?)"
tooltip built from `ImGui.TextDisabled`/`BeginItemTooltip`/`PushTextWrapPos`/`Text`/`EndTooltip`),
`_G.ImGuiMod` (Ctrl/Shift/Alt/Super key-mod bitmasks), and a large `_G.ImGui_Icon` table mapping
Font-Awesome icon names → UTF-8 glyph strings. *(INFERRED — the full engine-side `ImGui.*` binding
surface; only the Lua helpers above were read.)*

## Deeper

- DSL + constructors: `gamedata/scripts/options_builder.script`
- Aggregation: `gamedata/scripts/options_modded_exes.script`; per-feature: `options_modded_exes_*.script`
- Menu injection + quirk: `gamedata/scripts/ui_options_modded_exes.script`
- Force-load + XML context: `gamedata/scripts/dxml_core.script`, `flows/dxml-xml-interception.md`
- Base menu module `ui_options` (reference): `unpacked/scripts/ui_options.script`

## Maintenance

- If `options_builder` constructors change, re-derive the field list from them (source of truth).
- Re-verify the quirk against `ui_options_modded_exes.script` if `ui_options` internals change.
- The ImGui section is unverified — read those scripts before relying on it, then drop the marker.
