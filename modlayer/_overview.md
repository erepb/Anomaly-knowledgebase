---
domain: modlayer
tags: [modlayer, config, lua]
sources: [gamedata/, gamedata/scripts/, gamedata/configs/, src/xrCore/LocatorAPI.cpp, unpacked/]
related: [INDEX.md, GLOSSARY.md, modlayer/scripting.md, modlayer/config-ltx.md, modlayer/dxml.md]
updated: 2026-06-05
verified: code
---

# Mod layer overview (gamedata/)

What this repo ships on top of the engine: the Lua + config overlay that the modded exes need. This
is NOT the full Anomaly game — only the additions/overrides bundled with the patched exes.

## Layout

- `gamedata/scripts/` (~66 `.script`/`.lua`) — the Lua modules: DXML core, options framework,
  callback infra, weapon/AI patches, imgui helpers, migration. → `modlayer/scripting.md`,
  `modlayer/options-ui.md`, `modlayer/dxml.md`
- `gamedata/configs/` — LTX config, `mod_system_*.ltx` DLTX patches, plus subfolders:
  - `configs/ui/` — UI layout configs
  - `configs/text/` — localization strings
  - `configs/unlocalizers/` — local→global var promotion configs (`modlayer/unlocalizer.md`)
  - root `mod_*.ltx` (e.g. `mod_system_base_hud.ltx`, `mod_script_dxml.ltx`) → `modlayer/config-ltx.md`
- `gamedata/shaders/{r1,r2,r3}` — HLSL shaders (R3 also serves R4/DX11). → `modlayer/assets.md`
- `gamedata/textures/`, `gamedata/materials/`, `gamedata/levels/` → `modlayer/assets.md`

## Load priority & where content really lives

`gamedata` is a recognized FS entry point — verified: `LocatorAPI.cpp` special-cases the
`"gamedata"` alias (`fs_entry_point += "gamedata\\"`). **Override mechanism verified**: all content
(loose files via `ProcessOne`, archive contents via `ProcessArchive`/`LoadArchive`) is registered into
one `m_files` registry that dedups by path descriptor, and **the last registration of a given path
overwrites the earlier one in place** (`const_cast<file&>(*I) = desc`, `LocatorAPI.cpp`). So
precedence = registration order. *(INFERRED — that the loose `gamedata/` folder is scanned after the
packed `db/` archives so it wins; the concrete order depends on `fs.ltx` + directory traversal and was
not traced.)* This repo's `gamedata/` is the editable runtime overlay. `unpacked/` is a **read-only reference** copy of base assets/scripts (project convention,
not a code fact); never edit it (see `GLOSSARY.md#unpacked`).

## DLTX/DXML/unlocalizer in one line

The mod layer customizes the base game without overwriting whole files: **DLTX** patches `.ltx`
configs (`modlayer/config-ltx.md`), **DXML** patches XML from Lua (`modlayer/dxml.md`), and the
**unlocalizer** exposes selected script locals (`modlayer/unlocalizer.md`).

## Deeper

- Scripts index: list `gamedata/scripts/`
- Config index: list `gamedata/configs/` and `gamedata/configs/*/`
- Reference base copy (read-only): `unpacked/`

## Maintenance

- If new top-level `gamedata/` subfolders appear, add them to the layout list.
- Keep the "what this repo ships vs base game" framing accurate — it prevents wasted searches for
  base-game scripts that aren't here (look in `unpacked/` or the actual Anomaly install instead).
