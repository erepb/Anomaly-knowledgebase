---
domain: flows/condlist-library
tags: [modlayer, gameplay, unpacked, lua, condlist, conditions, effects, flow]
sources: [unpacked/scripts/xr_conditions.script, unpacked/scripts/xr_effects.script, unpacked/scripts/xr_conditions_addon.script, unpacked/scripts/xr_effects_addon.script, unpacked/scripts/xr_logic.script]
related: [flows/scheme-engine-core.md, flows/logic-ltx-switching.md, flows/relations-info.md, flows/dialog-trade.md, flows/db-registry.md, flows/utils-libraries.md, modlayer/gameplay/scheme-engine.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: condlist library — xr_conditions / xr_effects (F67)

The predicate + action vocabulary that logic LTX is written in. A condlist token like `=is_alive`,
`!has_enemy`, or `%=give_info(x)%` resolves at evaluation time to a function in `xr_conditions`
(returns bool) or `xr_effects` (side effect). These two namespaces are the open-ended library every
scheme, dialog, and task condlist draws on. The *parser* that turns strings into the token tables is
F68 (`flows/logic-ltx-switching.md`); the *evaluator* that calls these functions lives in `xr_logic`
(verified here).

**Provenance:** the registry mechanism, the call signature, and the full evaluator call sites are
**verified** against `xr_logic.script:pick_section_from_condlist` and the head of `xr_conditions.script`
(sample functions read: `is_actor_enemy_to_faction`, `is_npc_enemy_to_faction`, `toolkit_task_done`).
The two files are huge (`xr_conditions` and `xr_effects` are thousands of lines); the individual
function catalog is **sampled, not exhaustive** — the named examples beyond those three and the
`*_addon` dispatcher are *(agent-traced — not personally re-read)*. No `gamedata/` override of the core
two files (agent-checked).

## Registry mechanism (verified)

- A condition is a **bare global function** in `xr_conditions.script`, e.g.
  `function is_actor_enemy_to_faction(actor, npc, p) ... end`. X-Ray loads each `.script` as a
  **namespace table named after the file**, so that function becomes `xr_conditions.is_actor_enemy_to_faction`
  (verified: grep finds no `function xr_conditions.NAME` — they are written bare and namespaced by the
  loader, and `xr_logic` calls them as `xr_conditions[name]`). Effects work the same way in
  `xr_effects.script`.
- **Universal signature (verified):** `xr_conditions[name](actor, npc, p)` → boolean;
  `xr_effects[name](actor, npc, p)` → side effect (no return used). `actor` is the eval's `obj` (usually
  `db.actor`), `npc` is the logic owner, `p` is the parsed params table (or `nil`).
- **No explicit registration / init** — defining the function is registering it. Adding a new condition
  = add a global function to `xr_conditions.script` (or the addon).

## Evaluation (verified — in xr_logic)

`xr_logic.script:pick_section_from_condlist(obj, npc, condlist)` consumes the parsed tokens (F68). For
each condlist entry, it walks the **check list** `cond[2]`, where each token is `{name, type, extra}`:
- **type 1** `+info` — required: `has_info(name)` (npc-target if `name` has the `npcx_` flag, else actor,
  else `sim:has_info(0, name)`).
- **type 2** `-info` — required absent.
- **type 3** `=func` — `xr_conditions[name](obj, npc, extra)` must be **true** (extra = params table). If
  `xr_conditions[name]` is missing it prints a `xr_conditions fail!` warning.
- **type 4** `!func` — same call must be **false**.
- **type 5** `~prob` — fails when `extra < math.random(100)`.

If every check passes, it applies the **effect list** `cond[3]` (type 1/2 = give/disable info portion;
type 3/4 = `xr_effects[name](obj, npc, extra)`) and returns the entry's section `cond[1]`. First passing
entry wins; otherwise returns nil. This is the single seam where condlist functions are actually invoked.

## Representative functions (sample — verified ones flagged)

