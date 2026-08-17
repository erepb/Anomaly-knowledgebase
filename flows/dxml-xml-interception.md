---
domain: flows/dxml-xml-interception
tags: [modlayer, lua, dxml, flow, lifecycle]
sources: [src/xrXMLParser/xrXMLParser.cpp, src/xrGame/ScriptXMLInit.cpp, gamedata/scripts/dxml_core.script, DXML.md, gamedata/scripts/slaxml.script]
related: [modlayer/dxml.md, flows/lua-callback-lifecycle.md, flows/pda-map-news.md, engine/subsystems-misc.md, METHODOLOGY.md, GLOSSARY.md]
updated: 2026-08-17
---

# Flow: DXML XML interception

How a mod transforms an XML file's contents (UI layouts, dialogs, eng/rus text) before the engine
consumes it — without overwriting the file. C++ parser hands the raw XML string to Lua, Lua mutates a
DOM and returns a new string, the C++ parser reloads from it. The `xml_obj` DOM **API itself is in
`DXML.md`** (authoritative); this file is the runtime flow + digests. Line numbers are
current-as-of-`updated` hints; trust the `file:function` names.

## Trigger / entry

Any engine XML load. `CXml::Load(...)` reads the file into a buffer; right after, the parser calls
the Lua bridge once per file (`xrXMLParser.cpp`).

## Sequence

### Phase A — Registration (at script load, before any XML is parsed)

1. `dxml_core.script` runs as a module. It saves the base callback API
   (`RSC = _G.RegisterScriptCallback`) and **monkey-patches** `_G.RegisterScriptCallback`: calls with
   `name == "on_xml_read"` are appended to a private `xmlCallbacks` list (deduped); all other names
   fall through to `RSC` (the `_g.script`→`axr_main` bus, see `flows/lua-callback-lifecycle.md`).
2. `dxml_core` `AddScriptCallback("on_xml_read")` (+ `on_specific_character_dialog_list`,
   `on_specific_character_init`).
3. **modxml gather** (the `do … end` block): `dxml_core` enumerates `$game_scripts$` **root**
   `modxml_*.script`, `table_sort`s them (alphabetical → deterministic order, unlike the FS-ordered
   `axr_main` bootstrap), and calls each module's `module.on_xml_read()` *init* function. Inside that
   init, modules call `RegisterScriptCallback("on_xml_read", handler)` to append their real per-file
   `handler(xml_file_name, xml_obj, flags)` to `xmlCallbacks`. Then it force-loads
   `ui_options_modded_exes.on_xml_read()`.

### Phase B — Per-file interception (every XML load)

4. **C++ → bridge**: `xrXMLParser.cpp` calls `XMLLuaCallback(*this, xml_string)`.
5. **bridge → Lua**: `XMLLuaCallback` (`ScriptXMLInit.cpp`) strips the BOM, looks up the Lua functor
   `_G.COnXmlRead`, calls `res = COnXmlRead(m_xml_file_name, xml_string)`, then
   `m_xml.LoadFromString(res)` — re-parsing the (possibly modified) string back into the engine DOM.
