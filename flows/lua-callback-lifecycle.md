---
domain: flows/lua-callback-lifecycle
tags: [modlayer, lua, callbacks, luabind, flow, lifecycle]
sources: [src/xrServerEntities/script_engine.cpp, src/xrServerEntities/script_storage.cpp, unpacked/scripts/_g.script, gamedata/scripts/axr_main.script, gamedata/scripts/dxml_core.script, gamedata/scripts/dynamic_callbacks.lua, gamedata/scripts/_g_patches.script, docs/axr_main.md]
related: [modlayer/scripting.md, modlayer/dxml.md, engine/server-entities.md, METHODOLOGY.md, GLOSSARY.md]
updated: 2026-11-13
---

# Flow: Lua boot & callback lifecycle

The spine of the mod layer: how the engine starts Lua, how modules bootstrap, how the callback bus is
built, and how events dispatch at runtime. Almost every mod feature hangs off this. Spans base game
(`unpacked/scripts/`, read-only) + this repo's overrides (`gamedata/scripts/`).
Line numbers are current-as-of `updated` hints; trust the `file:function` names.

## Trigger / entry

Engine process start → renderer/game DLLs loaded → `CScriptEngine` builds the Lua VM. Then, on
**new game / save load / level change**, the engine calls the Lua global `on_game_start()`.

## Sequence

### Phase A — Engine builds the Lua VM (C++)

1. **`CScriptEngine::init()`** (`script_engine.cpp`) — creates the Lua state, calls
   `CScriptStorage::reinit()`, registers C++→Lua exports (luabind), then loads scripts.
2. **Kernel scripts → globals (DEBUG ONLY)** — during storage setup, inside the `isDebugEnabled`
   block (`script_storage.cpp`), `LoadKernelScriptToGlobal` injects `global.lua`,
   `dynamic_callbacks.lua`, `socket.lua`, `LuaPanda.lua` into the global state. **Verified: these load
   only when `lua_debug` is enabled** — in a normal (non-debug) run `dynamic_callbacks.lua` is *not*
   loaded, so its globals cannot shadow `_G`'s callback API (see "Subtleties"). socket+LuaPanda are
   the debugger transport.
3. **Load `_G`** — `init()` calls `process_file_if_exists("_G", false)` (`script_engine.cpp`),
   loading base `unpacked/scripts/_g.script`. This **defines the public callback API** as globals,
   each delegating to `axr_main` (see digests).
4. **Load common scripts** — `CScriptEngine::load_common_scripts()` (`script_engine.cpp`) reads
   `$game_config$/script.ltx` `[common] script = a, b, c…`, `process_file`s each module, and calls
   each module's `<name>_initialize()` if present. (Init phase — distinct from `on_game_start`.)
5. **DXML declares `on_xml_read`** — `dxml_core.script` (loaded as a module) calls
   `AddScriptCallback("on_xml_read")`. Until 2026.10.11 it monkey-patched `RegisterScriptCallback` into a
   private list; that wrapper is gone and `on_xml_read` is an ordinary bus callback.

### Phase B — Game-start fan-out (Lua)

6. Engine/game-start invokes global **`on_game_start()`** → base **`_g.on_game_start()`**
   (`unpacked/scripts/_g.script`) which calls `axr_main.on_game_start()` (plus base init like PDA
   quick slots).
7. **`axr_main.on_game_start()`** (`axr_main.script`) — enumerates `$game_scripts$` **root**
   `*.script` files, skips an ignore list and zero-byte files, and for every module exposing an
   `on_game_start` function, calls `module.on_game_start()`. Order = filesystem enumeration order.
8. Each module's `on_game_start` typically calls **`RegisterScriptCallback(name, handler)`** to
   subscribe to events. (Per `axr_main` comment: register at/before `on_game_load`.)

### Phase C — Runtime dispatch

