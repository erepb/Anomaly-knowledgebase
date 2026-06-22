---
domain: flows/smart-terrain-core
tags: [modlayer, gameplay, unpacked, lua, smart-terrain, jobs, alife, flow]
sources: [unpacked/scripts/smart_terrain.script, unpacked/scripts/gulag_general.script, unpacked/scripts/xr_gulag.script, unpacked/scripts/xr_motivator.script, unpacked/scripts/bind_monster.script]
related: [flows/ai-squad-logic.md, flows/scheme-engine-core.md, flows/gulag-jobs.md, flows/db-registry.md, flows/alife-services.md, flows/save-load.md, flows/per-smart-logic-configs.md, modlayer/gameplay/smart-terrain-jobs.md, modlayer/gameplay/_overview.md, GLOSSARY.md]
updated: 2026-06-10
verified: code
---

# Flow: smart_terrain core (F72)

A **smart_terrain** is a server-side ALife zone that owns a set of *jobs* and the NPCs/squads working
them. It is the bridge between ALife population and the scheme engine: when an NPC spawns under a smart,
the smart selects a job, and that job's logic section is handed to `xr_logic` (F66) to drive the NPC.
This flow is the smart_terrain class lifecycle + the job-assignment→logic seam; the job *table* internals
are F73 (`flows/gulag-jobs.md`), and the higher-level orchestration view is F12 (`flows/ai-squad-logic.md`).

**Provenance:** the load-bearing bridges are **verified** in `smart_terrain.script`: `setup_logic`
(the `xr_logic.configure_schemes`/`determine_section_to_activate`/`activate_by_section` calls) and
`setup_gulag_and_logic_on_spawn` (the binder entry + the gulag-vs-`initialize_obj` fork). The class
spine symbols (`on_register`, `register_npc`, `select_npc_job`, `load_jobs`, `STATE_Write`/`STATE_Read`)
are **confirmed present** and their roles are *(agent-traced — not personally re-read)* except as noted.
No `gamedata/` override of `smart_terrain.script` (agent-checked).

## Concepts (verified where noted)

- **`se_smart_terrain`** — the server class (extends `cse_alife_smart_zone`). Registered into
  `db.smart_terrain_by_id` and the `sim_board` on `on_register` *(agent-traced)*.
- **job** — a unit of work (a logic section + metadata: type, priority, precondition, path). Built by
  `gulag_general.load_job` into `smart.stalker_jobs`/`monster_jobs`/`heli_jobs` (F73).
- **`npc_info[id]`** — the smart's per-NPC record `{se_obj, need_job, job, begin_job, stype, current_index}`
  *(agent-traced structure)*. `arriving_npc[id]` holds NPCs not yet arrived.

## Sequence (spine)

### Spawn entry — `smart_terrain.script:setup_gulag_and_logic_on_spawn(obj, st, se_obj, stype, loaded)` (verified)
Called from `xr_motivator:net_spawn` (stalkers) and `bind_monster` (monsters). Verified body: resolve
`smart` from `se_obj.m_smart_terrain_id` (guard `~= 65535` and `clsid.smart_terrain`); if the smart has an
`npc_info` for this id: if it has **no job**, `smart:select_npc_job(npc_info, true)` and stash the job's
`alife_task` level-vertex into `db.spawned_vertex_by_id`; if it **has a job**, set the spawn vertex
(unless `begin_job`), mark `begin_job=true`, clear `db.offline_objects[id]`, and `smart:setup_logic(obj)`.
**Else** (no smart / no npc_info) fall back to `xr_logic.initialize_obj(obj, st, loaded, db.actor, stype)`
— the non-gulag direct-logic path. This is the single junction between "NPC under a smart" and "NPC with
standalone logic."

### Job → logic bridge — `smart_terrain.script:se_smart_terrain:setup_logic(npc)` (verified)
Verified body: fetch `npc_info.job` (if none → unregister+re-register the NPC and bail); pick the ltx
source `job.ltx or self.ltx` and name `job.ini_path or self.ltx_name`; then the **F66 bridge**:
`xr_logic.configure_schemes(npc, ltx, ltx_name, npc_info.stype, job.section, job.prefix_name or self:name())`
→ `xr_logic.determine_section_to_activate(npc, ltx, job.section, db.actor)` →
`xr_logic.activate_by_section(npc, ltx, sect, prefix, false)`. So a job's `section` becomes the NPC's
active scheme. (Exclusive jobs carry their own external ltx; generated jobs use the smart's in-memory
`dynamic_ltx`, F73.)

### Lifecycle & job assignment *(agent-traced)*
- `se_smart_terrain:on_register` — `cse_alife_smart_zone.on_register`, `db.add_smart_terrain(self)`,
  `simulation_objects.register`, `self:load_jobs()` (→ `gulag_general.load_job`, F73), `sim_board` init,
  and delayed/after-load NPC registration. Fires `server_entity_on_register`.
