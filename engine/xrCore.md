---
domain: engine/xrCore
tags: [engine, cpp, dltx, ltx]
sources: [src/xrCore/Xr_ini.cpp, src/xrCore/xrstring.h, src/xrCore/LocatorAPI.h, src/xrCore/LocatorAPI.cpp, src/xrCore/LocatorAPI_defs.cpp, src/xrCore/LocatorAPI_defs.h]
related: [engine/_overview.md, modlayer/config-ltx.md, flows/dltx-config-load.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-08-17
verified: code
---

# xrCore

Foundation library every other module depends on: strings, fast casts, file system, config parsing
(including the DLTX merge), math, debug/log.

## Key pieces

- **shared_str** (`xrstring.h`) — interned, ref-counted, pooled strings. Verified: holds
  `intrusive_ptr<str_container>`; `str_container` is an intrusive-refcounted (non-atomic)
  type and values persist in a global intern pool `g_pStringContainer`; raw pooling
  via `pool_block`/`alloc_in_pool`. **Interning** (`str_container::dock`, `xrstring.cpp`):
  hash via `xr_hash` (= `robin_hood::hash`, verified `_stl_extensions.h`) → slot `hash %
  buffer_size` in a **custom bucket table** (`buffer[slot]` forward-lists), guarded by an SRW lock
  (`xrSRWLockGuard` — shared for lookup, exclusive + double-check for insert: the MT-safe rework).
  Note: the table is custom, not a `robin_hood::unordered_map`; only the hash function is robin_hood.
  See `GLOSSARY.md#shared_str`.
- **Containers** — `xr_unordered_map`/`xr_unordered_set` and the flat variants are
  `robin_hood::unordered_*` (Robin-Hood-hashing maps), aliased in `_stl_extensions.h`.
- **smart_cast** — NOT in xrCore. Defined in `src/xrServerEntities/smart_cast.h`: template
  `has_dcast` specializations (e.g. `IRenderVisual`→`IKinematics` via `dcast_PKinematics`) with a
  fallback to the `fast_dynamic_cast` library. Verified by reading the header. (Listed here only as a
  pointer; see `engine/server-entities.md`.)
- **File system** — `FS` is a `CLocatorAPI` (`LocatorAPI.h`, global `xr_FS`). Verified:
  alias-based virtual path roots via `append_path(path_alias, root, add, …)` and `update_path(...)`;
  the `$game_config$`/`$game_scripts$`/… aliases are resolved here (parse loop in
  `LocatorAPI.cpp`). `FS.file_list(...)` is used by DLTX to find `mod_*` files. Readers:
  `file_stream_reader.*`, `stream_reader.*`.
  **`FS_Path` owns raw `char*` members** (`LocatorAPI_defs.cpp`) and its constructor was hardened
  (2026-07, "possible crash fix on level change"): all five pointers are nulled first, null `_Root`/`_Add`
  are treated as empty strings, duplication goes through a local `xr_strdup_lwr`, and the trailing
  separator is appended by `append_path_separator`. The matching call-site bug was
  `new FS_Path(xr_strdup(...))` in `CLocatorAPI::_initialize` for `$fs_root$` — the ctor duplicates its
  argument, so the extra `xr_strdup` leaked and the temporary it was built from could already be gone.
  When adding a path alias, pass a plain `const char*`; the `FS_Path` copies it.
- **LTX/ini + DLTX** (`Xr_ini.cpp`, `xr_ini.h`) — parses `.ltx` config into sections, handles
  `#include`, section inheritance, and the **DLTX delta-merge**. This is where `mod_<file>_<name>.ltx`
  patches are discovered and applied. Markers to grep: `DLTX`, `DLTX_DELETE`, `dltx_use_cache`,
  `print_dltx_warnings`, `FS.file_list(... "mod_"...)`, the duplicate-section fatal. Config-format
  details and operators: `modlayer/config-ltx.md` and `GLOSSARY.md#dltx`.
- **String/util** — `_std_extensions.*`, `string_concatenations.*`, `mezz_stringbuffer.*`
  (MerelyMezz, used by DLTX), `dump_string.*`.
- **Compression/init** — `rt_lzo_init.cpp`, LZO for archives.

## DLTX flow (high level)

On loading a base `.ltx`, the parser also collects matching `mod_<name>_*.ltx` files from the same
virtual path and merges their sections **key-by-key onto the base** (not whole-section replace) per
the DLTX operators (`!`/`@` = override fragment, `@` also creates if missing; `>`/`<` list
add/remove; `DLTX_DELETE` delete key). Results are cacheable (`dltx_use_cache`). Duplicate non-override
sections raise a fatal that names the offending files. Full flow: `flows/dltx-config-load.md`.

## Deeper

- Config parse + DLTX: `src/xrCore/Xr_ini.cpp`, `src/xrCore/xr_ini.h`
- Strings: `src/xrCore/xrstring.cpp/.h`, `src/xrCore/string_concatenations*`
- FS: `src/xrCore/LocatorAPI.h/.cpp` (`CLocatorAPI`, `append_path`/`update_path`), readers
  `src/xrCore/stream_reader.*`, `src/xrCore/file_stream_reader.*`
- DLTX helper buffer: `src/xrCore/mezz_stringbuffer.*`

## Maintenance

- DLTX behavior is the most consequential thing here — if `Xr_ini.cpp` changes, re-read the DLTX
  section and sync operator descriptions with `modlayer/config-ltx.md` and `GLOSSARY.md#dltx`.
- Verify the grep markers above still match (`print_dltx_warnings`, `dltx_use_cache`,
  `DLTX_DELETE`).
- If `shared_str`/`smart_cast` internals are reworked again (MT merges), update the one-liners.
