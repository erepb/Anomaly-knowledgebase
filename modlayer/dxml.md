---
domain: modlayer/dxml
tags: [modlayer, lua, dxml]
sources: [DXML.md, gamedata/scripts/dxml_core.script, gamedata/scripts/modxml_test.script, gamedata/scripts/slaxml.script]
related: [modlayer/_overview.md, modlayer/scripting.md, flows/dxml-xml-interception.md, engine/subsystems-misc.md, GLOSSARY.md]
updated: 2026-06-04
---

# DXML — XML manipulation from Lua

Lets mods transform XML files (UI layouts, dialogs, some text) before the engine parses them, without
overwriting whole XML files. By demonized. **Full guide: `DXML.md` (repo root) — read it for the API
and worked examples.** This file is the condensed map.

> Full runtime sequence (engine bridge → `COnXmlRead` → dispatch → reload) with function digests:
> **`flows/dxml-xml-interception.md`**. Summary below.

## How it works

1. Engine loads an XML file; the DXML hook passes the XML string to Lua (interception around the
   engine XML parser, `engine/subsystems-misc.md`; consumers `src/xrGame/ui/UIXmlInit.cpp`,
   `ScriptXMLInit.cpp`).
2. `dxml_core.script` fires the `on_xml_read` callback with `(xml_file_name, xml_obj)`, converting the
   string into a DOM-like `xml_obj` (built on `slaxml.script`).
3. Your patcher mutates `xml_obj`; DXML serializes it back and returns it to the engine.

## Writing a patcher

Create `gamedata/scripts/modxml_<yourname>.script` (the `modxml_` prefix is required). Expose
`function on_xml_read()` that calls `RegisterScriptCallback("on_xml_read", fn)`; inside, compare
`xml_file_name` to the target (e.g. `text\eng\_game_version.xml`) and edit `xml_obj` when it matches.
See `modxml_test.script`, `modxml_inject_keybinds.script`, `modxml_lua_fatal_window.script` for live
examples.

## xml_obj API (see DXML.md for signatures)

`query`, `getText`, `setText`, `setElementAttr`, `add`, `insertFromXMLString`, `insertFromXMLFile`
(`GLOSSARY.md#xml_obj`).

## Limits

DXML does NOT process translation strings outside `eng/`/`rus/` folders and
`gameplay\character_desc_general.xml` (per DXML.md "Additional functions"). For those, use the
dedicated functions described in DXML.md.

## Deeper

- API + examples (authoritative): `DXML.md`
- Core/runtime: `gamedata/scripts/dxml_core.script`, `gamedata/scripts/slaxml.script`
- Example patchers: `gamedata/scripts/modxml_*.script`
- Engine interception/consumers: `src/xrXMLParser/xrXMLParser.cpp`,
  `src/xrGame/ui/UIXmlInit.cpp`, `src/xrGame/ScriptXMLInit.cpp`

## Maintenance

- Don't duplicate `DXML.md` here — if the API changes, update `DXML.md` and just refresh the method
  list / pointers in this file.
- If the engine interception point moves, sync with `engine/subsystems-misc.md`.
