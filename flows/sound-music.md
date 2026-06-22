---
domain: flows/sound-music
tags: [modlayer, gameplay, unpacked, lua, sound, music, audio, flow]
sources: [unpacked/scripts/xrs_dyn_music.script, unpacked/scripts/xr_sound.script, unpacked/scripts/sound_theme.script, unpacked/scripts/sound_manager.script, unpacked/scripts/sound_ambient.script, unpacked/scripts/mob_sound.script, unpacked/scripts/ph_sound.script, unpacked/scripts/xr_zones_sound.script, unpacked/scripts/aaa_sound_object_patch.script, unpacked/scripts/level_weathers.script, unpacked/configs/game_levels_music.ltx]
related: [INDEX.md, modlayer/gameplay/_overview.md, flows/ui-news-tips.md, flows/options-runtime.md, flows/scheme-engine-core.md, flows/script-npc-actions.md, flows/scheme-sensors-misc.md, flows/monster-schemes.md, flows/anomaly-zones.md, flows/renderer-environment.md, flows/utils-libraries.md, flows/db-registry.md, GLOSSARY.md]
updated: 2026-06-21
verified: code
---

# Flow: sound & dynamic music

The audio layer: how music is chosen and crossfaded, how scripted/ambient sounds play, and the small
patch that keeps Lua sound objects from leaking. There is **no single "sound manager"** — it's a set of
loosely-coupled subsystems all sitting on the engine `sound_object` class. This is a P3 catalog flow
(standalone, no dedicated map; routed from the M1 gameplay hub).

**Provenance:** **verified, no agent.** `xrs_dyn_music` and `aaa_sound_object_patch` READ-IN-FULL;
`xr_sound` `get_sound_object`/`get_safe_sound_object` region read; `sound_theme`, `sound_manager`,
`sound_ambient`, `mob_sound`, `ph_sound`, `xr_zones_sound` STRUCTURE-SURVEYED (class/function inventory);
`level_weathers` + `game_levels_music.ltx` characterized from headers (weather is a sibling subsystem,
not re-read in depth here). **gamedata deltas diffed:** `aaa_sound_object_patch` (cosmetic only) and
`options_modded_exes_sound` (+ a new sound-delay subpage) — see below.

---

## Two music systems (they don't share code)

1. **Engine ambient / background level music** — driven by the engine from `game_levels_music.ltx`:
   per-level (and shared / day-normal / etc.) playlists, each track tagged with a start-hour, stop-hour,
   base volume, and two fade values. This is the calm exploration music; it plays on the engine's
   `snd_volume_music` channel. Pure config + engine — no Lua scheduler.
2. **Dynamic combat music** — `xrs_dyn_music.script`, a Lua state machine that **ducks** the ambient
   channel and layers combat themes over it. This is the F116 centerpiece below.

The link between them is the `snd_volume_music` console variable: `xrs_dyn_music` fades the engine
ambient music down to 0 while combat music rises, and restores it when combat ends.

## xrs_dyn_music — dynamic combat music (READ-IN-FULL)

A `main_loop` (registered via `AddUniqueCall`, throttled to ~1 s) runs a small state machine. The whole
feature is gated by the engine function **`IsDynamicMusic()`** (an option); `activate_feature` /
`deactivate_feature` add/remove the loop on main-menu toggle.

- **State (`get_theme_state`)** returns `start` / `idle` / `finish` / `nil`:
  - actor dead, or in a **silence zone**, or **no stalker enemy** → fade theme out → `finish`.
  - **has a stalker enemy** (scans `xr_combat_ignore.fighting_with_actor_npcs` for an `IsStalker`) →
    `start` (first time) then `idle`. **Only stalkers trigger combat music — monsters do not.**
  - Silence zones = `db.storage.silence_zone_table` plus three **hardcoded** `*_no_assault` restrictors
    (`zat_a2`, `jup_a6`, `jup_b41`).
- **Themes are hardcoded in the script** — four themes (`music\combat\themeN_part_1..3`), three parts
  each. On `start_theme` it picks a random theme + part; `select_next_track` chains parts gaplessly via
  `theme:play_at_time(... + TRACK_SWITCH_DELTA)` and loops.
- **Crossfade** — `fade_theme` raises the combat volume while `fade_ambient` lowers `snd_volume_music`;
  both step by `VOLUME_DELTA` per tick (set to `ambient_vol/50` at init), or jump instantly when
  `force_fade` is set (combat start/end, and the surge "killing all" case below).
- **Surge coupling** — when `surge_manager.sound_started()`: if `is_killing_all()` it force-restores the
  ambient channel; otherwise it force-fades ambient to 0 (the surge has its own audio).
- **`stereo_sound` class** wraps engine `sound_object` (2-D playback `s2d` attached to the actor) with
  `attach_tail` for the gapless part chaining.

So combat music is fully self-contained and **not config-driven** — adding/replacing themes means
editing the script's `combat_themes` table.

## The sound playback core