9. Engine code or scripts fire an event with **`SendScriptCallback(name,...)`**
   (`_g.script`) → **`axr_main.make_callback(name,...)`** (`axr_main.script`) which calls
   every registered handler **in ascending stored priority** (`spairs` + `sort_func_values_ascend`).
   With no ordering arguments that is registration order (FIFO); `before_funcs`/`after_funcs` and the
   Pre/Normal/Post tier change it (see `callback_set` below). Then it runs `axr_main[name](...)` if `axr_main` itself defines a same-named
   function (built-in handler shortcut).
10. **DXML branch**: engine XML load reaches `dxml_core._G.COnXmlRead`, which fires
    `SendScriptCallback("on_xml_read", xml_file_name, xml_obj, flags)` through `intercepts` like any
    other event (`flows/dxml-xml-interception.md`, `modlayer/dxml.md`).

### Phase D — Teardown

11. `UnregisterScriptCallback(name, handler)` → `axr_main.callback_unset` removes a handler. Unused
    intercept tables are pruned during `on_game_load` (per `axr_main` comment).

## Function digests (L3)

- **`script_engine.cpp:CScriptEngine::init()`** — *Does:* build Lua VM, reinit storage, register
  exports, load `_G` then common scripts. *Calls:* `CScriptStorage::reinit`, `process_file_if_exists("_G")`,
  `load_common_scripts`. *Gotcha:* `_G` loads before common scripts, so `_g.script`'s global defs win
  over earlier kernel-loaded ones.
- **`script_engine.cpp:CScriptEngine::load_common_scripts()`** — *Does:* read `script.ltx [common]
  script` list; `process_file` each; invoke `<name>_initialize()` per module. *Reads:*
  `$game_config$/script.ltx`. *Gotcha:* this is the `_initialize` phase, not `on_game_start`.
- **`_g.script:RegisterScriptCallback(name, f)`** (base) — *Does:* subscribe `f` to `name`. *Calls:*
  `axr_main.callback_set`. *Note:* wrapped by `dxml_core` for `on_xml_read`.
- **`_g.script:SendScriptCallback(name,...)`** (base) — *Does:* fire event. *Calls:*
  `axr_main.make_callback`, then `axr_main[name](...)` if present.
- **`_g.script:AddScriptCallback(name)`** / **`on_game_start()`** — *Does:* declare a new
  callback name (→`callback_add`) / kick the fan-out (→`axr_main.on_game_start`).
- **`axr_main.script:callback_add(name)`** — *Does:* create `intercepts[name]={}`,
  `next_index[name]=1`. *Writes:* `intercepts`, `next_index`. *Gotcha:* warns+callstack if the name
  already exists; a name must be added (here or in the static `intercepts` table) before handlers can
  attach.
- **`axr_main.script:callback_set(name, f, before_funcs, after_funcs, priority)`** — *Does:* registers
  `f` with ordering constraints, then `callback_reorder(name)` recomputes the numeric value stored in
  `intercepts[name][f]` (a stable topological sort per tier; tiers `CALLBACKS_PRE` → `CALLBACKS_NORMAL`
  (default) → `CALLBACKS_POST`). `before_funcs` = handlers `f` must run before, `after_funcs` = handlers
  it must run after; constraints may name handlers not yet registered. Reordering is transactional: a
  cycle or a cross-tier contradiction rolls the registration back with a `!` log line; a malformed
  constraint list or invalid tier is an `assert`. Shorthands
  `RegisterScriptCallbackPre` / `RegisterScriptCallbackPost` (`_g_patches.script`). User guide:
  `docs/axr_main.md`. *Gotcha:* the stored value is a computed priority, not a registration counter —
  `make_callback` still just sorts by it.
- **`axr_main.script:callback_unset(name, f)`** — *Does:* `intercepts[name][f] = nil`.
- **`axr_main.script:make_callback(name,...)`** — *Does:* iterate `intercepts[name]` via
  `spairs(..., sort_func_values_ascend)` and call each handler (`f(...)` for functions; `obj[name](obj,...)`
  for userdata). *Reads:* `intercepts`. *Gotcha:* warns if `name` missing.
- **`axr_main.script:on_game_start()`** — *Does:* scan `$game_scripts$` root `*.script`,
  call each module's `on_game_start()` (skip ignore list + empty files). *Gotcha:* only **root**
  scripts, not subfolders; order is FS enumeration.