- `xr_conditions.script:is_actor_enemy_to_faction(actor, npc, p)` — *Does:* `game_relations.is_factions_enemies(character_community(db.actor), p[1])`. *(verified.)*
- `xr_conditions.script:is_npc_enemy_to_faction(actor, npc, p)` — *Does:* same for `npc`'s community. *(verified.)*
- `xr_conditions.script:toolkit_task_done(actor, npc, p)` — *Does:* checks an `_upgrade_tier_<n>` alife info on the npc section. *(verified.)*
- Common others *(agent-traced):* alive/health/distance/enemy/wounded checks (e.g. `actor_alive`,
  `health_le`, `dist_to_actor_le`, `has_enemy`, `see_enemy`, `is_wounded`, `has_item`).
- Common effects *(agent-traced):* `give_info`/`disable_info` (handled inline by the evaluator for
  bare `+/-info`), plus functional ones like `play_sound`, `spawn_object`, `teleport_actor`,
  `reward_money`, `set_task_stage`, `inc_goodwill`.

## Addon dispatchers *(agent-traced)*

`xr_conditions_addon.script` / `xr_effects_addon.script` add helpers and notably a generic
`script(actor, npc, p)` dispatcher: `{=script(module:function:args)}` / `%=script(module:function:args)%`
calls `_G[module][function](...)` — the escape hatch to call arbitrary Lua from a condlist. *(Not
personally read; verify before relying on exact arg handling.)*

## Function digests (spine)

- **`xr_logic.script:pick_section_from_condlist`** — *Does:* the verified evaluator above; the only place
  `xr_conditions[]`/`xr_effects[]` are called. *(verified; full body across F67/F68.)*
- **`xr_conditions.script` (namespace)** — *Does:* boolean predicates `(actor, npc, p)`. *(mechanism +
  3 samples verified; catalog agent-traced.)*
- **`xr_effects.script` (namespace)** — *Does:* side-effect actions `(actor, npc, p)`. *(mechanism
  verified via the eval call site; bodies agent-traced.)*

## Isolation ledger (the seams)

- **Owned state:** none of its own — conditions/effects read/write *game* state (db, infoportions,
  inventory, relations, tasks) and the alife sim.
- **Caller (in):** only `xr_logic.pick_section_from_condlist` (for logic switching, F66) and the dialog
  system (its own condlist eval; `flows/dialog-trade.md`). Tokens are produced by the F68 parser.
- **Out:** into managers — `game_relations` (`flows/relations-info.md`), `task_manager`
  (`flows/tasks.md`), `xr_sound`, inventory/alife, `xr_wounded`, etc.
- **Info-portion seam:** bare `+/-info` is handled directly by the evaluator (not via a function); only
  `=func`/`!func`/`%=func%` hit these namespaces.

## Threading / failure / edge cases

- **Missing functor is non-fatal:** an unknown `=func` logs `xr_conditions fail!` and the check is
  skipped (treated as not-failing for that token path) — typo'd condition names silently misbehave
  rather than crash. Watch for this when logic "doesn't switch."
- **Params are strings unless numeric:** `extract_conditions` (F68) `tonumber`-or-keeps each `:`-split
  param, so a function receives mixed string/number params — read params defensively.
- **`script()` dispatcher** can call anything global; a wrong module/function name throws at call time.

## Deeper

- Conditions: `unpacked/scripts/xr_conditions.script` (bare global predicates)
- Effects: `unpacked/scripts/xr_effects.script` (bare global actions)
- Addons: `unpacked/scripts/xr_conditions_addon.script`, `unpacked/scripts/xr_effects_addon.script`
  (`script` dispatcher)
- Evaluator + parser: `unpacked/scripts/xr_logic.script` (`pick_section_from_condlist`); F68
  `flows/logic-ltx-switching.md` for token production

## Maintenance

- The verified core (registry-by-namespace + `(actor,npc,p)` signature + the evaluator type codes) is
  stable; re-read `pick_section_from_condlist` if condlist semantics change.
- The function *catalog* is intentionally sampled — when a task needs a specific condition/effect, read
  that one function and cite it `file:name` (promote from agent-traced).
- Re-check `gamedata/scripts/` for overrides of `xr_conditions`/`xr_effects`/the addons before relying.
