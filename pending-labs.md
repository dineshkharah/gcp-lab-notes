# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP355 | Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab | analysed, never started |
| GSP421 | APIs Explorer: Cloud Storage | analysed, never started |
| GSP524 | Analyze and Reason on Multimodal Data with Gemini: Challenge Lab | **started and halted at 30/100, do not start again without reading its section below** |

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

### GSP421

The APIs Explorer version of the operations `labs/gsp074-cloud-storage-qwik-start-cli-sdk.md` does from the CLI. Introductory, no cost, fifteen minutes stated, about ten. **Browser lab, essentially zero Cloud Shell**, six tasks and six checkpoints. Manual and last tested both April 2026.

- **Task 3 is the only step that touches your own machine.** Two images get downloaded off the lab page as `demo-image1.png` and `demo-image2.png`, then uploaded through the Cloud Storage console. Everything else is the APIs Explorer web form.
- **Three checkpoints get destroyed by later tasks.** Task 5 deletes both images that checkpoint 3 scores, and task 6 deletes the whole first bucket. So checkpoints 3 and 4 have to be claimed before task 5, and 5 before 6. Same shape as GSP074, three checkpoints deep instead of one.
- **Both Credentials checkboxes**, Google OAuth 2.0 and API key, have to be ticked. The lab repeats it on every task because it is the usual cause of a 401 that reads as a permissions problem.
- **`buckets.delete` needs an empty bucket**, which bucket 1 only becomes after task 5. Running task 6 early gives a 409.
- Bucket names are globally unique, so prefix with the project id. No trailing spaces in the project field; a pasted project id often carries one.
- The request body carries only `name`, so the bucket takes the `US` multi region default, which is what the lab's expected response shows. Do not add a location.
- **Task 7 quiz answers:** default storage class specified at creation, **True**. Every bucket name unique across the whole namespace, **True**. The four storage classes, **Coldline, Multi-Regional, Regional, Nearline**; `Local storage` and `Cross region storage` do not exist.

### GSP524

**Halted deliberately on 2026-08-18 at 30/100, part way through task 2. Stop and read this whole section before starting it again.** It is the only entry here that was abandoned mid run rather than never begun, and the reason it was abandoned is that its notebook ships broken.

Entirely JupyterLab, zero Cloud Shell. **Eight checkpoints**, twenty five minutes stated, and the twenty five minutes is not real.

| Value | |
|---|---|
| instance | `generative-ai-jupyterlab` |
| notebook | `gsp524-challenge.ipynb` |
| kernel | `Python 3` |
| model per the lab text | Gemini 3.5 Flash |

**The notebook's `MODEL_ID` does not work.** It ships as `gemini-2.0-flash-001` and every `generate_content` call returns:

```
ClientError: 404 NOT_FOUND. Publisher model
`projects/PROJECT/locations/europe-west4/publishers/google/models/gemini-2.0-flash-001`
was not found or your project does not have access to it.
```

**Checkpoint 1 passes anyway**, because it only verifies the imports and the client, not that the model id resolves. So the lab reads as working until the first real call in task 2. The circulating helper corrects it to **`gemini-2.5-flash`**, which is neither the shipped value nor the `3.5` the lab text advertises in all three topic lines. Probe before committing to either:

```python
from google import genai
c_test = genai.Client(vertexai=True, project="PROJECT_ID", location="REGION")
for m in ["gemini-3.5-flash", "gemini-3.5-flash-001", "gemini-2.5-flash", "gemini-2.0-flash"]:
    try:
        c_test.models.generate_content(model=m, contents="ping")
        print("WORKS:", m)
    except Exception as e:
        print("no:", m, "|", str(e)[:90])
```

Then change it **at its definition in task 1** and rerun that cell, not in a patch cell lower down. Eight sections read `MODEL_ID`, and a later rerun of the setup section silently restores the broken value.

**Do not touch `LOCATION`.** It is `os.environ.get("GOOGLE_CLOUD_REGION", "us-central1")`, the same shape as the GSP517 trap, but the 404 above already proves the env var is set, so the region is not the fault. Changing it introduces a second problem.

The eight checkpoints:

| # | Section |
|---|---|
| 1 | task 1 setup |
| 2 | task 2 initial analysis, text |
| 3 | task 2 deep dive, thinking |
| 4 | task 3 initial analysis, images |
| 5 | task 3 reasoning on image trends, thinking |
| 6 | task 4 initial analysis, audio |
| 7 | task 4 reasoning on audio insights, thinking |
| 8 | task 5 synthesis |

Three shapes, repeated ten times. A prompt string, a `generate_content` call, and for every second half of a task the thinking config:

```python
config=types.GenerateContentConfig(
    thinking_config=types.ThinkingConfig(
        include_thoughts=True,
        thinking_budget=1024
    )
)
thinking_model_response = client.models.generate_content(
    model=MODEL_ID, contents=thinking_mode_prompt, config=config
)
```

**Tasks 2, 3 and 4 each write a markdown file under `analysis/`**, starting with `analysis/text_analysis.md`, and **task 5 reads all three back**. The sections are not independent, so task 5 fails for reasons that look like a bad prompt if an earlier section never ran.

**The media is local, not `gs://`.** Setup copies it down:

```
!gcloud storage cp gs://{PROJECT_ID}-bucket/media/text/reviews.txt media/text/reviews.txt
!gcloud storage cp -r gs://{PROJECT_ID}-bucket/media/images media/
```

Five images, `casual_coffee.png`, `mountain_climber.png`, `trail_runner.png`, `urban_gym.png`, `yoga_studio.png`. So no `Part.from_uri` and no `gs://` versus `https` trap, unlike GSP520.

Two loose ends to pick up on a fresh run:

- **The task 3, 4 and 5 fills were never obtained.** The helper notebook truncated on fetch past task 2, so the image, audio and report cells are unknown. Get those cells first next time.
- **The kernel restarted twice during the halted run**, which wipes `client`, `MODEL_ID` and `text_data` while leaving the notebook file intact. The symptom is `NameError: name 'client' is not defined` and the fix is rerunning task 1, not rewriting anything.

The helper notebook is named `gsp524-challenge-v1.0.0.ipynb` against the lab's `gsp524-challenge.ipynb`. A wholesale swap would leave a filename the checkpoints do not read, and deleting the original first is the trap in `gotchas.md` with no undo.

## Not pending

- **GSP364**, Managed Service for Prometheus: Challenge Lab, already has a file at `labs/gsp364-managed-prometheus-challenge.md` and will not be repeated. Check the README index before starting anything that feels familiar; it caught that one before a lab attempt was spent.
