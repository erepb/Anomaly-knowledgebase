---
domain: features/spom
tags: [spom, renderer, shaders, r4, dx11]
sources: [claudedocs/design_spom_silhouette_skirt.md, claudedocs/implementation_spom_final.md]
related: [engine/renderer.md, GLOSSARY.md]
updated: 2026-06-05
branch: spom
---

# SPOM — Silhouette Parallax Occlusion Mapping

> **Branch-specific.** SPOM lives on the **`spom`** branch (and `spom_failed_improv`). It is **NOT
> present on the current working branch** (`docs`/`ratio` etc.) — verified: no `spom`/`silhouette`
> matches in `src/Layers` or `gamedata/shaders` on this branch. Because the KB is local/gitignored,
> this file persists across checkouts; only trust its code touch points when you are on `spom`.

Parallax occlusion mapping extended to deform the silhouette/edges of geometry, not just the interior
surface. R4/DX11 path. See `GLOSSARY.md#spom`.

## Where the real detail is

Read these before touching the code — do not re-derive:

- **Auto-memory (freshest state):** `C:/Users/Erepb/.claude/projects/B--Games-xray/memory/project_spom_state.md`
  — current architecture (cross-`Fvisual` buffer + manifold check + hybrid shader path), discarded
  approaches, a debug-visualizer trap to avoid, and residual limitations.
- **Design/research docs** (`claudedocs/`, present on this branch):
  - `design_spom_silhouette_skirt.md` — silhouette "skirt" design
  - `implementation_spom_final.md` — implemented approach
  - `implementation_spom_option_c.md` — alternative option
  - `research_silhouette_parallax_occlusion_mapping.md`
  - `research_edge_face_visibility_silhouette_detection.md`
  - `research_spom_height_convention.md`

## Code touch points (on the `spom` branch only)

Confirmed via `git grep -il spom spom` — SPOM is edits inside existing files, not new "spom_*" files:

- `src/Layers/xrRender/FVisual.cpp` / `FVisual.h`, `FProgressive.cpp` — the cross-`Fvisual` buffer.
- `src/Layers/xrRender/uber_deffer.cpp`, `R_Backend.h`, `R_Backend_Runtime.h`, `Shader.h`.
- `src/Layers/xrRenderDX10/dx10R_Backend_Runtime.h`, `dx10ResourceManager_Resources.cpp`,
  `dx10Texture.cpp`.
- Shaders: `gamedata/shaders/r3/` (POM/silhouette passes — confirm on the branch).

Inspect without switching branches via `git grep -n spom spom -- <path>` or
`git show spom:<path>`.

## Maintenance

- This file documents another branch; keep code pointers verified with `git grep... spom`, not by
  reading the current working tree (where SPOM is absent).
- The auto-memory is the live state — update it first when SPOM changes, then refresh pointers here.
- If SPOM merges into the main working branch, drop the branch-specific banner and re-verify touch
  points against the working tree.
