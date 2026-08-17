---
domain: meta
tags: [meta, methodology, flows, lifecycle]
sources: [src/, gamedata/, unpacked/]
related: [INDEX.md, CONVENTIONS.md, flows/lua-callback-lifecycle.md]
updated: 2026-08-17
---

# Methodology — how this codebase is analyzed & divided

This file defines *how* the KB goes from "where things live" to "how things run": the granularity
layers, the procedure for cutting the codebase into isolated domains, and the representation formats
for flows and function digests. Read this before adding depth (flows / digests) to the KB.

## Granularity layers

The KB is built in four layers, shallow→deep. A reader descends only as far as the task needs.

- **L0 — Glossary** (`GLOSSARY.md`): vocabulary/concepts. Answers "what does this term mean".
- **L1 — Domain map** (`engine/*`, `modlayer/*`): module responsibilities and where code lives.
  Answers "which file/dir owns X".
- **L2 — Flows & lifecycles** (`flows/*`): ordered cross-cutting sequences (boot, frame, save,
  callback dispatch, config load…). Answers "what happens, in what order, and where each step lives".
- **L3 — Function digests** (inside flow/domain files): per-key-function condensed "what it does".
  Answers "what does this function actually do, what does it touch, what calls it".

L3 is **selective**: only spine / hot-path / repo-specific / non-obvious functions get a digest.
Documenting every function in a 2686-file module is neither feasible nor useful; document the ones a
future task will need to understand or modify.

## Dividing into isolated domains

A domain is *well-isolated* if you can state its **inputs, outputs, owned state, and lifecycle**
using only its seams — without needing another domain's internals. Procedure:

1. **Seed from entry points & ownership.** Find the process/subsystem entry (engine: `WinMain`→
   device; Lua: `CScriptEngine` load→`on_game_start`) and the state each subsystem *owns*. Ownership
   of state is the strongest domain boundary.
2. **Cut along lifecycle phases.** Most subsystems split into **init / per-frame update / shutdown**,
   and several have orthogonal axes: **online vs offline** (ALife), **load vs save**, **main thread
   vs worker** (MT build). Each phase is often its own flow.
3. **Treat extension seams as contracts.** The Lua callback bus, DLTX/DXML hooks, and config params
   are the engine↔mod-layer interface. Document the seam from *both* sides (who fires / who handles).
4. **Apply the base+override rule (Lua).** The analyzable Lua universe is two corpora: the
   **read-only base game** (`unpacked/scripts/`, ~455 files incl. `_g.script`, `xr_logic.script`,
   `db.script`, `bind_*`) and this repo's **overrides + additions** (`gamedata/scripts/`, ~66 files,
   mostly same-named overrides). Document the *canonical mechanism* once (citing `unpacked/`), then
   the *modded delta* (citing `gamedata/`). Never edit `unpacked/` — it is reference only.
5. **Isolation test.** If describing domain A keeps dragging in domain B's internals, the cut is
   wrong — find the real seam (an interface, an exported function, a callback, a config key) and put
   the boundary there.

### Engine vs mod-layer analyzability

- **Engine (`src/`)**: fully self-contained C++; flows traceable end-to-end by reading the code.
- **Mod layer (`gamedata/`)**: depends on the base corpus in `unpacked/`. A mod-layer flow usually
  must read base scripts (for the mechanism) + repo overrides (for the behavior). Cite both.

## Representation formats

### Flow (L2) — file under `flows/`

Each flow file documents one sequence. Sections:

- **Trigger / entry** — what starts it (engine event, frame tick, user action, save).
- **Sequence** — numbered steps. Each step: the responsible `file:function` (or module) + a
  one-line "what it does". Show branches/loops inline.
- **State touched** — what is read/written/owned across the flow.
- **Threading** — main thread / worker / both (note MT-build differences).
- **Extension points** — callbacks fired, config keys read, hooks exposed to mods.
- **Failure & edge cases** — known crashes, ordering hazards, fatals.
- **Deeper** — pointers; **Maintenance** — how to re-verify.

