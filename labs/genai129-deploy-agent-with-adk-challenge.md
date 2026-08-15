# GENAI129, Deploy an Agent with Agent Development Kit (ADK): Challenge Lab

Six tasks, **five** checkpoints, two and a half hours stated and **7 credits**. Marked Advanced.

**Unusual scoring: 80% is the pass mark**, not 100%. This run finished at **90% with task 6 skipped**, which is a genuinely useful thing to know before starting, because task 6 is a nine turn conversation with a deployed agent and is worth skipping if the clock is short.

**Task 1 is console. Everything else is Cloud Shell**, but three tasks require you to *talk to the agent*, so it cannot be pasted end to end.

Manual last updated and last tested July 2026.

## Shape of it

| Task | Where | Notes |
|---|---|---|
| 1 Agent Search app and data store | console | then 10 to 20 min indexing |
| 2 install ADK, write `.env` | shell | 3 min |
| 3 fix the `AgentTool` bug | shell edit, then a terminal chat | |
| 4 implement session state | shell edit, then a browser chat in `adk web` | |
| 5 deploy to Agent Runtime | shell | 5 to 10 min |
| 6 query from Chainlit | shell, then a browser chat | skipped here |

**Two waits dominate**: data store indexing and `adk deploy`. Start task 1 first and do the task 3 and task 4 code edits while it indexes, since both edits are static and only their testing needs the data store.

Do task 1 in the console rather than through the Discovery Engine API. The graded settings, **Layout Parser**, **table annotation** and **ancestor headings in chunks**, live in `documentProcessingConfig`, and getting that JSON wrong means reimporting the PDF and waiting out indexing again.

## Task 2

```
export PATH=$PATH:"/home/${USER}/.local/bin"
export PROJECT=$(gcloud config get-value project)
export BUCKET=$PROJECT-bucket
gcloud storage cp -r gs://$BUCKET/adk_challenge_lab .
python3 -m pip install -q -r adk_challenge_lab/requirements.txt
python3 -m pip install -q chainlit==2.11.1
cd ~/adk_challenge_lab
```

Then the `.env`, with the search engine ID from Agent Platform > Agents > Search, which looks like `paint-search_1786818101590`:

```
cd ~/adk_challenge_lab
cat > .env <<'EOF'
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=us-central1
RESOURCES_BUCKET=YOUR_PROJECT_ID-bucket
MODEL=gemini-2.5-flash
SEARCH_ENGINE_ID=YOUR_SEARCH_ENGINE_ID
EOF
cp .env paint_agent/.env
diff .env paint_agent/.env && echo "BOTH COPIES MATCH"
```

**Both copies matter.** The root one is read by `adk run` and `adk web`; the `paint_agent/` one is packaged and read by the **deployed** agent. An agent that works locally and fails after deployment is usually the second copy missing.

**Expect a dependency conflict** from the install, because the lab has you add `chainlit` after `requirements.txt`:

```
google-adk 1.38.0 requires opentelemetry-api<=1.41.1, but you have opentelemetry-api 1.44.0
```

It is harmless in practice, and everyone taking this lab gets it. Only if `adk run` dies on an OpenTelemetry import is it worth pinning `opentelemetry-api==1.41.1` and `opentelemetry-sdk==1.41.1`, and that risks breaking Chainlit for task 6.

## Task 3, the bug and its two halves

The error the lab shows you:

```
400 INVALID_ARGUMENT ... 'Multiple tools are supported only when they are all search tools.'
```

`search_agent` uses a `VertexAiSearchTool`. Listing it under `sub_agents` gives the root agent an implicit `transfer_to_agent` tool alongside `set_session_value`, and a search tool cannot coexist with non search tools. Wrapping it in `AgentTool` isolates it.

**`AgentTool` is already imported** in `paint_agent/agent.py`, so this is two list edits:

```
cd ~/adk_challenge_lab
cp paint_agent/agent.py paint_agent/agent.py.bak
sed -i 's/    sub_agents=\[search_agent, room_planner_agent\],/    sub_agents=[room_planner_agent],/' paint_agent/agent.py
sed -i 's/^        set_session_value,$/        set_session_value,\n        AgentTool(agent=search_agent, skip_summarization=False),/' paint_agent/agent.py
sed -n '/before_model_callback/,$p' paint_agent/agent.py
```

Target state:

```
    sub_agents=[room_planner_agent],
    tools=[
        set_session_value,
        AgentTool(agent=search_agent, skip_summarization=False),
    ],
```

