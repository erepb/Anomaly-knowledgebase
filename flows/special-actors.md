---
domain: flows/special-actors
tags: [xrgame, ai, trader, crow, phantom, flow]
sources: [src/xrGame/ai/trader/ai_trader.cpp, src/xrGame/ai/trader/ai_trader.h, src/xrGame/ai/trader/trader_animation.cpp, src/xrGame/ai/crow/ai_crow.cpp, src/xrGame/ai/crow/ai_crow.h, src/xrGame/ai/phantom/phantom.cpp, src/xrGame/ai/phantom/phantom.h]
related: [flows/dialog-trade.md, flows/ai-planner.md, flows/monster-ai-control.md, flows/object-lifecycle.md, flows/damage-hit.md, engine/xrGame.md, GLOSSARY.md]
updated: 2026-06-06
verified: code
---

# Flow: special actors (trader / crow / phantom)

Three lightweight "AI" actor types that deliberately **do not** use the stalker GOAP planner
(`flows/ai-planner.md`) or the monster FSM+controls (`flows/monster-ai-control.md`): the **trader**
(stationary dialog/trade NPC), the **crow** (ambient bird), and the **phantom** (psy apparition). Each
is a simple, mostly-scripted or hardcoded special case.

**Provenance:** spine personally code-verified — the class declarations: `CAI_Trader : CEntityAlive,
CInventoryOwner, CScriptEntity, CAI_PhraseDialogManager`; `CAI_Crow : CEntity`; `CPhantom : CEntity`.
The behavior bodies (trader empty Think + animation/sound requests, crow fly/death-fall states, phantom
state machine + psy hit) are confirmed homes but *(agent-traced — not personally re-read)*. The Lua
binders (`bind_trader`/`bind_crow`/`phantom_manager`) are base-game `unpacked/` (read-only reference).

## CAI_Trader — stationary dialog/trade NPC (class verified; bodies agent-traced)

`CAI_Trader : CEntityAlive, CInventoryOwner, CScriptEntity, CAI_PhraseDialogManager` (verified). It is
the merchant you talk to: **no combat or movement AI** — its `Think()` is an empty stub; behavior is
entirely **dialog + trade + animation**, driven by script. `shedule_Update` just runs the inventory
owner + the script-control fork (`flows/script-npc-actions.md`); `UpdateCL` updates sound + the
`CTraderAnimation` (global/head motions). Script sets its animation/sound via the per-object callbacks
**`eTraderGlobalAnimationRequest`/`eTraderHeadAnimationRequest`/`eTraderSoundEnd`** and `set_trader_
global_anim`/`set_trader_head_anim`; the dialog system (`flows/dialog-trade.md`) drives head animation
+ voice via `dialog_sound_start`. It has a head bone-callback to look at the actor. *(bodies
agent-traced.)*

## CAI_Crow — ambient bird (class verified; bodies agent-traced)

`CAI_Crow : CEntity` (verified) — decorative, no real AI. A tiny state machine
(`eFlyIdle`/`eFlyUp`/`eDeathFall`/`eDeathDead`) flies toward a periodically-randomized goal direction
(with mild attraction to corpses) at a fixed speed/min-height; no pathfinding or player awareness. It
sleeps (`processing_deactivate`) until hit; `HitSignal` sets health 0 and `eDeathFall` → on `Die` it
builds a physics shell (`P_build_Shell`) and **ragdoll-falls** (`flows/physics-objects-ragdoll.md`),
becoming an AI-visible lootable corpse. Plays a random idle sound while flying. *(bodies agent-traced.)*

## CPhantom — psy apparition (class verified; bodies agent-traced)