- **`xr_sound.script`** — the scheme-facing sound layer NPCs play through (the `sound` logic field).
  Public surface: `set_sound_play` / `play_sound` / `play_sound_looped` / `stop_sound_looped`,
  `get_sound` / `set_sound` (per-NPC theme assignment), `update` / `update_actor`, `stop_all_sounds` /
  `stop_sounds_by_id`, and a tutorial-on-sound hook. The selector **`get_sound_object(theme, t_type)`**
  reads a theme's entries (collected from the sound-theme LTX) and returns a `sound_object` —
  `random` (avoids immediate repeat via a per-theme `last_id`), sequential, or looped; stereo themes
  return paired `_r` / `_l` objects. Objects are cached per path by **`get_safe_sound_object`**
  (a `path → sound_object` table) so the same file isn't reopened.
- **`sound_theme.script`** — the config-driven **theme registry**: classes `npc_sound` (NPC voice
  lines), `actor_sound`, `object_sound` (object-attached), and `looped_sound` (continuous), plus
  `load_sound` / `init_npc_sound` that build them from the sound-theme config. Each handles
  play / `select_next_sound` / end-callback / stop and full save/load. This is the registry the news
  layer's "spoken news" reads (`sound_theme`, F112 `flows/ui-news-tips.md`).
- **`aaa_sound_object_patch.script`** — loaded **first** (the `aaa_` filename prefix) it **replaces the
  global `sound_object` class**: `_G.sound_object = my_sound_object`, which registers every created sound
  into a `soundCache` and runs an `AddUniqueCall` GC loop that drops finished ones. A lifetime/leak guard
  for Lua-created sounds. **gamedata copy = cosmetic-only delta** (`printf` → `printf_me` in dead
  comments); functionally identical.

## Scheme-driven sound systems

These are normal logic schemes / per-frame systems that emit sounds:

- **`sound_manager.script`** — the `sm` scheme: campfire **storyteller** sequences. A `CStory` is an
  ordered list of phrases; `register_npc` / `set_storyteller` / `choose_random_storyteller` /
  `get_next_phrase` walk a group through a conversation. This is the "stalkers telling stories at the
  campfire" system.
- **`sound_ambient.script`** — the level **ambient** system: `update_ambient` (random level ambient
  sounds), `update_wind`, and `update_rain_helm_sound` (the rain-on-helmet sound), ticked from
  `actor_on_update`. Config-driven from the environment/ambient data.
- **`ph_sound.script`** — the `ph_sound` scheme for **physics/world objects** (`snd_source` class:
  looped or hit-triggered sounds on placed objects, with a `hit_callback`).
- **`xr_zones_sound.script`** — **zone-entry** sounds (`zone_sound` class, `on_enter`).
- **`mob_sound.script`** — monster sound selection (`pick_sound_from_set`), the audio side of monster
  schemes (`flows/monster-schemes.md`).

## Weather sibling — level_weathers (characterized, not deep-read)

`level_weathers.script` is the Lua **dynamic-weather cycle driver** (AtmosFear-3 lineage, reworked by
Alundaio + Tronex for all levels). It reads `environment\dynamic_weather_graphs.ltx` and advances the
weather descriptor the engine `CEnvironment` then renders (`flows/renderer-environment.md`). It is
listed alongside sound here because `game_levels_music.ltx` selection is time-of-day-driven like
weather, but it is **really its own environment subsystem** — covered briefly; deep treatment belongs to
a future weather/environment flow.

## gamedata deltas (runtime-winning)

- **`aaa_sound_object_patch`** — present in `gamedata/` but **byte-identical except comment text**
  (`printf` → `printf_me`). No behavioral change; the patch ships in `unpacked/` too.
- **`options_modded_exes_sound`** (OVERRIDE) — adds one extra options subpage,
  `options_modded_exes_sound_delay.PAGE`, into the sound options page.
- **`options_modded_exes_sound_delay`** (NEW, gamedata-only) — a small sound-delay options subpage
  (F17, `flows/options-runtime.md`).

## Verified quirks

- **Combat music ignores monsters** — only an `IsStalker` enemy in `fighting_with_actor_npcs` starts
  `xrs_dyn_music`; a mutant ambush plays no combat theme.
- **Themes are hardcoded** — `combat_themes` is a literal table in the script, not config; modders edit
  the script to change them (unlike the config-driven ambient music).
- **Silence zones are partly hardcoded** — beyond `db.storage.silence_zone_table`, three specific
  `*_no_assault` restrictor names are baked into `actor_in_silence_zone`.
- **Two cooperating caches** — `xr_sound.get_safe_sound_object` (path→object) and the
  `aaa_sound_object_patch` global override (lifetime GC) both manage `sound_object` lifetime; they are
  independent and both live.

## Deeper

- Engine ambient music data → `game_levels_music.ltx`; the option gate `IsDynamicMusic()` is engine-side.
- Spoken news uses the theme registry → `flows/ui-news-tips.md` (F112); sound options pages →
  `flows/options-runtime.md` (F17).
- Sound-emitting schemes plug into the scheme engine → `flows/scheme-engine-core.md`,
  `flows/script-npc-actions.md`; monster audio → `flows/monster-schemes.md`.
- Weather/environment rendering counterpart → `flows/renderer-environment.md`.
- Gameplay hub → `modlayer/gameplay/_overview.md` (M1).
