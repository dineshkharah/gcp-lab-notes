# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP355 | Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab | analysed, never started |
| GSP525 | Enhance Gemini Model Capabilities: Challenge Lab | analysed, never started |
| GSP523 | Implement Multimodal Vector Search with BigQuery: Challenge Lab | analysed, never started |
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

### GSP525

Entirely a Jupyter notebook on Agent Platform Workbench, `#TODO` blanks in four sections. No Cloud Shell at all, four checkpoints all reading notebook state.

- **Task 1** installs the Gen AI SDK, imports, and a `genai.Client` for Vertex.
- **Task 2** a `Tool` wrapping `ToolCodeExecution()`, passed through `GenerateContentConfig`.
- **Task 3** a `Tool` wrapping `GoogleSearch()`, with **Nike Air Jordan XXXVI** as the product to search for.
- **Task 4** a response schema plus `response_mime_type="application/json"` and `response_schema=`.
- Check whether the notebook's location variable defaults to `global`, per the note in `gotchas.md`. GSP517 hit a 404 that way and this calls the same API.
- Send the TODO cells as text before filling them; this SDK has changed shape more than once.

### GSP523

BigQuery SQL plus one external connection and three IAM grants. Doable entirely from Cloud Shell or entirely in the BigQuery SQL editor; the choice barely matters. Four tasks, four checkpoints, fifteen minutes stated, expect **twenty to twenty five**, nearly all of it generating embeddings over the images.

**Task 1** is the only non SQL part:

```
bq mk --connection --location=REGION --connection_type=CLOUD_RESOURCE vector_conn
bq show --connection --format=json PROJECT.REGION.vector_conn
```

The `show` output carries the auto generated service account, which needs three roles. **The lab gives display names and the api wants ids**, the same mismatch as GSP373 and GENAI129:

| Lab's name | Role id |
|---|---|
| BigQuery Data Owner | `roles/bigquery.dataOwner` |
| Storage Object Viewer | `roles/storage.objectViewer` |
| Agent Platform User | `roles/aiplatform.user` |

**Tasks 2 to 4** are three SQL statements with bracketed blanks. Most are mechanical substitutions of the project, the dataset `gcc_bqml_dataset`, and the table names the lab supplies. **Four blanks are actual knowledge:**

- `[DEFINE_ENDPOINT]` the model endpoint, given as `MODEL_NAME` on the started page
- `[EMBEDDINGS_FUNCTION]` is `ML.GENERATE_EMBEDDING`
- `[VECTOR_SEARCH_FUNCTION]` is `VECTOR_SEARCH`
- `[STATEMENT_TO_SELECT_TOP_2_RESULTS]` is `top_k => 2`, a named argument in a table function's argument list, not a `LIMIT`

Three things likeliest to cost time:

- **IAM propagation.** The connection's service account is brand new and the grants take a minute or two to work. Task 2's object table create is the first thing to exercise them, so a permission error there means wait and retry rather than regrant. Same family as the service agent note in `gotchas.md`.
- **Region consistency.** The connection region, the dataset region and the model must line up. A connection in the wrong region errors at the object table step in a way that reads like a missing connection.
- **The bucket is named after the project**, `uris=['gs://PROJECT_ID/*']`, as in ARC119 and GSP305. Confirm with `gcloud storage ls` rather than assuming.

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
