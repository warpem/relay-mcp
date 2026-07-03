---
name: relay-mcp
description: Use when operating Relay's MCP server to manage cryo-EM/cryo-ET processing workflows — creating projects and spaces, building job pipelines, connecting ports, queuing jobs, monitoring status, or retrieving results. Also use when asked how Relay is organized or what job types are available.
---

# Relay MCP

## Overview

Relay is a platform for cryo-electron microscopy (cryo-EM) and cryo-electron tomography (cryo-ET) data processing. It exposes a Model Context Protocol (MCP) server that lets agents build and run processing pipelines programmatically.

## Organization Hierarchy

```
Project
 └─ Space  (one workflow scope; may hold many pipelines)
     ├─ View  (visual arrangement; jobs must be placed in a view)
     └─ Job  (a processing step with typed input/output ports)
          └─ Edge  (port-to-port connection between jobs)
```

- **Project** — top-level container, owned by a user, may have members.
- **Space** — a workflow scope inside a project. Holds all jobs and edges. Always has at least one View; `create_space` auto-creates "View 1".
- **View** — visual canvas. `create_job` requires a view id. Use `list_views` to find one.
- **Job** — a processing step. Has strongly-typed input and output **ports**. Port resource types must match when connecting jobs. Only jobs in `Building` status can have parameters changed.
- **Edge** — a directed connection from an output port (`PortOut`) to an input port (`PortIn`). Each PortIn may declare `MinItems`/`MaxItems` for how many upstream connections it accepts.

## Authentication

All tools require a Bearer token (Personal Access Token, PAT). Tokens carry three independent access levels — `None`, `Read`, `EditRun`, or `Manage` — for each of three tiers:

| Tier | Controls |
|------|----------|
| Project | `list_projects`, `create_project`, `delete_project` |
| Space | `list_spaces`, `list_views`, `create_space`, `delete_space` |
| Job | All job tools (list, get, create, configure, connect, queue, abort, delete) |

Read tools (list/get) need `Read`. Mutation tools (`create_*`, `configure_*`, `queue_*`, `abort_*`, `connect_*`) need `EditRun`. Destructive tools (`delete_*`, `clear_job`) need `Manage`.

## Tool Reference

### Discovery (read)

| Tool | Purpose |
|------|---------|
| `list_projects` | All projects the current user can access |
| `list_spaces(projectId)` | Spaces in a project |
| `list_views(projectId, spaceId)` | Views in a space (needed for `create_job`) |
| `list_jobs(projectId, spaceId)` | All jobs with their status |
| `get_job(projectId, spaceId, jobId)` | Full job detail: parameters, ports, connections |
| `list_job_types()` | All registered job types with category path and GUID |
| `get_job_type(typeGuid)` | Parameter schema and port definitions for one type |
| `list_queues()` | Available local and cluster queues |

### Monitoring

| Tool | Purpose |
|------|---------|
| `get_job_stdout(projectId, spaceId, jobId, lines?)` | Raw stdout tail (progress bars collapsed) |
| `get_job_stderr(projectId, spaceId, jobId, lines?)` | Raw stderr tail |
| `get_job_log(projectId, spaceId, jobId, iteration?, lines?)` | Relay's cleaned per-iteration log; use when stdout is absent |
| `list_job_results(projectId, spaceId, jobId, iteration?)` | Downloadable output files for latest (or specified) iteration |
| `get_job_result_link(projectId, spaceId, jobId, port, name, iteration?)` | Absolute download URL for a named result |

### Mutation

| Tool | Permission | Purpose |
|------|-----------|---------|
| `create_project(alias?, emoji?)` | Project EditRun | New project |
| `delete_project(projectId)` | Project Manage | Delete project |
| `create_space(projectId, alias?, emoji?)` | Space EditRun | New space + auto view |
| `delete_space(projectId, spaceId)` | Space Manage | Delete space |
| `create_job(projectId, spaceId, viewId, typeGuid)` | Job EditRun | New job (Building status) |
| `configure_job(projectId, spaceId, jobId, parameters)` | Job EditRun | Set parameters by name→value map |
| `connect_jobs(projectId, spaceId, fromJobId, fromPort, toJobId, toPort)` | Job EditRun | Link output port to input port |
| `disconnect_jobs(...)` | Job EditRun | Remove an edge |
| `queue_job(projectId, spaceId, jobId, queueId?)` | Job EditRun | Submit to local or cluster queue |
| `abort_job(projectId, spaceId, jobId)` | Job EditRun | Abort running/queued job |
| `clear_job(projectId, spaceId, jobId)` | Job Manage | Reset to Building, keep parameters |
| `clone_job(projectId, spaceId, jobId, viewId?)` | Job EditRun | Copy job + connections |
| `delete_job(projectId, spaceId, jobId)` | Job Manage | Delete a job |

## Job Lifecycle

