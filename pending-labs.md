# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP355 | Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab | analysed, never started |
| GSP525 | Enhance Gemini Model Capabilities: Challenge Lab | analysed, never started |
| GSP1210 | Multimodality with Gemini | analysed, never started |

## What is already known about each

### GSP355

The PostgreSQL twin of `labs/gsp351-migrate-mysql-to-cloud-sql-dms-challenge.md`, using Database Migration Service with **VPC peering** and a **continuous** job.

- **Split:** the pglogical prep on `postgres-vm` and tasks 2 to 4 are Cloud Shell; the DMS connection profile and migration job are console, for the same reason as GSP351, that the CLI's destination expects a profile that provisions a *new* Cloud SQL instance while the lab requires **Existing instance**.
- **The first value to read off the started page is the PostgreSQL major version.** The unstarted text leaves it as an unrendered template, and the package name `postgresql-VERSION-pglogical` plus both config paths under `/etc/postgresql/VERSION/main/` depend on it. One earlier run showed 14; do not assume it.
- **Order matters for pglogical.** Install the package, edit `postgresql.conf`, restart, *then* `CREATE EXTENSION`. The extension needs `shared_preload_libraries = 'pglogical'` already loaded, and doing it in the intuitive order gives an error that reads like a bad install.
- **The migration user name has spaces**, `Postgres Migration User`, so every SQL reference needs double quotes and the DMS username field takes it literally.
- **The primary key requirement is the silent failure.** Five tables need keys and the lab warns a badly prepared source "might fail to migrate some individual tables", so a migration that copies four of five looks green.
- **Two restarts** that cannot overlap the migration: enabling the IAM auth database flag, and enabling point in time recovery.
- Task 4 needs a UTC RFC 3339 timestamp taken *before* the row is added, and the clone must be named `postgres-orders-pitr` and left in place.

### GSP525

Entirely a Jupyter notebook on Agent Platform Workbench, `#TODO` blanks in four sections. No Cloud Shell at all, four checkpoints all reading notebook state.

- **Task 1** installs the Gen AI SDK, imports, and a `genai.Client` for Vertex.
- **Task 2** a `Tool` wrapping `ToolCodeExecution()`, passed through `GenerateContentConfig`.
- **Task 3** a `Tool` wrapping `GoogleSearch()`, with **Nike Air Jordan XXXVI** as the product to search for.
- **Task 4** a response schema plus `response_mime_type="application/json"` and `response_schema=`.
- Check whether the notebook's location variable defaults to `global`, per the note in `gotchas.md`. GSP517 hit a 404 that way and this calls the same API.
- Send the TODO cells as text before filling them; this SDK has changed shape more than once.

### GSP1210

**A guided lab, not a challenge lab**, despite arriving alongside them. No `TODO` or `INSERT` blanks anywhere; every task says "run through the *X* section of the notebook". Entirely JupyterLab, zero Cloud Shell.

- **Nine checkpoints**, one per notebook section: multi image understanding, video description, audio understanding, reason across a codebase, video and audio, all modalities at once, recommendations from images, ER diagrams, image comparison.
- **Twenty five minutes stated, expect thirty to forty**, because multimodal inference over video and audio is slow and there are nine sections of it.
- **The constraint is rate limiting, not difficulty.** Do **not** Run All. Run one section, read its output, click its checkpoint, move on, which is how the lab is structured anyway. A 429 means wait a minute and rerun; rerunning immediately makes it worse.
- **The checkpoints almost certainly verify the API call happened**, as in GSP515 and GSP517, so a cell that errored and got skipped fails its checkpoint even though the notebook looks finished.
- Check the Getting Started cell's location variable is not `global` before running nine sections against it.
- **Lab text bug:** the "Reason across a codebase" section is described as *"Gemini can directly process audio for long-context understanding"*, copied from the Audio understanding section above it. Read the heading, not the description.

## Not pending

- **GSP364**, Managed Service for Prometheus: Challenge Lab, already has a file at `labs/gsp364-managed-prometheus-challenge.md` and will not be repeated. Check the README index before starting anything that feels familiar; it caught that one before a lab attempt was spent.
