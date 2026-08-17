# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP355 | Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab | analysed, never started |
| GSP520 | Inspect Rich Documents with Gemini Multimodality and Multimodal RAG: Challenge Lab | analysed, never started |

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

### GSP520

Entirely JupyterLab, zero Cloud Shell. Four checkpoints. **The heaviest of the notebook labs**: thirty five minutes stated, expect **forty five to sixty**, and the time goes on task 2 rather than task 1.

- **Save the notebook before every checkpoint.** The lab says so explicitly and it applies to all four: *"Save the notebook script before clicking on the Check my progress button for every task."* An unsaved run scores nothing.
- **Two setup steps, not one.** The Getting Started section, then *"the 4 cells in the Setup and requirements section"* before task 1. Skipping the second is the likeliest way to have task 1 fail for no visible reason.
- **The checkpoint mapping is uneven.** Checkpoint 1 covers one section, checkpoint 2 one section, **checkpoint 3 covers four** (video description, object tags, more questions, extra information beyond the video), checkpoint 4 covers all of task 2.
- **The video is given as an https url**, `https://storage.googleapis.com/spls/gsp520/google-pixel-8-pro.mp4`, and is used in three sections. Gemini on Vertex takes a **`gs://`** file uri for media, so if a cell feeds it to `Part.from_uri` it needs `gs://spls/gsp520/google-pixel-8-pro.mp4`. Read the surrounding cell before pasting the url as given.
- **Task 2 builds metadata by calling Gemini once per image**, across a 14 page Google 10K split into Part 1 and Part 2, full of tables, charts and graphs. That is minutes of inference and the most likely place to hit the 429s the lab warns about. Run it once and let it finish rather than rerunning.
- Task 2's helper functions are provided, so its fills are about calling them correctly rather than writing logic: `get_similar_text_from_query`, `print_text_to_text_citation`, `get_similar_image_from_query`, `print_text_to_image_citation`, `get_gemini_response`, `display_images`.
- The other document, the Terms of Service, is **text only**, so image related helpers do nothing against it.

## Not pending

- **GSP364**, Managed Service for Prometheus: Challenge Lab, already has a file at `labs/gsp364-managed-prometheus-challenge.md` and will not be repeated. Check the README index before starting anything that feels familiar; it caught that one before a lab attempt was spent.