### Function digest (L3)

One compact entry per function (inside the relevant flow/domain file):

- **`file:function(args) → ret`** — *Does:* one/two lines. *Reads/Writes:* state/globals. *Calls:*
  key callees. *Called by / fires:* callers or callbacks. *Gotcha:* invariant or hazard (optional).

Keep it behavioral ("what it does"), not a code transcription. No pasted code (see `CONVENTIONS.md`).

### Spine-function rule (which functions to digest)

Documenting every function in a large module is infeasible. For each domain, digest only the
**lifecycle spine** — the functions on the critical path: constructor / `net_Spawn` (or equivalent
init), the per-frame update entry/entries, each state-transition function, and teardown
(`net_Destroy`). A leaf/helper gets a digest *only if non-obvious or repo-specific*. This bounds
"read the whole module" down to the ~6–12 functions that define how the domain behaves. Everything
else stays an L1 pointer until a task needs it.

### Isolation ledger (per domain/flow)

Every domain/flow file ends its body with an explicit **seam list** — the contract that makes the
domain analyzable in isolation and is what future tasks actually need:

- **Callbacks fired** (exact engine/Lua callback names).
- **Config keys read** (LTX/`pSettings`/spawn_ini).
- **Scheduler interaction** (RT vs deferred; how it registers).
- **ALife / server seam** (CSE_* link, online/offline).
- **Lua exposure** (wrapper class, binder, exported surface).
- **Other domains touched.**

`flows/object-lifecycle.md` is the worked template for both rules.

### Provenance of agent-assisted research

When a read-only sub-agent gathers the raw trace (to save context on a large module), its claims are
a **draft, not verified**. Spot-verify the spine entries against code yourself before marking a file
`verified`; mark anything you did not personally re-read *(agent-traced — not personally re-read)*,
distinct from *(INFERRED)* (which means no one read the code). See `flows/object-lifecycle.md`.

## Rollout backlog (flows to document, prioritized)

> The detailed coverage plan — full flow inventory (F1–F20), phasing, per-flow execution recipe, and
> resume-after-compaction protocol — lives in **`ROADMAP.md`**. This list is the short status mirror;
> keep both in sync.

High value = repo-specific, crash-prone, or frequently touched. Status as of `updated` — statuses
re-checked against the tree on 2026-08-17 (all ten are now documented; `ROADMAP.md` remains the detailed
plan, and `INDEX.md`'s "Branch sync" section records what was reconciled after a code sync).

1. **Lua boot + callback lifecycle** — ✅ done → `flows/lua-callback-lifecycle.md` (flagship/template).
2. **Config load: LTX parse + DLTX merge** — ✅ done → `flows/dltx-config-load.md`.
3. **DXML XML interception** — ✅ done → `flows/dxml-xml-interception.md`.
4. **Engine boot → device init → main/frame loop** — ✅ done → `flows/engine-boot-frame.md`.
5. **Render frame (R4/DX11)**: cull → g-buffer → lighting → post → present — ✅ done →
   `flows/render-frame.md` (+ the `flows/renderer-*.md` family).
6. **Save / load lifecycle** — ✅ done → `flows/save-load.md`.
7. **ALife online↔offline transition** — ✅ done → `flows/alife-online-offline.md`.
8. **Weapon fire → hit → callback chain** — ✅ done → `flows/weapon-fire-callbacks.md`.
9. **Options framework lifecycle** — ✅ done → `flows/options-runtime.md`.
10. **Object lifecycle (spawn/update/destroy) + scheduler split** — ✅ done →
    `flows/object-lifecycle.md` (the xrGame backbone; per-domain flows attach to its spine).

When you complete a flow, flip its status here and add the file to `INDEX.md`.

## Maintenance

- Keep the backlog honest: mark done only when the flow file reaches L2 with L3 digests for its
  spine functions and is verified against the code this session.
- If a new analysis axis appears (e.g. Vulkan path, networking model change), add it to "cut along
  lifecycle phases".