6. **Lua transform**: `dxml_core._G.COnXmlRead(file, xml_string)` (`dxml_core.script`):
   - **Skip** files under `text\` that are not `text\eng\` or `text\rus\` → return unchanged.
   - **Skip** `gameplay\character_desc_general.xml` (too large) → return unchanged.
   - **Cache hit**: if `xml_string_cache[file]` exists, return it (callbacks not re-fired).
   - Parse to a table: `parser.simple_dom(xml_string, parser_options)` (slaxml). On failure, return
     original.
   - Wrap in `xml_obj = xml_object(file, xml_string, xml_table)` (the DOM mods manipulate).
   - `flags = { cache = false }`.
   - **`xmlDispatch(file, xml_obj, flags)`** — fire every handler in `xmlCallbacks` in order.
   - Serialize back: `xml_string = parser:simple_xml(xml_table)`.
   - If a handler set `flags.cache = true`, store result in `xml_string_cache[file]`.
   - Return the new string.

### Phase C — Special character hooks (separate entry points)

7. `specific_character.cpp` (engine) calls Lua `_G.CSpecificCharacterDialogList(id, dialog_list)` and
   `_G.CSpecificCharacterInit(id, data)` (`dxml_core.script`), which fire
   `on_specific_character_dialog_list` / `on_specific_character_init` through the **normal** bus and
   return the (mutated) list/data. This is how `character_desc_*` data is edited despite the
   `COnXmlRead` skip.

## Function digests (L3)

- **`xrXMLParser.cpp`** — *Does:* after reading an XML file to a buffer, call the extern
  `XMLLuaCallback(*this, buffer)`. *Pairs with:* the bridge below.
- **`ScriptXMLInit.cpp:XMLLuaCallback(CXml& m_xml, LPCSTR xml_string)`** — *Does:* C++↔Lua
  bridge — clear BOM, get functor `_G.COnXmlRead`, call it with `(file_name, xml_string)`, reload
  engine DOM from the returned string via `m_xml.LoadFromString`. *Gotcha:* no-op if string empty or
  functor missing (DXML silently inert).
- **`dxml_core.script:_G.COnXmlRead(file, xml_string)`** — *Does:* the transform pipeline
  above (skip/cache/parse/dispatch/serialize). *Reads/Writes:* `xml_string_cache`. *Returns:* XML
  string (modified or original). *Gotcha:* returns original on any parse error — failures degrade,
  don't crash.
- **`dxml_core.script:xmlDispatch(file, xml_obj, flags)`** — *Does:* `ipairs` over
  `xmlCallbacks`, calling each `handler(file, xml_obj, flags)`. *Reads:* `xmlCallbacks`.
- **`dxml_core.script:RegisterScriptCallback` override** — *Does:* route `on_xml_read`
  registrations into `xmlCallbacks` (dedup); delegate everything else to base `RSC`. *Writes:*
  `xmlCallbacks`.
- **`dxml_core.script` modxml gather block** — *Does:* find+sort `modxml_*.script`
  root files, call each `on_xml_read()` init; force-load `ui_options_modded_exes`. *Gotcha:*
  `table_sort` → deterministic init order; the `on_xml_read()` here is the module *init*, distinct
  from the per-file *handler* it registers.
- **`dxml_core.script:openXMLFile(file)`** — *Does:* load a config XML off
  `$game_config$`, parse, apply DXML edits via `xmlDispatch`, return the `xml_obj` (for editing files
  outside the callback path).
- **`dxml_core.script:CSpecificCharacterDialogList/Init`** — *Does:* engine-invoked
  hooks for character-desc data; fire the corresponding bus callbacks and return the mutated arg.

## State touched

- `dxml_core` locals: `xmlCallbacks` (handler list), `xml_string_cache` (per-file result cache),
  the saved `RSC`/`URSC` originals.
- Engine: `CXml m_xml` DOM (reloaded from the Lua-returned string).

## Threading

XML load and the Lua transform are synchronous on the loading thread (UI/load). No DXML-specific
locking; relies on Lua running single-threaded for these calls.

## Extension points

Write `gamedata/scripts/modxml_<name>.script` exposing `function on_xml_read()` that calls
`RegisterScriptCallback("on_xml_read", function(file, xml_obj, flags) … end)`. Match `file` to the
target path, mutate `xml_obj` (API in `DXML.md`), optionally set `flags.cache = true` for static
results. See `modlayer/dxml.md` and example `modxml_*.script` files.

## Failure & edge cases

- **Non-eng/rus `text\` files and `character_desc_general.xml` are never processed** by `COnXmlRead`
  (use the `CSpecificCharacter*` hooks / DXML.md "Additional functions" for character data).
- **Parse errors return the original string** — a malformed edit silently no-ops rather than
  crashing; check the DXML parser log lines.
- **`flags.cache = true` freezes a file** — subsequent loads return the cached string and **skip all
  callbacks**; don't cache files other mods may also edit.
- The `on_xml_read` name is invisible in `axr_main.intercepts` (it lives in `xmlCallbacks`).

## The `parent` invariant (mutation correctness, 2026-08 fixes)

Every node in a DXML tree carries a `parent` back-pointer, and the mutation helpers rely on it —
`insertElement` (before/after), `removeElement` and the sibling selectors all navigate *upward* first.
The 2026-08 correctness pass in `dxml_core.script` made maintaining it a rule rather than an accident:

- a local recursive `setParent(node, parent)` is applied on **every** insertion path
  (`insertElementIn*`, before/after, `insertFromXMLString`), so a subtree inserted from a parsed string
  has parents all the way down, not just at its root; `setText` fixes up the text child it creates; and
  `removeElement` **clears** the removed node's parent.
- `slaxml.script` sets `parent` on the processing-instruction node it pushes (`?xml …`), which
  previously had none.
- `insertElement` used to insert into `el.parent` itself rather than `el.parent.kids` (corrupting the
  node) and computed its position by iterating; it now bails with a parser error when the target has no
  usable parent, and uses `getElementPosition`.
- `insertFromXMLString` creates `where.kids` if absent, returns early on an empty source document, and
  inserts the source's children **in order** at the requested position (it previously reversed them by
  re-clamping per child).
- the `+` (next sibling) selector unwraps `findFirstSibling`'s result list instead of storing the list
  itself as the match.

**Reverted, deliberately:** an earlier commit in the same pass made *malformed closing tags* acceptable
in `slaxml`; it broke existing mods and was reverted the same week. Malformed XML is a parse error
again (and a parse error means the whole file silently keeps its original content — see below).

## Subtleties

- Two senses of `on_xml_read`: (1) a *module function* `module.on_xml_read()` called once at gather
  time (init), and (2) the *callback name* a module registers a per-file handler under. Don't conflate.
- **A mutation helper that reports "target element has no parent"** means the node did not come from
  this tree (e.g. it was built by hand, or removed earlier) — find it with `query`/`findElement` on the
  live `xml_obj` instead of reconstructing it.
- `dxml_core` exists in both `unpacked/scripts/` (base) and `gamedata/scripts/` (repo override);
  document/verify against the repo copy.

## Deeper

- Engine bridge: `src/xrXMLParser/xrXMLParser.cpp` (call site), `src/xrGame/ScriptXMLInit.cpp`
  (`XMLLuaCallback`)
- Lua core: `gamedata/scripts/dxml_core.script` (`COnXmlRead`, `xmlDispatch`, RSC override, gather
  block, `xml_object`), `gamedata/scripts/slaxml.script` (parser)
- `xml_obj` API + examples: `DXML.md`
- Registration mechanics: `flows/lua-callback-lifecycle.md`

## Maintenance

- Re-grep anchors if lines drift: `XMLLuaCallback` (ScriptXMLInit.cpp), `_G.COnXmlRead` /
  `function xmlDispatch` (dxml_core.script).
- If the engine adds more `_G.C*` XML hooks (like the character ones), add them to Phase C.
- Keep the `xml_obj` API out of here — it lives in `DXML.md`.