**Both halves are required.** Adding the `AgentTool` while leaving `search_agent` in `sub_agents` reproduces the original error exactly, because the implicit transfer tool is still there.

**That second `sed` appends, so it is not safe to run twice.** It ran twice here and produced a duplicated `AgentTool(...)` line, with no error. The `cp` at the top of the block is what fixed it, since the second pass backed up the correctly edited file before breaking it. Now a section in `gotchas.md`.

Then, once the data store shows **Ready** plus a few extra minutes, `adk run paint_agent` and ask it for the prices of EcoGreens and Forever Paint. That conversation is checkpoint 3. `exit` to leave.

## Task 4, two edits

`paint_agent/tools.py` ships as a stub returning `{"status": "tool not implemented"}`. Eleven lines, so rewrite it whole rather than patching:

```
cat > paint_agent/tools.py <<'EOF'
from dotenv import load_dotenv
import os
from google.adk.tools import ToolContext

load_dotenv()


async def set_session_value(tool_context: ToolContext, key: str, value: str):
    """Sets a value in the tool_context's state dictionary."""

    tool_context.state[key] = value
    return {"status": f"stored '{value}' in '{key}'"}
EOF
```

**Quoted heredoc**, so the f-string braces and inner quotes survive. The status string is worded exactly as the lab specifies, in case the checkpoint reads it.

Then the coverage calculator's instruction, six directories down:

```
export CC=paint_agent/sub_agents/room_planner/sub_agents/coverage_calculator/agent.py
sed -i 's/is COVERAGE_RATE,/is {COVERAGE_RATE?},/; s/The price is PRICE per bucket/The price is {PRICE?} per bucket/' $CC
grep -n "COVERAGE_RATE\|PRICE" $CC
```

**Only two terms in that file are ALL CAPS state keys**, `COVERAGE_RATE` and `PRICE`. The `X liters`, `Y buckets` and the `2,4-2,7M` ceiling height are not, and substituting them would break the instruction. The lab's phrasing, "replace terms in ALL CAPS", reads broader than it is.

**The `?` is load bearing.** `{COVERAGE_RATE?}` is the optional form; without it the agent raises when the key is unpopulated, which is exactly the state on turn one.

Test with the lab's seven turn script in:

```
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

The **State** tab is the thing to watch: after "I'd like to use EcoGreens" it should show `SELECTED_PAINT`, `COVERAGE_RATE` and `PRICE`. Expect 74 sq meters at the end. `CTRL+C` to stop the server.

## Task 5, deploy and the two roles

```
cd ~/adk_challenge_lab
adk deploy agent_engine --display_name "Paint Agent" .
```

The **`.`** is required; the lab calls it out as "the agent directory as the final positional argument". Five to ten minutes, and the resource name prints at the end. **Copy it**, task 6 needs it.

While it deploys, the two grants. The lab gives display names and the API wants ids:

| Lab's name | Role id |
|---|---|
| Agent Platform User | `roles/aiplatform.user` |
| Discovery Engine User | `roles/discoveryengine.user` |

```
export SA=service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:$SA --role=roles/aiplatform.user
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:$SA --role=roles/discoveryengine.user
gcloud projects get-iam-policy $PROJECT --flatten="bindings[].members" --filter="bindings.members:$SA" --format="value(bindings.role)"
```

Note the service account domain is **`gcp-sa-aiplatform-re`**, with the `-re` suffix, not the plain `gcp-sa-aiplatform` you may be used to. The lab prints the full address; use it verbatim.

## Task 6, not attempted

Recorded from the lab text rather than from a run, so treat as untested.

Edit `chainlit_ui/app.py`, replacing `YOUR_AGENT_RESOURCE_NAME` in:

```
agent = client.agent_engines.get(name='YOUR_AGENT_RESOURCE_NAME')
```

then:

```
cd ~/adk_challenge_lab/chainlit_ui
chainlit run app.py
```

and the lab's nine turn conversation on the web preview at port 8000, ending at 77 and 53 sq meters.

**Skipping it cost 10 points and the lab still passed at 90 against an 80 threshold.** Worth knowing, but do it if the clock allows, since the deploy is already paid for by then.

## Related files

- `gsp527-gemini-code-assist-challenge.md` for the other generative AI challenge lab in these notes.
- `gsp1077-gke-pipeline-using-cloud-build.md` and `gsp328-serverless-applications-cloud-run-challenge.md` for the deploy and query pattern on non agent workloads.
