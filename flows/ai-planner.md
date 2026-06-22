---
domain: flows/ai-planner
tags: [engine, cpp, ai, goap, planner, flow, lua, alife]
sources: [src/xrGame/ai/stalker/ai_stalker.cpp, src/xrGame/stalker_planner.cpp, src/xrGame/action_planner.h, src/xrGame/action_planner_inline.h, src/xrGame/problem_solver.h, src/xrGame/problem_solver_inline.h, src/xrGame/action_base.h, src/xrGame/property_evaluator.h, src/xrGame/stalker_decision_space.h, src/xrGame/script_entity.cpp]
related: [flows/object-lifecycle.md, flows/lua-callback-lifecycle.md, engine/xrGame.md, engine/server-entities.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Flow: AI think loop / GOAP planner

How an NPC decides what to do each update: the per-frame think entry, the **GOAP** (goal-oriented
action planning) cycle that selects an action by A*-searching action space against a goal world
state, and the **seam to the Lua logic layer** (`xr_logic` schemes) that can take control instead.
This is the spine/mechanism — not a catalog of every action class (there are hundreds; they are
leaves on this spine).

**Provenance:** researched with a read-only agent, then spot-verified against code. The think entry
(`CAI_Stalker::Think` → `brain().update()`), the script-control seam (`GetScriptControl` →
`ProcessScripts`), and the `StalkerDecisionSpace::eWorldProperty*`/`eWorldOperator*` enums are
personally code-verified. The A* solver internals (`CProblemSolver::solve`/`value`/`is_goal_reached`)
and the action/evaluator virtuals are *(agent-traced — not personally re-read)*; the agent itself
marked the action lifecycle and evaluators SKIMMED. The base Lua logic (`xr_logic.script`,
`bind_stalker.script`) lives in the read-only base corpus (`unpacked/`) and is cited, not re-read
here.

## Where the think runs (verified)

NPCs (`CAI_Stalker`, monsters under `ai/monsters/`) update on the **deferred** scheduler channel
(`shedule_Update`, the backbone — `flows/object-lifecycle.md`). For a stalker, `shedule_Update`
reaches **`CAI_Stalker::Think()`** (verified, `ai_stalker.cpp`), which calls **`brain().update(dt)`**
(verified) — the planner — and the movement manager update. The same `brain().update()` entry recurs
across humans, monsters, and offline ALife abstracts (verified call sites). The brain for a stalker
is a `CStalkerPlanner` (`stalker_planner.cpp`), a `CActionPlanner` specialization.

## The script-control fork (verified)

Before GOAP runs, the per-frame path forks on **`GetScriptControl()`** (verified, `script_entity.cpp`
+ the decision site in `ai_stalker.cpp`): if a script has taken control, **`ProcessScripts()`** runs
the queued Lua `CScriptEntityAction`s (animation/movement/sound subactions) **instead of** the GOAP
planner. The Lua logic layer (`xr_logic.script` in `unpacked/`) toggles this flag per active scheme,
so high-level behavior is script-driven while GOAP handles the low-level execution it delegates to.
This is the engine↔mod AI seam: the C++ planner and the Lua action queue are two modes selected by
one flag.

## The GOAP decision cycle (agent-traced solver; enums verified)

The world model is fixed enums in `stalker_decision_space.h` (verified):
`StalkerDecisionSpace::eWorldProperty*` (boolean sensors — alive, enemy, danger, anomaly, items, …)
and `eWorldOperator*` (actions/sub-planners — e.g. `eWorldOperatorCombatPlanner`,
`eWorldOperatorDeathPlanner`). A `CStalkerPlanner` registers evaluators (one per property) and
operators (actions), and a target world state, at setup.

Each `brain().update(dt)` (cycle structure agent-traced from `problem_solver_inline.h` /
`action_planner_inline.h`):

1. **Evaluate world state** — A* lazily calls each needed property's `CPropertyEvaluator::evaluate()`,
   which queries perception/memory (selected enemy, visibility, danger, nearby items) and condition;
   results cache into the current world state.
2. **Compare to goal** — `is_goal_reached` tests the current state against the target state.
3. **Solve** — `CProblemSolver::solve` runs A* over operator space: each action is an edge whose
   **preconditions** gate applicability and whose **effects** transform the state; `get_edge_weight`
   uses the action's `weight()`. The result is an ordered action path (`m_solution`).
4. **Run the front action** — `CActionPlanner::update` transitions the lifecycle: `finalize()` the
   old action if the solution changed, `initialize()` the new front action, then `execute()` it every
   frame. The action drives movement goals / weapon / animation; its effects change world state, so
   next tick the plan re-solves.

This is the loop: **evaluate → goal-test → A* search by precondition/effect → execute current action
→ effects mutate state → repeat.** Sub-planners (combat, danger, death…) are themselves operators,
giving a hierarchy of GOAP planners rather than one flat action list.

## Isolation ledger (the seams)

- **Scheduling:** driven on the **deferred** channel (`shedule_Update` → `Think` → `brain().update`).
  Movement/animation side-effects apply via the manager updates in the same tick. (Serial scheduler —
  `engine/engine-core.md`.)
- **Config / data read:** character/logic `.ltx` (loaded at `CAI_Stalker::Load`/`net_Spawn`); the
  object's `spawn_ini` logic section name selects the scheme; smart-terrain/task data via the Lua
  layer. *(specific keys agent-traced.)*
- **Lua seam (the AI contract):** `GetScriptControl()` switches between GOAP and the Lua action queue
  (`ProcessScripts`). The base `xr_logic.script` / `bind_stalker.script` (`unpacked/`) implement the
  scheme system that sets control and pushes actions — document the *mechanism* here, the *base
  scripts* in the mod-layer corpus (`modlayer/scripting.md`, `flows/lua-callback-lifecycle.md`).
  Decision-space enums are also exported to Lua (`ai_stalker_script.cpp`: `property_alive`,
  `action_combat_planner`, …) so scripts can reference planner states.
- **ALife / server seam:** **online** = full GOAP on the local client (this flow); **offline** = the
  ALife simulator runs a reduced brain (`alife_*_brain` — `engine/server-entities.md`, and the
  verified offline monster brain); **remote** (network slave) = no local AI, just interpolation. The
  same `brain().update()` symbol appears in the offline abstracts.
- **Perception/memory feeds the world state:** evaluators read the memory managers (visual/sound/
  danger, selected enemy) — perception is its own pending flow (F11) that this one consumes.
- **Other domains:** movement manager (pathfinding/cover), object handler (weapon selection/fire →
  `flows/weapon-state-machine.md`), animation manager, sound.

## Threading

Main thread (deferred scheduler channel, serial).

## Failure & edge cases

- **`GetScriptControl()` is the master switch:** if a scheme took control, GOAP is *not* running —
  debugging "the planner won't pick action X" often means the NPC is script-controlled, not GOAP.
- **GOAP re-solves from world state, not a fixed script:** an action persists only while it remains
  the A* solution front; a changed sensor (enemy seen, health dropped) re-plans mid-action.
- **Sub-planners are operators:** combat/danger/etc. are nested GOAP planners, so a single
  `current_action_id()` can be a planner, not a leaf action (`ai_stalker_fire.cpp` checks
  `eWorldOperatorCombatPlanner`).
- **Offline ≠ online AI:** an offline NPC uses the ALife brain, not this planner — behavior differs
  by switch state (`flows/object-lifecycle.md` ALife seam).

## Deeper

- Think entry: `src/xrGame/ai/stalker/ai_stalker.cpp` (`Think`, `shedule_Update`, the
  `GetScriptControl` fork)
- Planner: `src/xrGame/stalker_planner.cpp` (`CStalkerPlanner` setup: evaluators + actions),
  `src/xrGame/action_planner.h` + `action_planner_inline.h` (`CActionPlanner::update`, lifecycle)
- A* solver: `src/xrGame/problem_solver.h` + `problem_solver_inline.h` (`solve`, `value`,
  `is_goal_reached`, `evaluate_condition`)
- Actions/evaluators: `src/xrGame/action_base.h` (`initialize`/`execute`/`finalize`),
  `src/xrGame/property_evaluator.h`
- World model: `src/xrGame/stalker_decision_space.h` (`StalkerDecisionSpace::eWorldProperty*`,
  `eWorldOperator*`)
- Script seam: `src/xrGame/script_entity.cpp` (`GetScriptControl`, `ProcessScripts`); base Lua in
  `unpacked/scripts/xr_logic.script`, `unpacked/scripts/bind_stalker.script`

## Maintenance

- Re-grep `CAI_Stalker::Think`/`brain().update`/`GetScriptControl` and the
  `StalkerDecisionSpace::eWorldOperator*` enums if the AI dispatch changes.
- Promote the *(agent-traced)* solver internals (`CProblemSolver::solve`/`value`/`is_goal_reached`)
  and the action/evaluator virtuals to verified by reading `problem_solver_inline.h` /
  `action_planner_inline.h` when a task needs planner detail.
- This is the F10 anchor; F11 (perception/memory) feeds its evaluators and F12 (squad/smart_terrain ↔
  `xr_logic`) builds on the script-control seam — reference this flow rather than re-describing GOAP.