- **`dynamic_callbacks.lua:RegisterScriptCallback/SendScriptCallback`** — *Does:* a
  separate narrow bus over a fixed `intercepts = {save, load, update, save_state, load_state}`.
  *Gotcha:* **debug-only and always superseded** — see Subtleties.

## State touched

- `axr_main.intercepts` (name → {handler → index}) and `axr_main.next_index` (name → counter) — the
  main bus registry.
- Globals on `_G`: `RegisterScriptCallback`, `SendScriptCallback`, `AddScriptCallback`,
  `UnregisterScriptCallback` (defined by `_g.script`; `_g_patches.script` adds the ordering arguments
  and `RegisterScriptCallbackPre`/`Post`).

## Threading

Lua runs on the main thread. The MT build adds idle-time parallel Lua GC and *optional* parallel
execution of `CreateTimeEvent`/`AddUniqueCall` (off by default) — see README MT notes. Callback
dispatch itself is synchronous on the calling thread.

## Extension points

This flow *is* the extension mechanism. To add an event: ensure the name exists (static `intercepts`
table in `axr_main.script` or `AddScriptCallback`), then fire it with `SendScriptCallback`; mods
subscribe with `RegisterScriptCallback`. Engine-originated callbacks are fired from C++ (e.g. XML
read, hit, input) — see `engine/xrGame.md` / `engine/engine-core.md`.

## Failure & edge cases

- **Registering to an unknown name** → warning + `callstack()`, handler dropped. The name must be in
  `axr_main.intercepts` (the big static table at `axr_main.script`) or added via `AddScriptCallback`.
- **`nil` handler** → warning, ignored.
- **`on_xml_read` uses `intercepts` now** (since 2026.10.11). Older notes describing a separate
  `dxml_core.xmlCallbacks` list are stale.
- **Module load order** for `on_game_start` is filesystem order of `$game_scripts$` root; don't rely
  on registration order across modules — declare it with `before_funcs`/`after_funcs` or a tier.

## Subtleties

- `dynamic_callbacks.lua` defines global `RegisterScriptCallback`/`SendScriptCallback` over a small
  fixed table, but it is loaded **only in debug mode** (`script_storage.cpp`, inside the
  `isDebugEnabled` block) — in a normal run it never loads. Even in debug, `CScriptEngine::init` loads
  `_G` (`_g.script`) **after** storage reinit, so `_g.script`'s definitions overwrite the kernel ones.
  **Resolved (was flagged uncertain): the live bus is always `_g → axr_main`** — `dynamic_callbacks.lua`
  is either absent (release) or shadowed (debug). Its sole verified consumer is the LuaPanda debugger
  path. Only the engine grep above is needed to re-confirm if `LoadKernelScriptToGlobal` call sites
  change.

## Deeper

- Engine VM/init: `src/xrServerEntities/script_engine.cpp` (`init`, `load_common_scripts`),
  `src/xrServerEntities/script_storage.cpp` (`LoadKernelScriptToGlobal`)
- Bus API (base): `unpacked/scripts/_g.script` (`RegisterScriptCallback`, `SendScriptCallback`,
  `AddScriptCallback`, `on_game_start`, `spairs`)
- Bus registry (repo override): `gamedata/scripts/axr_main.script` (`intercepts` table,
  `callback_add/set/unset`, `make_callback`, `on_game_start`)
- Ordering helper: `gamedata/scripts/_g_patches.script:sort_func_values_ascend`
- XML event: `gamedata/scripts/dxml_core.script` (`COnXmlRead`) → `modlayer/dxml.md`

## Maintenance

- Re-grep the digest anchors if line numbers drift: `function callback_set` etc. in
  `axr_main.script`; `RegisterScriptCallback`/`load_common_scripts` in `script_engine.cpp`.
- If the `dynamic_callbacks.lua` shadowing is ever traced definitively, replace the Subtleties note
  with the verified behavior.
- Keep the `on_xml_read` divert in sync with `modlayer/dxml.md` and the (future) DXML flow file.
- This is the L2/L3 template — mirror its structure for other `flows/*` files.
