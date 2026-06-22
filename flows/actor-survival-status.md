---
domain: flows/actor-survival-status
tags: [modlayer, gameplay, unpacked, lua, actor, survival, sleep, thirst, psy, boosters, hud, flow]
sources: [unpacked/scripts/actor_status.script, unpacked/scripts/actor_status_sleep.script, unpacked/scripts/actor_status_thirst.script, unpacked/scripts/arszi_psy.script, unpacked/scripts/actor_effects.script, unpacked/scripts/game_difficulties.script, unpacked/scripts/ui_sleep_dialog.script, unpacked/scripts/_g.script, unpacked/configs/plugins/actor_effects.ltx]
related: [flows/actor-condition.md, flows/item-behaviors-lua.md, flows/game-modes.md, flows/lua-callback-lifecycle.md, flows/alife-services.md, flows/damage-hit.md, modlayer/gameplay/actor-survival.md, modlayer/gameplay/items-lua.md, GLOSSARY.md]
updated: 2026-06-11
verified: code
---

# Flow: actor survival & status (F103)

The Lua survival layer stacked on top of the engine condition tick (`flows/actor-condition.md`,
F30). The engine owns health/power/satiety/radiation/bleeding/boosters; this layer adds what the
engine does *not* simulate — **sleep deprivation, thirst, and psy health** — plus the **custom HUD
indicator strip** that visualizes both worlds, and the **item-use / overlay effects** module
(`actor_effects`). All of it is callback plugins on the F65 bus (`flows/lua-callback-lifecycle.md`);
none of it is a binder. **No `gamedata/` override of any file in this domain** (checked this
session; `plugins\actor_effects.ltx` also unforked).

**Provenance:** NO AGENT — `actor_status.script`, `actor_status_sleep.script`,
`actor_status_thirst.script`, `arszi_psy.script` READ-IN-FULL; `actor_effects.script` spine
READ-IN-FULL (init/settings, parse_anims, Update_ItemSwap, Update_Animations scheduler,
play_item_fx, play_continuous_effect, breath sounds, the entire callbacks section); its individual
overlay bodies (Update_Rad/Mask/Fog/Breathing/Bleeding/Blood/Impact, HUD_fog/HUD_mask, the three
Hit_ key-triggered animations) SKIMMED via structure scan only. Toggle wiring, `actor_on_sleep`
emitter, `BoosterID`, and the `[vodka]` LTX shape grep-verified.

## The four modules at a glance

- `actor_status` — read-only status getters (engine-backed), active-booster scan, safe-cover
  tracker, and the script-side **HUD indicator window** with an `add_indicator` extension point.
- `actor_status_sleep` / `actor_status_thirst` — near-twin optional meter plugins (own Lua meters,
  the engine knows nothing about them), toggled by difficulty factors.
- `arszi_psy` — Lua psy-health manager that **confiscates telepatic damage from the engine
  entirely** and runs its own meter, death scene, and (dormant) zombification.
- `actor_effects` — LTX-driven item-use animations, camera/sound effects, screen overlays, and a
  handful of keybind-triggered hand animations. Not a survival simulation; included here because
  it is the sensory half of item use and shares the same callback surface.

## actor_status — getters, boosters, safe cover, HUD

On `actor_on_first_update` it inverts the global `BoosterID` table (`_g.script`; HpRestore=0 …
TelepaticProtection=6 … MaxCount=17), snapshots surge covers, and attaches the HUD. On every
`actor_on_update` (1 s real-time throttle) it re-scans active boosters via the engine export
`conditions():BoosterForEach` into a name→{value,period} table (`get_boost(name)` is the public
read), and re-evaluates the current/nearest safe cover, publishing the result with
`SetEvent("current_safe_cover")` / `"nearest_safe_cover"` — **this is what `surge_manager` polls**
to decide whether the emission kills you (`flows/alife-services.md`). Covers come from the `list`
section of `misc\surge_manager.ltx`, restricted to server objects on the actor's level; being in
any `db.actor_inside_zones` cover, underground, or near an `item_tent` counts. Note there are two
scanner implementations and the *old* one is live — the rewritten `scan_safe_zone` is dead code
(the update calls `scan_safe_zone_old`).

Status getters wrap engine state, each with a dual signature — raw value, or a `visual` 0–4
severity tier for the indicator strip: `get_satienty` (sic — typo'd name is the real API; tiers
from engine `GetSatiety` vs `SatietyCritical`), `get_radiation` (ceil of radiation×4),
`get_overweight` (total weight vs max walk weight + outfit/backpack/belt-artefact bonuses).

