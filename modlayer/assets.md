---
domain: modlayer/assets
tags: [modlayer, assets, shaders, localization]
sources: [gamedata/shaders/r3/, gamedata/textures/, gamedata/materials/, gamedata/configs/ui/, gamedata/configs/text/, gamedata/levels/]
related: [modlayer/_overview.md, engine/renderer.md, modlayer/dxml.md, GLOSSARY.md]
updated: 2026-06-05
verified: code
---

# Assets (shaders, textures, materials, UI, text, levels)

Non-script content the mod layer ships. Contents/formats verified by listing the actual directories.

## Shaders (`gamedata/shaders/{r1,r2,r3}`)

HLSL GPU programs by render path. **The `r3` folder serves both R3 (DX10) and R4 (DX11)** — verified:
`getShaderPath()` returns `"r3\\"` in both `r3.h`/`r4.h` (`engine/renderer.md`). **Extensions actually
present in `gamedata/shaders/r3` (verified by listing):** `.ps` (16), `.vs` (4), `.s` (4 script-side
blender config), `.h` (5 shared includes, incl. `common_functions.h`). The engine also supports
`.gs`/`.cs` and hull/domain shaders, but **no such files ship in this folder** here. Each shader is
bound by a C++ **blender** (`src/Layers/...`, see `engine/renderer.md`). (SPOM shaders exist only on
the `spom` branch — `features/spom.md`.)

## Textures & materials

- `gamedata/textures/` — `.dds` textures (verified: 3 `.dds` files ship here).
- `gamedata/materials/` — **LTX configs** `materials.ltx` and `material_pairs.ltx` (verified by
  reading), not a binary gamemtl (the base `gamemtl.xr` lives outside this repo; these files
  override/extend it). `materials.ltx`: per-material physics + flags (`friction`, `damping`, `spring`,
  `density_factor`, `flag_breakable`/`bounceable`/`skidmark`/`bloodmark`/`climable`/`passable`/
  `injurious`/`bullet_no_ricochet`/…). `material_pairs.ltx`: two-material interaction
  (`[mat1@mat2]` sections → `breaking_sounds`, `step_sounds`, `collide_sounds`, `collide_particles`,
  `collide_marks`). Both support DLTX overrides via `mod_materials_*`/`mod_material_pairs_*` with `@`.

## UI & text configs

- `gamedata/configs/ui/` — UI layout XML/LTX (dir exists). These XML files are also DXML targets
  (`modlayer/dxml.md`) and load via `UIXmlInit.cpp` (`engine/subsystems-misc.md`).
- `gamedata/configs/text/` — localization string tables (dir exists). DXML only processes translation
  strings in `eng/`/`rus/` (verified in `COnXmlRead`, see `modlayer/dxml.md`).

## Levels

- `gamedata/levels/` — per-level folders (verified: `jupiter`, `k00_marsh`, `l05_bar`, … exist).
  *(INFERRED — unverified: their exact contents; the full base levels live in the Anomaly install /
  `unpacked/` reference, not here.)*

## Deeper

- Shaders: `gamedata/shaders/r3/` (`.ps/.vs/.s/.h`), shared include `r3/common_functions.h`
- UI XML: `gamedata/configs/ui/`; loader `src/xrGame/ui/UIXmlInit.cpp`
- Localization: `gamedata/configs/text/`; materials: `gamedata/materials/*.ltx`

## Maintenance

- Re-list `gamedata/shaders/r3` to refresh the extension counts; if `.gs/.cs/.hs/.ds` files appear,
  add them.
- `gamedata/materials/*.ltx` fields verified by reading; re-check the field list if the base files
  gain new flags.
- If a `gamedata/shaders/r4` folder appears, update the "r3 serves R4" note here + `engine/renderer.md`.
