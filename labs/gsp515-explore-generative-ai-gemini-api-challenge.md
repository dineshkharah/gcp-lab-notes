# GSP515, Explore Generative AI with the Gemini API in Agent Platform: Challenge Lab

Four tasks, **three** checkpoints, twenty five minutes stated and **5 credits**. About **fifteen minutes**.

**Task 1 is Cloud Shell, tasks 2 to 4 are a Jupyter notebook** in Agent Platform Workbench. Task 2 is just opening the notebook and has no checkpoint of its own.

Manual last updated and last tested August 2026, the freshest lab in these notes.

## Task 1, the curl

The lab gives you the variables and points you at the *Send Chat Prompts to Gemini* documentation rather than the command:

```
PROJECT_ID=YOUR_PROJECT_ID
LOCATION=global
API_ENDPOINT=aiplatform.googleapis.com
MODEL_ID="gemini-3.5-flash"

gcloud services enable aiplatform.googleapis.com

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://${API_ENDPOINT}/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/google/models/${MODEL_ID}:generateContent" \
  -d '{"contents":{"role":"user","parts":{"text":"Why is the sky blue?"}}}'
```

Two things about that.

**The API enable is a step the lab only hints at**, saying *"You can perform this step in Cloud Console in the Agent Platform section of the UI."* One `gcloud services enable` is faster and leaves a clearer trail than clicking through the console.

**`LOCATION=global` is correct here**, and the endpoint is the plain `aiplatform.googleapis.com` rather than a region prefixed one. Worth flagging against `gsp517-genai-apps-gemini-streamlit-challenge.md`, where `global` was the *default* and produced a 404 that the lab warned about. Here it is what the lab specifies and it works. Follow the lab in front of you rather than carrying the fix across.

The model is **`gemini-3.5-flash`**, newer than the `gemini-2.5-flash` in the neighbouring labs.

## Tasks 3 and 4, the notebook

Agent Platform > Notebooks > Workbench, the `generative-ai-jupyterlab` instance, Open JupyterLab, then `gemini-explorer-challenge.ipynb` with the **Python 3 (Local)** kernel.

The blanks are marked `INSERT` rather than `TODO`. Five fills across both tasks:

| Where | Fill |
|---|---|
| Task 3, declaring the function | `FunctionDeclaration` |
| Task 3, wrapping it | `Tool` |
| Task 3, the generate call | `tools=[weather_tool]` |
| Task 4, the video input | `Part.from_uri` |
| Task 4, the generate call | `models.generate_content_stream` |

That is the whole of the code content. `FunctionDeclaration` describes the function's name, description and parameter schema; `Tool` wraps one or more declarations; `tools=[weather_tool]` passes it into the request. On the video side, `Part.from_uri` builds a content part from a `gs://` uri with a mime type, and `models.generate_content_stream` streams the response back.

**Run the Getting Started section first.** Project and location are pre configured, so it needs no edits, but nothing after it works until the client exists.

## Two notes the lab gives that are worth heeding

**429 means wait, not broken.** *"If you experience a 429 response from any of the notebook cell executions, wait one minute before running the cell again."* Rate limiting, and rerunning immediately makes it worse.

**Task 3's checkpoint wants the weather data visible.** *"Ensure you can see the weather related data in the response that is printed."* Same species as GSP517's task 3: a cell that executes without error but prints nothing useful is not the same as a cell that produced a result. Check the output before clicking.

## Related files

- `gsp517-genai-apps-gemini-streamlit-challenge.md` for the same API from a Streamlit app, and for the `global` versus region distinction.
- `genai129-deploy-agent-with-adk-challenge.md` for tools and function calling at the agent level rather than a single request.
- `gsp527-gemini-code-assist-challenge.md` for the other Gemini badge lab.