The HUD is a script-defined `CUIScriptWnd` (`UIIndicators`) pushed into the engine render list via
`get_hud():AddDialogToRender` — same technique as other script HUDs. A static `indicators` table
maps name → {index, typ, functor, icon, background}; `typ = "state"` rows colour their slot
background white→grey→yellow→orange→red from the functor's 0–4 return, `typ = "booster"` rows show
while the booster is active and switch to a flashing variant when under 5 s remain. Defaults:
Radiation, Hunger, Overweight + eight booster slots. **`add_indicator(k,v)` is the extension
point** — sleep and thirst inject their rows through it (and remove them by passing nil), and mods
can do the same. The window redraws on a 1 s timer, on GUI show/hide, and hides itself whenever a
GUI occupies the screen (`main_hud_shown`).

## Sleep & thirst — twin Lua meters

Both are *features*, not always-on systems: `game_difficulties` calls their `toggle_feature` from
the per-difficulty game factors (`sleep` / `thirst`) at game load and on difficulty change —
`create()` registers the callbacks and injects the HUD indicator, `destroy()` unregisters,  wipes
the saved state and removes the indicator. Their own `on_game_load` auto-enable is commented out;
the difficulty manager is the only driver.

Each keeps a private 0–10000 meter (`last_sleep` / `last_drink`, higher = worse), advanced by a
fixed step (+27 / +30) every 300 in-game seconds inside `actor_on_update` (skipped entirely under
`g_god`). Items move the meter through the use pipeline (`flows/item-behaviors-lua.md`): on
`actor_on_item_use` they read the consumable's `eat_sleepiness` / `eat_thirstiness` ini key ×1000
— negative values on coffee/water relieve, positives worsen. Sleeping (`actor_on_sleep(hours)`,
emitted by `ui_sleep_dialog` after the sleep timer) *reduces* sleepiness by hours×700 but
*increases* thirst by hours×300. Past the second threshold both bleed actor power every 100 ms
(thirst also bleeds health; sleep's health-drain line is commented out). Four visual thresholds
drive the indicator tier and a blur post-effect; at meter ≥ force threshold (9000 sleep / 7200
thirst) the module gives itself an info portion and calls `ui_sleep_dialog.sleep_forced()` — the
forced-sleep "chance" constant is 10001 vs `math.random(100)`, i.e. always fires; randomness was
deliberately disabled. Thirst's forced collapse additionally rolls a **5% death chance**
(`db.actor:kill`). State persists through the `save_state`/`load_state` m_data callbacks
(`USE_MARSHAL` gated), keyed `m_data.sleep` / `m_data.drink`.

**Verified interaction quirk — shared blur effector id.** Both modules' `test_blur` use pp-effector
id 39568 and unconditionally `remove_pp_effector(39568)` at the top. A well-rested but thirsty
actor has the thirst blur silently cleared every time the sleep module ticks (and vice versa);
whichever module ran last wins until the other's next 300-game-second step re-asserts it. Also
within one module, the first (grey) tier adds the effector then immediately removes it, and the
second (yellow) tier leaves it added without ever setting its factor — only tiers 3–4 actually
set blur intensity.

## arszi_psy — Lua-owned psy health

The engine still has a psy-health stat (F30), but in Anomaly the actor's psy is simulated here.
The hijack is `actor_on_before_hit`: any hit of type 4 (Telepatic) aimed at the actor has its
`s_hit.power` **zeroed before the engine applies it**, and the damage is instead debited from the
module's own `psy_table.actor_psy_health` (1.0 = healthy), after subtracting stacked protection:
outfit + helmet `GetDefHitTypeProtection(Telepatic)` engine exports, belt artefacts
(condition × `telepatic_immunity` from each artefact's `hit_absorbation_sect`), and the active psi
booster value read via `BoosterForEach` (type 6). Controller tube attacks get a distance falloff
first (−1%/m, computed in 2D — the Y axis is ignored). Hits under 0.01 power are free, and a
zombified actor ignores psy damage entirely.

The meter regenerates per in-game second (base 0.0005 + psi-booster/25 + artefact-protection/100),
much faster while sleeping, and `drug_anabiotic` instantly restores it to 1 (this is the engine-side
half of the anabiotic; the inventory/combat-veto half lives in `itms_manager`, F102). Display is
one-way: `conditions():SetPsyBar(value)` pushes the Lua meter into the engine HUD bar every tick —
the engine bar is a *view*, not the source of truth. Below 0.9 a whisper loop + blur/sound-shock
post-effects scale with depletion. At ≤0: input is disabled, a black-screen death scene plays with
`on_enemy_eval` forcing all NPCs to ignore the actor, and after 25 in-game seconds the actor dies.
The alternative — zombification (community switch to `actor_zombied`, psy reset, permanent
radiation/alcohol overlays) — is **dead code in the shipped tree**: it is gated on the file-local
`fate_of_player_zombfication = false`, which nothing can flip externally. State persists as
`m_data.psy_table`.

## actor_effects — the sensory layer

A data-driven *timed-action scheduler*. `plugins\actor_effects.ltx` has one section per consumable
section name where every line is `millisecond-offset = lua-statement` (e.g. `[vodka]`: hide
weapon, hide helmet, disable NV, camera effector, disable inventory UI, re-enable everything,
gulp sounds). `actor_on_item_use` → `play_item_fx(obj:section())` copies the section into an
in-flight table; `Update_Animations` (per frame) executes each line via
`assert(loadstring(action))()` once its offset elapses. This is the engine room behind every
eat/drink/bandage hand animation — and the long tool animations too: `play_continuous_effect`
synthesizes the same kind of schedule for an arbitrary duration (repeating camera shake +
input/UI lockout), which is what `item_cooking`, `item_repair`, `item_parts`, `bind_campfire` etc.
call (F102). `use_weapon`/`use_helmet`/`actor_can_shoot` are the helpers those LTX lines invoke;
while an animation runs, `actor_on_weapon_before_fire` vetoes firing via `flags.ret_value`.

The same before-fire callback also implements the **broken-weapon click**: active non-melee weapon
at condition ≤ 0.014 never fires — the shot is vetoed and the weapon's `snd_empty` plays instead.

The rest of the module, dispatched from one `actor_on_update` with staggered timers (bleeding
overlay 400 ms, breathing/radiation 180 ms, blood-splash/mask 120 ms, the rest per frame), is
overlays and feedback: radiation PPE past a threshold, helmet/visor mask HUD with per-helmet
settings sections (breakability, blur, respirator flag, condition-based cracked variants, R1 vs
R2+ render branches), breathing fog and mask/exertion breath sounds (volume fades with third-person
camera distance), hit-impact camera kicks, weapon/detector draw-and-holster camera sways, shot
recoil effects from per-weapon LTX sections with `parent_section` inheritance, and health/stamina
HUD autohide. `on_key_release` adds the keybind features: mask wipe, torch-toggle hand animation,
grenade quickthrow, HUD toggle, item-animation toggle, screenshot shutter sound, and the PDA
map-key handling. Settings re-read on `on_option_change` from `ui_options`. The only persisted
state is whether the detector was out (`m_data.detector_was_active`, redrawn on load).

## Isolation ledger

- Sleep/thirst write nothing into the engine except power/health drains via `change_power`/
  `change_health` — uninstalling them (difficulty toggle off) cleanly removes meters, save keys,
  and indicators.
- `arszi_psy` is authoritative for actor psy damage; the engine's own psy pipeline never sees
  telepatic hits. Anything reading the engine psy bar reads what Lua last wrote.
- `actor_status` is read-only over engine state; its only outputs are the two `SetEvent` safe-cover
  keys (consumed by surge/psi-storm logic) and the HUD.
- `actor_effects` executes arbitrary Lua from LTX (loadstring) — `plugins\actor_effects.ltx` is
  effectively a script surface, not just data. Mods extend item animations by adding sections.
- Cross-module: the only inter-module call surface is `actor_status.add_indicator` (sleep/thirst,
  open to mods) and `ui_sleep_dialog.sleep_forced` (both meters force sleep through the same UI).

## Verified quirks & failure cases

- `get_satienty` — the typo is load-bearing; the HUD functor table and any mod must spell it
  this way.
- Shared blur pp-effector id 39568 between sleep and thirst (see above) — cross-suppression.
- `scan_safe_zone` (new implementation) is dead; `scan_safe_zone_old` is what runs.
- Forced-sleep "random" gates compare `math.random(100)` against 10001 — always true.
- Zombification in `arszi_psy` is unreachable without editing the file (local constant false).
- Controller tube falloff distance is 2D; vertical separation from a controller does not reduce
  tube damage.
- `gamemode_azazel` is the lone external consumer of `arszi_psy` (resets psy on respawn).

## Deeper

- Engine condition tick, booster apply path, satiety/radiation internals → `flows/actor-condition.md` (F30).
- Item use pipeline that feeds these callbacks (veto, multiuse, `eat_*` keys) → `flows/item-behaviors-lua.md` (F102).
- Surge/psi-storm consumption of `current_safe_cover` → `flows/alife-services.md`.
- Hit struct and `actor_on_before_hit` mechanics → `flows/damage-hit.md`.
- Domain routing → `modlayer/gameplay/actor-survival.md` (M17).