```
Building → Waiting → Staging → Running → Finalizing → Finished
              ↑                   ↑           ↑
           queue_job         abort_job   abort_job
                          (→ Aborting → Aborted)
                                              ↓
                                    clear_job resets to Building
```

A job must be in `Building` status to have its parameters or connections changed. `queue_job` transitions it to `Waiting`. You cannot reconfigure a job that is `Running` or `Finished` — use `clear_job` first, or `clone_job` to work on a copy.

## Job Type Catalog

Call `list_job_types()` at runtime to get the current list with GUIDs. The TypeCategory path shows where a job sits in the menu hierarchy (e.g. `"Tilt-series.Reconstruction.Tomograms"`).

**Frame-series (single-particle / 2D)**

| Category | TypeName |
|----------|---------|
| Frame-series.Import | Frame-series data set |
| Frame-series.Motion & CTF | Motion correction, CTF estimation, Motion & CTF |
| Frame-series.Picking | BoxNet particle picking |
| Frame-series.Extraction | Extract particles |

**Tilt-series (tomography / 3D)**

| Category | TypeName |
|----------|---------|
| Tilt-series.Import | Tilt-series data set |
| Tilt-series.CTF | CTF estimation |
| Tilt-series.Alignment | AreTomo 2, Etomo alignment, MISS, Auto-level, Peak alignment, Stack tilts, Import alignments |
| Tilt-series.Selection | Select tomograms, Deselect tilts, Select particles |
| Tilt-series.Reconstruction | Reconstruct tomograms, Reconstruct map, Denoise tomograms |
| Tilt-series.Picking | Template matching |
| Tilt-series.Extraction | Extract particles |

**Refinement (RELION-based)**

| Category | TypeName |
|----------|---------|
| Refinement.Initial model | Initial reference |
| Refinement.2D classes | 2D classification, 2D class selection |
| Refinement.3D classes | 3D classification, Continue 3D, Supervised 3D, 3D class selection |
| Refinement.3D refinement | 3D refinement |
| Refinement.Masks | Create mask |
| Refinement.Post-process | Post-processing |

**M (multi-particle / Warp-based)**

| Category | TypeName |
|----------|---------|
| M | Create data source, Create population, Create species, Get species, Modify species, Get tilt series, Estimate weights, M refinement |

**Common**

| Category | TypeName |
|----------|---------|
| Common.Import | Import particles, Import particle positions, Import map, Import mask |
| Common.Tools | Threshold statistics |
| Common.Notes | Note, Vibe |

## Typical Cryo-ET Pipeline

A full subtomogram averaging workflow in Relay has three broad phases: frame-series preprocessing, tilt-series processing and particle extraction, and then refinement (RELION followed by M).

### Phase 1 — Frame-series preprocessing

Raw data arrives as movie frames (one per tilt angle per tilt series), processed as a `DataSetFs`.

1. **Frame-series data set** (`Frame-series.Import`) — points to the directory of raw movie frames.
2. **Motion & CTF** (`Frame-series.Motion & CTF`) — motion-corrects each frame and estimates per-micrograph CTF. This produces the motion-corrected averages and CTF parameters that the tilt-series importer needs.

### Phase 2 — Tilt-series processing

3. **Tilt-series data set** (`Tilt-series.Import`) — reads a folder of mdoc files. Each mdoc defines one tilt series and references the motion-corrected frame averages from Phase 1 by filename. Tilts are sorted by acquisition time (for dose accumulation), then by angle for storage. Bad tilts (too dark, too tilted, too much masked area) are culled at import.
4. **Etomo alignment** (`Tilt-series.Alignment`) — initial patch-tracking alignment via IMOD. Produces per-tilt 2D shifts and a tilt-axis angle. This is the starting geometry all later steps build on.
5. **Auto-level** (`Tilt-series.Alignment`) — estimates the specimen's out-of-plane inclination (physical tilt of the sample around the X and Y axes, i.e. a pre-tilted stage) and folds a geometric correction into the alignment so the reconstructed slab is level.
6. **MissAlignment** (`Tilt-series.Alignment`) — self-supervised deep learning refinement of the per-tilt image registration. Trains a small 3D CNN to score reconstruction quality by deliberately introducing synthetic misalignments ("making it worse") and learning to distinguish aligned from misaligned reconstructions — no external reference or picked particles required. The trained scorer is then used as a differentiable objective: L-BFGS optimizes per-tilt shifts and local warp fields end-to-end through the reconstruction operator to maximize predicted quality. Run after the initial Etomo + Auto-level alignment; produces a cleaner alignment used for all subsequent reconstruction and CTF steps.
7. **CTF estimation** (`Tilt-series.CTF`) — fits one defocus value per tilt using the solved geometry. Includes a defocus handedness check (verifies the sign of the defocus gradient across the tilt axis matches the reconstruction geometry).
8. **Select tomograms** (`Tilt-series.Selection`) — curate which tomograms pass quality before reconstruction.
9. **Reconstruct tomograms** (`Tilt-series.Reconstruction`) — weighted back-projection into full 3D tomograms. These are used for **picking and visualization only**; the final high-resolution map is built from per-particle tilt images, not these tomograms.
10. **Template matching** (`Tilt-series.Picking`) — CTF-aware 3D template matching against the reconstructed tomograms to locate candidate particle positions (PositionSet).
11. **Select particles** (`Tilt-series.Selection`) — filter picks by template-matching score or other criteria.
12. **Extract particles** (`Tilt-series.Extraction`) — extracts per-particle image series from the aligned tilt images (not the tomogram) at the target pixel size, producing a ParticleSet and a RELION optimisation set for downstream refinement.

