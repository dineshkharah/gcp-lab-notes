# GSP541, Use Agent Skills with Multi-Agent Systems: Challenge Lab

Six tasks, **six** checkpoints, ninety minutes stated and **5 credits**, marked Advanced. The window was **2h14m**, and the work took about fifty minutes of which twenty five were Cloud Build. **Entirely Cloud Shell**; the console appears only as an optional log viewer.

New lab, manual and last tested both July 2026.

## The scoring surface is a bucket, not the running system

**Checkpoints 1 to 4 read files you copy into `gs://PROJECT_ID-agent-bucket`**, at the same relative paths as the repo. Nothing needs to execute for them to pass. So an edit that is not uploaded scores zero, and an edit made after uploading scores zero too.

Weights are uneven: **15, 15, 15, 15, 20, 20**.

**Checkpoint 5 is split in half.** Uploading a correct `run_local.sh` scored **10 of 20**, and the other 10 came only after actually running the local simulation and putting a topic through the web interface. That is the first checkpoint in these notes to award partial credit inside a single objective, and it is worth knowing because 10 of 20 looks like a wrong answer rather than a half done task.

Checkpoint 6 reads the deployed Cloud Run services.

## Do the edits from the terminal, not the Editor

The lab walks you through eight files in the Cloud Shell Editor. Doing it from the terminal takes one paste and is verifiable. Short files get rewritten whole, which is idempotent; the Python files get exact string replacements.

**The two `sub_agents=[None]` lines in `orchestrator/agent.py` are textually identical**, and only their trailing comments differ:

```
    sub_agents=[None], # TODO: Add research, judge, and escalation checker agents
    sub_agents=[None], # TODO: Add LoopAgent followed by ContentBuilder agent
```

So match on the whole line including the comment, or use line numbers. A naive `sed 's/sub_agents=\[None\]/.../'` writes the same value into both.

**Verify before uploading**, which costs seconds and saves a build cycle:

```
grep -rn "None\|TODO\|YOUR_" agents/*/agent.py run_local.sh
for f in agents/*/agent.py agents/content_builder/skills/scripts/check_format.py; do python3 -m py_compile "$f" && echo "OK $f"; done
bash -n run_local.sh && echo "OK run_local.sh"
```

The `grep` will still show the lab's own instructional comments containing TODO, plus real `-> None:` and `AsyncGenerator[Event, None]` annotations. What must be gone is `tools=[None]`, `output_schema=None`, `sub_agents=[None]` and every `YOUR_`.

## The fills

Everything except `check_format.py` is named in the lab text.

| File | Change |
|---|---|
| `researcher/skills/SKILL.md` | append the supplied paragraph forbidding raw function calls |
| `researcher/agent.py` | `tools=[researcher_toolset],` |
| `judge/agent.py` | `output_schema=JudgeFeedback,` and both transfer flags to `True` |
| `content_builder/skills/scripts/check_format.py` | return `{"success": bool, "message": str}` |
| `content_builder/skills/SKILL.md` | instruct the agent to run `check_format` before finalising |
| `content_builder/agent.py` | `tools=[content_builder_toolset],` |
| `orchestrator/agent.py` | `after_agent_callback=create_save_output_callback("judge_feedback")`, `actions=EventActions(escalate=True)`, then the two `sub_agents` lists |
| `run_local.sh` | `$(gcloud config get-value project)`, `us-central1`, and the two localhost agent card urls |

`check_format.py` ships with `has_h1` and `has_h2` already computed and a bare `pass`, so only the return needs writing:

```python
    if has_h1 and has_h2:
        return {"success": True, "message": "Format is valid: found an H1 title and at least one H2 section heading."}
    missing = []
    if not has_h1:
        missing.append("an H1 title starting with '# '")
    if not has_h2:
        missing.append("at least one H2 section heading starting with '## '")
    return {"success": False, "message": "Format check failed. Missing " + " and ".join(missing) + "."}
```

