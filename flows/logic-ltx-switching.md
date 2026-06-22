---
domain: flows/logic-ltx-switching
tags: [modlayer, gameplay, unpacked, lua, logic-ltx, condlist, parsing, flow]
sources: [unpacked/scripts/xr_logic.script, unpacked/scripts/utils_data.script]
related: [flows/scheme-engine-core.md, flows/condlist-library.md, flows/dltx-config-load.md, flows/smart-terrain-core.md, flows/db-registry.md, flows/per-smart-logic-configs.md, modlayer/gameplay/scheme-engine.md, modlayer/config-ltx.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: logic-LTX format & scheme switching (F68)

How an NPC's scripted-logic config is **authored** and **parsed** into the in-memory switch list
(`st.logic`) that the scheme engine evaluates. This is the data side of F66: F66 *consumes* `st.logic`
(in `try_switch_to_another_section`) and condlists (in `pick_section_from_condlist`); F68 *produces*
them from the LTX text. The condlist token vocabulary is F67 (`flows/condlist-library.md`); raw LTX/DLTX
loading is F2 (`flows/dltx-config-load.md`).

**Provenance:** the parse spine is **verified** in `xr_logic.script`: `parse_condlist`,
`extract_conditions`, `cfg_get_switch_conditions`, `cfg_get_overrides`, and the
`cfg_get_*_and_condlist` family (existence + the two key bodies read). `utils_data.get_scheme_by_section`
is *(agent-traced — not personally re-read)*. The authoring-format description is reconstructed **from
the parser**, so it is grounded in code (what the parser accepts) rather than from external docs.

## Authoring format (reconstructed from the parser — verified-by-parse)

- **`[logic]` section** — the NPC's master block. `active = <condlist>` selects the initial scheme;
  `custom_data`/`cfg` may point at an external `.ltx` (e.g. `configs/scripts/...`) holding the actual
  sections (the `configure_schemes` seam, F66). Generic-scheme fields here (trade, etc.) are handled by
  `modules.enable_generic_schemes`.
- **Scheme sections** — named `scheme@variant` (e.g. `[walker@guard]`). The prefix before `@` is the
  scheme; `utils_data.get_scheme_by_section` extracts it. Inside: scheme-specific fields (paths, etc.)
  plus the **`on_*` switch fields** below.
- **`on_*` switch fields (verified set, from `cfg_get_switch_conditions`):** `on_actor_dist_le`,
  `on_actor_dist_le_nvis`, `on_actor_dist_ge`, `on_actor_dist_ge_nvis`, `on_signal`, `on_first_update`,
  `on_info`, `on_timer`, `on_game_timer`, `on_actor_in_zone`, `on_actor_not_in_zone`, `on_actor_inside`,
  `on_actor_outside`, `on_npc_in_zone`, `on_npc_not_in_zone`. Trailing digits are stripped
  (`on_info2` → `on_info`) so a field can repeat.
- **condlist syntax (verified, from `parse_condlist`/`extract_conditions`):**
  `{checks} target_section %effects%, {checks2} section2, … fallback_section`. Comma separates
  alternatives (first match wins; a trailing bare section is the fallback). Tokens inside `{}` and `%%`:
  `+info`/`-info` (infoportion), `=func`/`!func` (xr_conditions, optionally `func(p1:p2)` params),
  `~NN` (NN% chance). Effects in `%%` use the same token forms against xr_effects (F67).

## Parse spine (verified)

- **`xr_logic.script:parse_condlist(npc, section, field, src)`** — memoized (`memoize_condlist[src]`).
  Splits `src` on commas; per alternative, pulls the `{...}` group into the **check list** (index 2 via
  `extract_conditions`) and the `%...%` group into the **effect list** (index 3), and sets the trimmed
  remainder as the **target section** `lst[n][1]`. Returns the nested table condlists are stored as.
  *(verified.)*
- **`xr_logic.script:extract_conditions(c, lst, n, index)`** — `gmatch`es `(sign)(name)` tokens and
  appends `{name, type, extra}`: `+`→1 (+info, extra = `npcx_` flag), `-`→2, `=`→3 / `!`→4 (extra =
  params table, each `:`-split param `tonumber`-or-string), `~`→5 (extra = number). Unknown sign →
  `abort_syntax_error_in_cond`. This produces exactly the token shape F67's evaluator reads. *(verified.)*
- **`cfg_get_*_and_condlist` family** — `cfg_get_condlist` (field → condlist), `cfg_get_number_and_condlist`
  (`"N, condlist"` → `.v1`+condlist, for dist/timer), `cfg_get_string_and_condlist` (signal/zone name),
  `cfg_get_npc_and_zone` (npc story-id + zone). Each wraps `parse_condlist`. *(family verified by
  existence + the number/string variants' use in the router.)*
- **`xr_logic.script:cfg_get_switch_conditions(ini, section, npc)`** — iterates the section's lines,
  strips trailing digits off each id, and routes each `on_*` id to the right `cfg_get_*` helper,
  building the `st.logic` array (`{name=..., v1/v2=..., npc_id=..., condlist=...}` per entry). Returns
  `{}` if the section doesn't exist. This array is exactly what `try_switch_to_another_section` walks
  (F66). *(verified.)*
- **`xr_logic.script:cfg_get_overrides(ini, section, npc, l)`** — parses the combat/offline override set
  (`combat_ignore_cond`, `enemy_ignore_cond`, `no_combat_job`, `combat_ignore_keep_when_attacked`, …)
  into `db.storage[id].overrides` (reuses table `l` to avoid GC). *(verified head.)*

## Function digests (spine)

- **`parse_condlist`** — *Does:* string→nested condlist (memoized). *Calls:* `extract_conditions`.
  *Returns:* `lst` of `{section, checks, effects}`. *(verified.)*
- **`extract_conditions`** — *Does:* tokenize `{}`/`%%` into `{name,type,extra}` with the F67 type codes.
  *(verified.)*
- **`cfg_get_switch_conditions`** — *Does:* build `st.logic` by routing `on_*` lines. *Reads:* the ini
  section lines. *(verified.)*
- **`cfg_get_overrides`** — *Does:* build `overrides` (combat/offline). *(verified head.)*
- **`utils_data.get_scheme_by_section`** — *Does:* `"walker@guard"` → `"walker"` (gsub the `@…`).
  *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned output:** `st.logic` (switch list) and `st.overrides` on `db.storage[id]`; the module-local
  `memoize_condlist` cache.
- **Producer (in):** schemes call `cfg_get_switch_conditions` from their `set_scheme` (F66), passing the
  ini + section; `configure_schemes` resolves which ini (custom_data/cfg) — the DLTX/ini seam (F2).
- **Consumer (out):** `try_switch_to_another_section` (walks `st.logic`) and `pick_section_from_condlist`
  (evaluates the condlists) — both F66/F67.
- **Token vocab seam:** `=func`/`%=func%` names are resolved late against `xr_conditions`/`xr_effects`
  (F67), not at parse time — a typo parses fine and only warns at eval.

## Threading / failure / edge cases

- **Memoization is by source string:** identical condlist strings share one parsed table globally
  (`memoize_condlist`). Don't mutate a returned condlist table in place — it's shared.
- **Syntax errors abort:** a malformed `{}`/`%%` group calls `abort_syntax_error_in_cond` (hard error),
  so bad logic LTX fails loudly at parse — distinct from F67's *silent* unknown-functor at eval.
- **Trailing-digit fields:** `on_info`, `on_info2`, `on_info3` all become `on_info` entries — repetition
  is intentional; order in the section is the evaluation order in `st.logic`.

## Deeper

- Parser: `unpacked/scripts/xr_logic.script` (`parse_condlist`, `extract_conditions`,
  `cfg_get_switch_conditions`, `cfg_get_*_and_condlist`, `cfg_get_overrides`)
- Section→scheme: `unpacked/scripts/utils_data.script` (`get_scheme_by_section`)
- Consumers: `flows/scheme-engine-core.md` (`try_switch_to_another_section`); evaluator + token vocab:
  `flows/condlist-library.md`; raw ini/DLTX loading: `flows/dltx-config-load.md`

## Maintenance

- The parse spine is verified and stable; re-read `extract_conditions` + `cfg_get_switch_conditions` if
  a new `on_*` trigger or token sign appears (and mirror it in F66's trigger taxonomy + F67's type codes).
- Promote `utils_data.get_scheme_by_section` and the remaining `cfg_get_*` variant bodies to verified on
  next read.
- Check `gamedata/scripts/xr_logic.script` for an override before trusting (none at time of writing).
