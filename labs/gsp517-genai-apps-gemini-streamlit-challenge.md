# GSP517, Develop GenAI Apps with Gemini and Streamlit: Challenge Lab

Five tasks, **five** checkpoints, forty minutes stated and **5 credits**. About **twenty minutes** once the clone is handled properly.

**Task 1 is JupyterLab. Tasks 2 to 5 are one Cloud Shell script**, plus two browser tests that are not optional.

Manual last updated June 2026, lab last tested June 2026.

## Task 1, JupyterLab, and the 404 the lab warns about

Agent Platform > Notebooks > Workbench, the `generative-ai-jupyterlab` instance, Open JupyterLab, open `prompt.ipynb`, kernel **Python 3 (Local)**.

Two edits, and doing them in this order saves a full rerun:

1. **Cell 3: set `location` to the lab's region**, not `global`. The lab documents the 404 this causes and buries the fix in a note *after* telling you to run the cells.
2. **Cell 5: replace the prompt** with the lab's Japanese low sodium text.

Then Run All, then **Save**. The checkpoint reads the saved notebook, so an unsaved run scores nothing.

There is a shell route, since a Workbench instance is a Compute Engine VM you can `gcloud compute ssh` into and drive with `jupyter nbconvert --to notebook --execute --inplace`. Not worth it on a forty minute clock: it is more moving parts for less certainty that the saved notebook has the outputs the checkpoint wants.

## The clone is the first trap and it is not obvious

`GoogleCloudPlatform/generative-ai` is well over a gigabyte. A full clone takes minutes. And **`git clone ... 2>/dev/null`, to suppress an "already exists" error, also suppresses the progress output**, because git writes progress to stderr. The result is a blank terminal that looks hung.

```
[ -d ~/generative-ai ] || git clone --depth 1 https://github.com/GoogleCloudPlatform/generative-ai.git
```

`--depth 1` brought it down to 125 MiB and about twenty seconds. Nothing in the lab reads git history. Now a section in `gotchas.md`.

## What chef.py actually looks like

Worth recording, because two details determine how you edit it.

```
    15  PROJECT_ID = "GCP_PROJECT_ID"
    16  LOCATION = os.environ.get("GOOGLE_CLOUD_REGION", "global")
    21      text_model_flash = "GEMINI_FLASH_MODEL_ID"
    82  # Task 2.5
    83  # Complete Streamlit framework code for the user interface, add the wine preference radio button
    84  # https://docs.streamlit.io/library/api-reference/widgets/st.radio
    88  # Task 2.6
    89  # Modify this prompt with the custom chef prompt.
    90  prompt = f"""Why is the sky blue?"""
```

**The whole file is at module level**, no indentation, so the wine radio needs none either.

**The placeholder prompt opens and closes on one line.** Any edit script that looks for a closing `"""` on a *later* line crashes here. Anchor on the exact string instead.

## Line 16 is the finding

`LOCATION = os.environ.get("GOOGLE_CLOUD_REGION", "global")`.

The lab's task 5 table tells you to deploy with `--set-env-vars PROJECT=$PROJECT,REGION=$REGION`. **The application reads neither of those.** It reads `GOOGLE_CLOUD_REGION`, and the default is `global`, which is the same value the lab warns about in task 1.

Set only what the table lists and you get a service that deploys, answers `HTTP 200`, and fails the moment anyone presses the button. Set all three:

```
--set-env-vars=PROJECT=$PROJECT,REGION=$REGION,GOOGLE_CLOUD_REGION=$REGION
```

Also in `gotchas.md`, generalised: grep the source for `os.environ` before trusting a lab's environment variable table.

## Tasks 2 to 5 in one script

`set -e` and a script file rather than a paste, so a broken `chef.py` edit stops before it is uploaded, built, pushed and deployed.