**The two `True` flags in task 2 are not busywork.** An ADK agent with an `output_schema` cannot delegate, so leaving either `disallow_transfer_to_*` at its default contradicts the schema.

## Two locations, and they are different

```
REGION=asia-east1                 # where Cloud Run deploys
GOOGLE_CLOUD_LOCATION=us-central1 # where the Gemini calls go
```

Both come out of the same `.env`. Do not unify them; `run_local.sh` explicitly wants `us-central1` for its location while every deploy uses `$REGION`.

`REGION` and `MODEL` are randomised per run. `gemini-2.5-flash` and `asia-east1` on this one.

## The parallel deploy race, which cost one build cycle

Task 6 is five `gcloud run deploy --source` builds. The three specialists do not depend on each other, so running them concurrently should turn fifteen minutes into six. It half worked:

```
Creating Container Repository........failed
ERROR: (gcloud.run.deploy) ALREADY_EXISTS: Requested entity already exists
```

**All three tried to create the `cloud-run-source-deploy` Artifact Registry repository at once.** Judge won, researcher and content-builder aborted before building anything. Retrying both after the repository existed succeeded first time.

So the pattern that actually works is **deploy one, then parallelise the rest**:

```
gcloud run deploy judge --source agents/judge/ --region $REGION --allow-unauthenticated --memory 512Mi --quiet --set-env-vars ...
gcloud run deploy researcher --source agents/researcher/ ... > /tmp/r.log 2>&1 &
gcloud run deploy content-builder --source agents/content_builder/ ... > /tmp/cb.log 2>&1 &
wait
tail -n 6 /tmp/r.log /tmp/cb.log
```

Redirect each background build to its own log; three concurrent progress bars in one terminal are unreadable. `--quiet` handles the repository prompt the lab warns about.

The orchestrator and frontend stay sequential: the orchestrator needs the three specialist urls, the frontend needs the orchestrator's. Echo them bracketed before use, because an empty one yields an env var like `/a2a/agent/.well-known/agent-card.json` with no host and the service starts anyway.

## Smaller things

**`gcloud services enable` is denied** to the student account. Expected; the lab pre-enables Run, Cloud Build, Artifact Registry and Vertex AI, and the error is not a problem to solve.

**The service is `content-builder`, the source directory is `agents/content_builder/`.** Hyphen and underscore in the same command.

**Web Preview is port 8080** while `run_local.sh` starts agents on 8001 to 8004. The frontend is the thing on 8080; the lab never says so.

**`tail -3 file1 file2` fails** with `option used in invalid context`. Use `tail -n 3`.

**`describe` and the deploy output print different hostnames** for the same service, `researcher-pfba7lrn2a-de.a.run.app` versus `researcher-PROJECTNUMBER.asia-east1.run.app`. Both are valid aliases; the mismatch is not a sign anything went wrong.

## What the local run should show

In order, in the terminal running `run_local.sh`: the orchestrator called, `Saved output to state['research_findings']`, then the judge, then `[EscalationChecker] Feedback: {...}` with a status, possibly a second loop iteration, then the content builder calling `check_format`. `max_iterations=3` caps the loop.

All four agents plus the frontend reported `Application startup complete` on the first attempt, so the clean temporary runtime directory the script creates does its job and the loader crash the lab warns about did not occur.

## Related files

- `genai129-deploy-agent-with-adk-challenge.md` for ADK agents, `AgentTool` and editing supplied agent source.
- `gsp328-serverless-applications-cloud-run-challenge.md` and `gsp659-deploy-website-on-cloud-run.md` for Cloud Run deploys and the auth flags.
- `gsp515-explore-generative-ai-gemini-api-challenge.md` and `gsp525-enhance-gemini-model-capabilities-challenge.md` for the Gen AI SDK.
- `gotchas.md`, the parallel operations section, which came from this lab.
