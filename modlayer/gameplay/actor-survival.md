---
domain: modlayer/gameplay/actor-survival
tags: [modlayer, gameplay, unpacked, lua, actor, survival, hud, map]
sources: [unpacked/scripts/actor_status.script, unpacked/scripts/arszi_psy.script, unpacked/scripts/actor_effects.script, unpacked/configs/plugins/actor_effects.ltx]
related: [INDEX.md, ROADMAP-gameplay.md, modlayer/gameplay/_overview.md, flows/actor-survival-status.md, flows/actor-menu-inventory-ui.md, flows/actor-condition.md, flows/item-behaviors-lua.md, modlayer/gameplay/items-lua.md, GLOSSARY.md]
updated: 2026-06-11
verified: map
---

# Actor survival & status — L1 map (M17)

The Lua survival layer over the engine condition tick: sleep/thirst/psy meters the engine doesn't
simulate, the custom HUD indicator strip, and the item-use animation/overlay module. Mechanics
live in `flows/actor-survival-status.md` (F103); the engine half (health/power/satiety/radiation/
boosters) is `flows/actor-condition.md` (F30).

## The pieces

| Piece | File(s) | Role | Flow |
|-------|---------|------|------|
| status hub + HUD | `actor_status.script` | engine-backed getters (satiety/rad/overweight), booster scan, safe-cover `SetEvent` feed for surge, `UIIndicators` strip with `add_indicator` extension point | F103 ✅ |
| sleep meter | `actor_status_sleep.script` | optional 0–10000 meter, `eat_sleepiness` items, forced sleep, power drain | F103 ✅ |
| thirst meter | `actor_status_thirst.script` | twin of sleep (`eat_thirstiness`, +health drain, 5% collapse death) | F103 ✅ |
| psy health | `arszi_psy.script` | confiscates telepatic hits from the engine into a Lua meter; protection stack, death scene, dormant zombification; engine psy bar is display-only (`SetPsyBar`) | F103 ✅ |
| sensory layer | `actor_effects.script` + `plugins/actor_effects.ltx` | LTX timed-action scheduler (item-use anims via loadstring), overlays/breath/recoil/mask HUD, broken-weapon click, keybind animations | F103 ✅ (overlay bodies SKIMMED) |
| toggles & sleep UI | `game_difficulties.script`, `ui_sleep_dialog.script` | difficulty factors drive sleep/thirst `toggle_feature`; sleep dialog emits `actor_on_sleep(hours)` and hosts `sleep_forced` | F103 ✅ (grep-verified) |
| actor menu & inventory UI | `actor_menu.script`, `actor_menu_inventory.script`, `ui_inventory.script`, `utils_ui.script` | engine ActorMenu replaced by Lua `UIInventory` (functor veto seam, mode mirror + info portions, bags/context-menu/drag-drop, `useN_*` LTX custom rows, cell framework) | F104 ✅ |

## How it fits

Engine condition tick (F30) owns the real stats; these plugins ride the F65 callback bus —
`actor_on_update` for meters/overlays, `actor_on_item_use` for consumable effects (F102 pipeline),
`actor_on_before_hit` for the psy hijack, `save_state`/`load_state` for persistence. Outputs back
into the engine are narrow: power/health drains, `SetPsyBar`, the safe-cover `SetEvent` keys
surge logic polls, and fire vetoes via `flags.ret_value`.

## Subsystem deep-dives (delegates to flows)

- Meters, psy hijack, HUD strip, the `actor_effects` scheduler, verified quirks (shared blur
  effector id, `get_satienty` typo, dead zombification, always-true force gates) →
  `flows/actor-survival-status.md` (F103) ✅
- Inventory window, takeover seam, context-menu framework, `utils_ui` cells, monkey-patch
  surface → `flows/actor-menu-inventory-ui.md` (F104) ✅
- Engine condition & booster internals → `flows/actor-condition.md` (F30)
- Item use pipeline feeding the meters → `flows/item-behaviors-lua.md` (F102)
- Surge consumption of safe-cover events → `flows/alife-services.md`

## Maintenance

- F103 ✅ + F104 ✅ — domain complete per the roadmap's M17 scope. No `gamedata/` override of any
  core file at writing (the patch files hooking UIInventory are forked but identical in the
  relevant regions); re-check on pickup. Promote `actor_effects` overlay bodies and the
  UIInventory *(agent-traced)* internals from SKIMMED/draft when a task touches them.
