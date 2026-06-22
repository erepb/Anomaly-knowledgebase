---
domain: engine/sound
tags: [engine, cpp, sound]
sources: [src/xrSound/]
related: [engine/_overview.md, modlayer/scripting.md, GLOSSARY.md]
updated: 2026-06-04
---

# xrSound

Audio subsystem: source playback, 3D positioning, environment/reverb, streaming, and the **doppler**
effect added by the modded exes (code by Cribbledirge, edited by demonized).

## Notes

- Doppler is toggleable via options (`gamedata/scripts/options_modded_exes_doppler.script`) and a
  sound-delay option (`options_modded_exes_sound_delay.script`).
- A Lua-side sound object patch exists (`gamedata/scripts/aaa_sound_object_patch.script`) — relevant
  when debugging script-played sounds.
- Weapon sound extras (`snd_shoot_actor_first`, silencer-first-shot variants) are weapon LTX params
  consumed in `xrGame` (see `ltx_help_ex.script`), not here.

## Deeper

- Engine audio: `src/xrSound/` (grep `CSound`, `doppler`, `SoundRender`)
- Options: `gamedata/scripts/options_modded_exes_doppler.script`, `..._sound.script`,
  `..._sound_delay.script`
- Script patch: `gamedata/scripts/aaa_sound_object_patch.script`

## Maintenance

- Small module; update only if the audio backend or doppler implementation changes.
