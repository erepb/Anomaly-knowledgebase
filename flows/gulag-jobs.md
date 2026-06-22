---
domain: flows/gulag-jobs
tags: [modlayer, gameplay, unpacked, lua, gulag, jobs, smart-terrain, flow]
sources: [unpacked/scripts/gulag_general.script, unpacked/scripts/xr_gulag.script, unpacked/scripts/smart_terrain.script]
related: [flows/smart-terrain-core.md, flows/scheme-engine-core.md, flows/logic-ltx-switching.md, flows/ai-squad-logic.md, flows/db-registry.md, flows/per-smart-logic-configs.md, modlayer/gameplay/smart-terrain-jobs.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: gulag job board (F73)

The "gulag" system defines the **jobs** a smart_terrain offers. `gulag_general.load_job` builds a
smart's job arrays (and an in-memory logic LTX) from the level's patrol paths / smartcovers / exclusive
configs; `smart_terrain` (F72) then assigns those jobs to NPCs by priority + precondition and hands the
job's logic section to `xr_logic` (F66). This flow is the job *definition + ranking* side; assignment is
F72 (`flows/smart-terrain-core.md`).

**Provenance:** **verified** in `gulag_general.script`: the job-property accessors
(`get_job_prior`/`get_job_precondition`/`get_job_type` and their `exclusive`-vs-`job_type_id` fork) and
the full **`job_info_by_job_type_id`** catalog (types 0–20 with their prior/precondition). `load_job`
(the builder) and the per-type `init_*_jobs` generators + `add_exclusive_job` are **confirmed present**
and their behavior is *(agent-traced — not personally re-read)*. `xr_gulag.get_npc_smart` is agent-traced.
No `gamedata/` functional override of `gulag_general`/`xr_gulag` (agent-checked; gamedata only uses the
read-only accessors).

## Job model (verified)

A job is a table. Two flavors:
- **generated** — carries a `job_type_id` (index into the catalog) and a `section` (its logic section).
  Its prior/type/precondition come from `job_info_by_job_type_id[job_type_id]`.
- **exclusive** — `job.exclusive == true`; carries its own `prior`, `job_type`, `precondition_function`,
  and an external `ltx`/`ini_path` (a `.script`/.ltx file), plus optional `prefix_name`.

Accessors fork on `exclusive` (verified): `get_job_prior(job)` → `job.prior` (exclusive) else
`job_info_by_job_type_id[job.job_type_id].prior`; same shape for `get_job_precondition`/`get_job_type`.

### The job-type catalog — `gulag_general.script:job_info_by_job_type_id` (verified, full)
Indices 0–20, each `{job_type, prior, precondition_function?}`:
`0` generic_point (point_job, 3) · `1` campfire_point (point_job, 10) · `2` surge (path_job, 60,
`precond_surge`) · `3` sleep (path_job, 10, `precond_safe_job_sleeper`) · `4` collector (path_job, 25,
`precond_collector`) · `5` walker (path_job, 15, `precond_safe_job`) · `6` patrol (path_job, 25,
`precond_safe_job_patrol`) · `7` animpoint (smartcover_job, 15, `precond_animpoint`) · `8`
animpoint_surge (smartcover_job, 60, `precond_surge`) · `9` guard (path_job, 25, `precond_safe_job_guard`)
· `10` follower (path_job, 14, `precond_safe_job_guard`) · `11` sniper (path_job, 30, `precond_sniper`) ·
`12` camper (path_job, 45, `precond_camper`) · `13` beh (point_job, 30) · `14` beh_surge (point_job, 60,
`precond_surge`) · `15` beh_trader (point_job, 59, `precond_trade_job`) · `16` beh_mechanic (point_job,
59, `precond_upgrade_job`) · `17` mob home (point_job, 40) · `18` heli_move (heli_path_job, 45) · `19`
heli_hide (heli_hide_job, 100, `precond_heli_hide`, prefix "") · `20` heli point (point_job, 3). Higher
prior = preferred; surge jobs (60) outrank normal work; heli_hide (100) dominates.

## Sequence (spine)

### Build the jobs — `gulag_general.script:load_job(smart, debug_save)` *(agent-traced)*
Called from `se_smart_terrain:load_jobs` (F72) on register. Builds the smart's `stalker_jobs`/
`monster_jobs`/`heli_jobs` arrays and an in-memory logic ini (`db.dynamic_ltx[gname]`, aliased as
`smart.ltx`, `smart.ltx_name = "*"..gname`) by: emitting common `[meet@…]` sections, then running the
per-type `init_*_jobs` generators (campfire/generic_point/surge/sleep/collector/walker/patrol/animpoint/
guard/sniper/camper/beh/monster/heli…), each of which reads level patrol paths or registered smartcovers,
emits `[logic@…]` + scheme subsections into a string buffer, and registers a `{section, job_type_id}`
entry; then `add_exclusive_job` for the smart's `work*` config fields (external `.script` logic with its
own prior/`suitable` condlist); finally sorts each array by priority descending. *(agent-traced — the
generators are a large, repetitive family; read the specific one a task needs and promote it.)*