- `register_npc(obj)` — routes the NPC to `arriving_npc` (not yet arrived) or fills `npc_info` and
  `select_npc_job` (arrived).
- `select_npc_job(npc_info, now, surge_started)` — walks the stype's job array by priority
  (`gulag_general.get_job_prior`/`get_job_precondition`, F73), picks the best available job not already
  taken (`npc_by_job_section`), and calls `setup_logic` when online. Re-evaluates to allow higher-prior
  jobs to preempt.
- `update` — per-frame: promote arrived NPCs (`update_jobs`), `try_respawn`, faction checks, campfire mgmt.
- `STATE_Write`/`STATE_Read` + `init_npc_after_load` — persist `npc_info`/`arriving_npc`/respawn state and
  rebuild job links on load (`flows/save-load.md`).

## Function digests (spine)

- **`smart_terrain.script:setup_gulag_and_logic_on_spawn`** — *Does:* spawn-time fork (gulag job setup vs
  `initialize_obj`). *Reads:* `se_obj.m_smart_terrain_id`, `smart.npc_info`. *Writes:* `db.spawned_vertex_by_id`,
  `npc_info.begin_job`. *Called by:* `xr_motivator`/`bind_monster` net_spawn. *(verified.)*
- **`smart_terrain.script:se_smart_terrain:setup_logic`** — *Does:* hand the NPC's job section to xr_logic.
  *Calls:* `xr_logic.configure_schemes`/`determine_section_to_activate`/`activate_by_section`. *(verified.)*
- **`se_smart_terrain:select_npc_job`** — *Does:* pick the best available job by priority + precondition.
  *Writes:* `npc_info.job`, `npc_by_job_section`. *(agent-traced.)*
- **`se_smart_terrain:on_register`/`register_npc`/`load_jobs`/`STATE_Write`/`STATE_Read`** — lifecycle +
  job-table load + persistence. *(agent-traced; load_jobs → F73.)*

## Isolation ledger (the seams)

- **Owned state:** the `se_smart_terrain` instance — `npc_info`, `arriving_npc`, `npc_by_job_section`,
  the `stalker/monster/heli_jobs` arrays, the in-memory `dynamic_ltx` (F73), respawn counters.
- **ALife seam (in):** registered as a `cse_alife_smart_zone`; spawn drives `setup_gulag_and_logic_on_spawn`
  via the binders (`flows/object-lifecycle.md`, `flows/alife-online-offline.md`).
- **Job seam:** `gulag_general.load_job` builds the jobs; `get_job_prior`/`precondition` rank them (F73).
- **Logic seam (out):** `setup_logic` → `xr_logic` (F66) — the smart decides *what* logic, xr_logic runs it.
- **Registries:** `db.smart_terrain_by_id` (F69), `sim_board`/`simulation_objects` (squad sim, F86/F41).
- **Persistence:** `STATE_Write`/`STATE_Read` + `init_npc_after_load` (`flows/save-load.md`).

## Threading / failure / edge cases

- **No-job recovery:** `setup_logic` with `npc_info.job == nil` unregisters+re-registers the NPC (a
  self-heal) — repeated "setup_logic: npc_info has no job" usually means the smart has no suitable job.
- **Arrival gating:** an NPC sits in `arriving_npc` until `arrived_to_smart`; jobs/logic only bind after
  arrival — newly spawned distant NPCs won't have smart logic yet.
- **`m_smart_terrain_id == 65535`** means "no smart" → the `initialize_obj` standalone path.
- **Priority preemption:** `select_npc_job` can move an NPC to a higher-priority job mid-life, switching
  its logic — expected churn, not a bug.

## Deeper

- Class + bridges: `unpacked/scripts/smart_terrain.script` (`setup_gulag_and_logic_on_spawn`,
  `se_smart_terrain:setup_logic`/`select_npc_job`/`on_register`/`register_npc`/`load_jobs`/`STATE_*`)
- Job tables: `unpacked/scripts/gulag_general.script` (F73 `flows/gulag-jobs.md`); helpers
  `unpacked/scripts/xr_gulag.script` (`get_npc_smart`)
- Logic seam: `flows/scheme-engine-core.md`; orchestration view: `flows/ai-squad-logic.md`; squad sim:
  `flows/alife-services.md`; persistence: `flows/save-load.md`

## Maintenance

- The verified bridges (`setup_logic`, `setup_gulag_and_logic_on_spawn`) are the stable core; re-read if
  the job→logic hand-off changes.
- Promote `select_npc_job`/`on_register`/`STATE_*` to verified on next read (good pairing: F73 or a
  save/load task).
- Check `gamedata/scripts/smart_terrain.script` for an override before trusting (none at writing).