### Phase 3a — RELION refinement

13. **Initial reference** (`Refinement.Initial model`) — generates or imports a starting 3D model.
14. **2D classification** (`Refinement.2D classes`) — optional; identifies and removes bad particle classes.
15. **3D classification** (`Refinement.3D classes`) — sorts heterogeneous particles into structural classes.
16. **3D refinement** (`Refinement.3D refinement`) — high-resolution alignment of the selected class.
17. **Post-processing** (`Refinement.Post-process`) — masking, B-factor sharpening, FSC calculation. At this point MissAlignment (step 6) can also be run using the RELION output as the reference, then CTF re-estimated, and RELION re-run for a better starting point before M.

### Phase 3b — M multi-particle refinement

M jointly refines beam-induced motion, CTF, and particle poses across the entire dataset, going back to the original tilt movies for higher resolution than RELION alone. The model is: a **Population** contains one or more **Data Sources** (tilt-series processing projects) and one or more **Species** (a particle class being refined).

18. **Create population** (`M.Create population`) — creates the M project container.
19. **Create data source** (`M.Create data source`) — registers the Warp tilt-series processing project with M, linking it back to the original tomostar files and frame data.
20. **Create species** (`M.Create species`) — imports the particle set and RELION half-maps into M, establishing the species to be refined. Particle positions are matched to data sources by tilt-series name.
21. **M refinement** (`M.Refine`) — iteratively refines particle poses, per-tilt CTF, per-frame motion, stage angles, and other optical parameters jointly across the full dataset.
22. **Estimate weights** (`M.Estimate weights`) — estimates per-tilt and per-frame exposure weighting from the data, then M refinement is run again with updated weights.

Connect jobs in order: output port resource type must match the downstream input port resource type. Use `get_job_type(typeGuid)` to inspect a job's declared ports before connecting.

## Key Resource Types

| ResourceType | Description |
|-------------|-------------|
| `DataSetTs` | Tilt-series dataset (raw movies + metadata) |
| `DataSetFs` | Frame-series dataset (2D micrographs) |
| `MicrographSet` | Motion-corrected 2D images |
| `ParticleSet` | Extracted particles with coordinates |
| `PositionSet` | 3D coordinates (picking results) |
| `TemplateSet` | Reference volumes for template matching |
| `Map` | 3D volume with half-maps, masks, FSC |

## Building a Pipeline: Step-by-Step

```
1. list_projects()                          → pick projectId
2. create_space(projectId, alias)           → spaceId (View 1 auto-created)
3. list_views(projectId, spaceId)           → viewId
4. list_job_types()                         → find typeGuid for each step
5. get_job_type(typeGuid)                   → inspect parameters and ports
6. create_job(projectId, spaceId, viewId, typeGuid)  → jobId
7. configure_job(projectId, spaceId, jobId, {ParameterName: value, ...})
8. connect_jobs(..., fromPort, ..., toPort) → edges must match resource types
9. list_queues()                            → pick queueId (omit for local)
10. queue_job(projectId, spaceId, jobId, queueId?)
11. Poll: get_job() → watch Status field (Running → Finished or Failed)
12. list_job_results() → get_job_result_link() to download outputs
```

## Common Mistakes

**Wrong port type** — `connect_jobs` will error if `fromPort` and `toPort` have different `ResourceType`. Always call `get_job_type` first to verify port types.

**Job not in Building** — `configure_job` and `connect_jobs` fail on jobs that are Waiting, Running, etc. Call `clear_job` (needs Manage) to reset, or `clone_job` for a fresh copy.

**No view exists** — `create_job` needs a `viewId`. `create_space` auto-creates one view, but always call `list_views` to get its id before calling `create_job`.

**Skipping CTF before picking** — CTF estimation must run before template matching in cryo-ET workflows; the CTF data is embedded in the dataset and required for correlation scoring.

**Monitoring the wrong log** — iterative jobs (classification, refinement) write per-iteration logs; use `get_job_log` with an explicit `iteration`. For single-pass jobs use `get_job_stdout`.