### Resolve an NPC's smart — `xr_gulag.script:get_npc_smart(obj)` *(agent-traced)*
Returns the `se_smart_terrain` for `obj` via `alife():object(id).m_smart_terrain_id` (or nil). Used by
the scheme engine and many schemes to find "which smart am I in / what job."

### Assignment + logic (the consumers — F72/F66)
`select_npc_job` (F72) ranks the arrays via `get_job_prior`/`get_job_precondition` and assigns;
`setup_logic` (F72) feeds `job.section` + `job.ltx`/`smart.ltx` to `xr_logic.configure_schemes`/
`activate_by_section` (F66). So: **gulag defines & ranks → smart_terrain assigns → xr_logic runs.**

## Function digests (spine)

- **`gulag_general.script:get_job_prior` / `get_job_precondition` / `get_job_type`** — *Does:* fork on
  `job.exclusive` → job's own field or `job_info_by_job_type_id[job_type_id]`. *(verified.)*
- **`gulag_general.script:job_info_by_job_type_id`** — *Does:* the job-type catalog (0–20). *(verified.)*
- **`gulag_general.script:load_job`** — *Does:* build the smart's job arrays + in-memory logic ini.
  *Writes:* `smart.stalker/monster/heli_jobs`, `smart.ltx`/`ltx_name`, `db.dynamic_ltx`. *(agent-traced.)*
- **`gulag_general.script:add_exclusive_job`** — *Does:* register an external-`.script` job from a smart's
  `work*` field (prior + `suitable` condlist). *(agent-traced.)*
- **`xr_gulag.script:get_npc_smart`** — *Does:* obj → its `se_smart_terrain`. *(agent-traced.)*

## Isolation ledger (the seams)

- **Owned output:** the smart's `stalker/monster/heli_jobs` arrays + the in-memory `db.dynamic_ltx[gname]`
  logic ini (built once per smart). Module-level: `job_info_by_job_type_id` catalog + the `precond_*`
  predicate set.
- **Producer (in):** `se_smart_terrain:load_jobs` calls `load_job` on register (F72).
- **Consumers (out):** `select_npc_job` (ranking, F72) and `setup_logic` (the `job.section`→`xr_logic`
  bridge, F66). Preconditions call into `xr_conditions` (e.g. `surge_started`, F67) + community/time checks.
- **Level seam:** generators read `level.patrol_path_exists` / registered smartcovers / `beh_gulag_jobs.ltx`
  — jobs are derived from level geometry + config.

## Threading / failure / edge cases

- **Priority ties / preemption:** equal-prior jobs are order-dependent in the sorted array; surge (60) and
  heli_hide (100) deliberately dominate — expected during surges.
- **Preconditions gate suitability:** a job with a failing `precondition_function` is skipped for that NPC
  (e.g. `precond_collector` needs a detector). A smart can have jobs no current NPC can take.
- **In-memory ini lifetime:** `db.dynamic_ltx[gname]` is the generated logic source — it is not on disk;
  don't look for these `[logic@…]` sections in `configs/`. Exclusive jobs *do* live in real `.script` files.

## Deeper

- Job builder + catalog: `unpacked/scripts/gulag_general.script` (`load_job`, `job_info_by_job_type_id`,
  `get_job_prior`/`get_job_precondition`, `add_exclusive_job`, the `init_*_jobs` generators, `precond_*`)
- Smart lookup: `unpacked/scripts/xr_gulag.script` (`get_npc_smart`)
- Consumers: `flows/smart-terrain-core.md` (assignment + `setup_logic`), `flows/scheme-engine-core.md`
  (logic), `flows/logic-ltx-switching.md` (the `[logic@…]` format the generators emit)

## Maintenance

- The verified core (the `job_info_by_job_type_id` catalog + the `exclusive` accessor fork) is the stable
  reference; re-read if job types are added/reprioritized.
- The `init_*_jobs` generators are a repetitive family — read + promote the specific generator a task
  needs rather than all at once.
- Check `gamedata/scripts/` for overrides of `gulag_general`/`xr_gulag` before trusting (none at writing).
