# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP524 | Analyze and Reason on Multimodal Data with Gemini: Challenge Lab | **started and halted at 30/100, do not start again without reading its section below** |

## What is already known about each

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