`CPhantom : CEntity` (verified) — a spawned ghost effect (used by the controller / psy mechanics). A
lightweight state machine (`stBirth → stFly → stContact`/`stShoot → stIdle`) **kinematically** steers
toward its target (`m_enemy`, defaults to the current entity; `angle_lerp` + `Position().mad`, no
physics shell), playing per-state particles + sound; on contact it deals a **telepathic hit**
(`PsyHit` → `SHit` of `eHitTypeTelepatic`, `flows/damage-hit.md`) and vanishes (`stIdle` →
`DestroyObject`). Spawned/counted by the Lua `phantom_manager` (legacy + dynamic). *(bodies
agent-traced.)*

## Isolation ledger (the seams)

- **Trader:** dialog/trade (`flows/dialog-trade.md`), inventory (`CInventoryOwner`), animation
  (`CTraderAnimation` via `eTrader*` callbacks), script control (`flows/script-npc-actions.md`). No
  combat/movement AI.
- **Crow:** animation (fly/death cycles), physics ragdoll on death
  (`flows/physics-objects-ragdoll.md`), sound; spatial AI-visibility toggled (invisible flying, visible
  dead). Lua `bind_crow` handles corpse cleanup.
- **Phantom:** particles + sound per state, telepathic damage (`flows/damage-hit.md`), kinematic motion;
  spawned/limited by Lua `phantom_manager`; no physics shell.
- **Lua/config:** `creatures/m_person.ltx` (`[m_trader]` → `AI_TRD_S`, `bind_trader`),
  `creatures/m_crow.ltx` (`[m_crow]` → `AI_CROW`, `bind_crow`), `creatures/m_phantom.ltx`
  (`[m_phantom]` → `AI_PHANT`, `phantom_manager`); trader `set_trader_*_anim`/dialog sound; phantom
  `spawn_dynamic_phantom`/`SetEnemy`. All base-game scripts (`unpacked/`, read-only).

## Threading

Main thread (scheduled/render updates). Crow ragdoll stepped by `xrPhysics`; phantom motion is
kinematic on the main thread.

## Failure & edge cases

- **Don't expect AI from these:** trader has an empty `Think` (dialog-only), crow has no combat/path AI
  (random flight), phantom is a fixed state machine — none uses GOAP or the monster control manager.
- **Crow death = ragdoll + cleanup timer:** a shot crow falls via a physics shell and is removed by the
  Lua binder after a timeout (lootable meanwhile).
- **Phantom is kinematic, contact-detonating:** it ignores physics/cover and flies straight at its
  target, dealing a psy hit on contact then self-destructing; count is Lua-capped.
- **Trader animation is request-driven:** the trader fires `eTrader*AnimationRequest` callbacks asking
  *script* what to play — without a binder driving it, it just idles.

## Deeper

- Trader: `src/xrGame/ai/trader/ai_trader.cpp`/`.h` (`Think` empty, `shedule_Update`, `UpdateCL`,
  `dialog_sound_start`), `trader_animation.cpp` (`set_animation`/`set_head_animation`/`set_sound`,
  `eTrader*` callbacks) — dialog `flows/dialog-trade.md`
- Crow: `src/xrGame/ai/crow/ai_crow.cpp`/`.h` (`ECrowStates`, `shedule_Update`, `state_Flying`,
  `state_DeathFall`, `Die`/`CreateSkeleton`) — ragdoll `flows/physics-objects-ragdoll.md`
- Phantom: `src/xrGame/ai/phantom/phantom.cpp`/`.h` (`EState`, `SwitchToState_internal`, `OnFlyState`,
  `UpdateFlyMedia`, `PsyHit`) — damage `flows/damage-hit.md`
- Contrast: `flows/ai-planner.md` (stalker GOAP), `flows/monster-ai-control.md` (monster FSM)

## Maintenance

- Keep the "these three bypass GOAP/monster-FSM — trader=dialog-only, crow=ambient, phantom=psy state
  machine" verdicts; they are the reason these are separate flows.
- Re-grep the class declarations if a type gains real AI.
- Promote the *(agent-traced)* behavior bodies to verified when a task needs trader animation timing,
  crow flight tuning, or phantom spawn/contact exactness.