```
cat > ~/run_gsp517.sh <<'OUTER'
#!/bin/bash
set -e
export PROJECT=YOUR_PROJECT_ID
export REGION=YOUR_REGION
export GOOGLE_CLOUD_REGION=YOUR_REGION
export AR_REPO=chef-repo
export SERVICE_NAME=chef-streamlit-app
cd ~/generative-ai/gemini/sample-apps/gemini-streamlit-cloudrun
cp chef.py chef.py.bak

python3 - <<'PYEOF'
import sys

WINE = r'''wine = st.radio(
    "What is your wine preference?",
    ("Red", "White", "None"),
    key="wine",
    horizontal=True,
)
'''

NEW_PROMPT = r'''prompt = f"""I am a Chef.  I need to create {cuisine} \n
recipes for customers who want {dietary_preference} meals. \n
However, don't include recipes that use ingredients with the customer's {allergy} allergy. \n
I have {ingredient_1}, \n
{ingredient_2}, \n
and {ingredient_3} \n
in my kitchen and other ingredients. \n
The customer's wine preference is {wine} \n
Please provide some for meal recommendations.
For each recommendation include preparation instructions,
time to prepare
and the recipe title at the beginning of the response.
Then include the wine paring for each recommendation.
At the end of the recommendation provide the calories associated with the meal
and the nutritional facts.
"""
'''

s = open('chef.py').read()
s = s.replace('GCP_PROJECT_ID', 'YOUR_PROJECT_ID')
s = s.replace('GEMINI_FLASH_MODEL_ID', 'gemini-2.5-flash')

anchor = '# https://docs.streamlit.io/library/api-reference/widgets/st.radio\n'
if anchor not in s:
    print("WINE ANCHOR NOT FOUND"); sys.exit(1)
s = s.replace(anchor, anchor + '\n' + WINE)

old = 'prompt = f"""Why is the sky blue?"""\n'
if old not in s:
    print("PROMPT ANCHOR NOT FOUND"); sys.exit(1)
s = s.replace(old, NEW_PROMPT)

open('chef.py', 'w').write(s)
print("CHEF.PY EDITED OK")
PYEOF

python3 -c "import ast;ast.parse(open('chef.py').read());print('SYNTAX OK')"
if grep -q "GCP_PROJECT_ID\|GEMINI_FLASH_MODEL_ID" chef.py; then echo "PLACEHOLDER STILL PRESENT"; exit 1; else echo "PLACEHOLDERS REPLACED"; fi

grep -qx 'google-cloud-logging' requirements.txt || echo 'google-cloud-logging' >> requirements.txt
cp Dockerfile Dockerfile.bak
sed -i 's/app\.py/chef.py/g' Dockerfile
grep -n "chef.py" Dockerfile
printf 'gemini-streamlit/\n__pycache__/\n' > .gcloudignore

gcloud storage cp chef.py gs://$PROJECT-generative-ai/
echo "===== CHECKPOINT 2 READY ====="

python3 -m venv gemini-streamlit
source gemini-streamlit/bin/activate
python3 -m pip install -q -r requirements.txt
nohup streamlit run chef.py --browser.serverAddress=localhost --server.enableCORS=false --server.enableXsrfProtection=false --server.port 8080 > ~/streamlit.log 2>&1 &
for i in $(seq 1 12); do sleep 5; if curl -s -o /dev/null http://localhost:8080; then echo "STREAMLIT UP"; break; fi; echo "waiting for streamlit ($i)"; done
echo "===== NOW GENERATE A RECIPE IN WEB PREVIEW BEFORE CLICKING CHECKPOINT 3 ====="

gcloud artifacts repositories create "$AR_REPO" --location="$REGION" --repository-format=Docker || echo "repo already exists"
gcloud builds submit --tag "$REGION-docker.pkg.dev/$PROJECT/$AR_REPO/$SERVICE_NAME"
echo "===== CHECKPOINT 4 READY ====="

gcloud run deploy "$SERVICE_NAME" --port=8080 --image="$REGION-docker.pkg.dev/$PROJECT/$AR_REPO/$SERVICE_NAME" --allow-unauthenticated --region=$REGION --platform=managed --project=$PROJECT --set-env-vars=PROJECT=$PROJECT,REGION=$REGION,GOOGLE_CLOUD_REGION=$REGION
RUN_URL=$(gcloud run services describe $SERVICE_NAME --region=$REGION --format='value(status.url)')
curl -s -o /dev/null -w "cloud run HTTP %{http_code}\n" $RUN_URL
echo "CLOUD RUN URL: $RUN_URL"
OUTER
bash ~/run_gsp517.sh
```

Roughly twelve minutes. The Cloud Build was **1m44s**, much faster than the eight the lab predicts, because `.gcloudignore` keeps the virtualenv out of the build context. Without it the context is a few hundred megabytes of site packages.

Two buckets, easy to conflate: you **download** `chef.py` from `PROJECT-gemini` and **upload** it to `PROJECT-generative-ai`.

## Checkpoint 3 wants a generation, not a running server

The one that cost points. The script produced `local HTTP 200` and checkpoint 3 still scored **0 of 20**.

`chef.py` calls `logging.info(response)` into Cloud Logging *after* a successful generation, and that entry is what the checkpoint reads. A rendered page writes nothing.

So: **Web Preview on port 8080**, choose a cuisine and a dietary preference (both dropdowns start at `index=None`), pick a wine option, click **Generate my recipes**, and wait for actual recipe text under "Your recipes:". Then click the checkpoint. It went straight to 100.

Another instance of the action versus end state note in `gotchas.md`. The same applies to task 5: generate a recipe at the Cloud Run URL rather than trusting `HTTP 200`.

Incidentally this is also the cheapest proof that `GOOGLE_CLOUD_REGION` took effect, since a `global` location is what produces the 404.

## The community script

It routes around the lab rather than through it, and six things are wrong:

- **It copies `chef.py` from a hardcoded `gs://spls/gsp517/chef.py` and then deletes it** on the next line, before downloading a **stranger's** `chef.py`, `Dockerfile` and `requirements.txt` from an unrelated repo. The lab's own path is randomised per run.
- A third party `chef.py` carries someone else's project id, which is the exact thing task 2 asks you to replace.
- **`--set-env-vars GCP_PROJECT=...,GCP_REGION=...`**, matching neither the lab's table nor what the code reads.
- **Those variables are never exported**, so the local Streamlit child process does not see them either.
- **`GCP_REGION` from `commonInstanceMetadata.items[google-compute-default-region]`**, frequently unset, and the lab gives you the region directly.
- **`requirements.txt` replaced rather than appended to**, where the lab asks you to add one line to the existing two.

It also downloads a prebuilt `prompt.ipynb` for task 1, which will carry that author's region in cell 3, the precise cause of the 404 the lab warns about.

## Related files

- `genai129-deploy-agent-with-adk-challenge.md` for the other generative AI build and deploy lab, including the same "run the script twice" hazard.
- `gsp328-serverless-applications-cloud-run-challenge.md` for Cloud Run deploys and the authentication flags.
- `gsp304-docker-image-to-kubernetes-challenge.md` and `gsp1131-artifact-registry-qwik-start.md` for the build and push cycle.
