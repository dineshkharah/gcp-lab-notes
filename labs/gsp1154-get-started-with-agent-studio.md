# GSP1154, Get Started with Agent Studio

Five tasks, **five** checkpoints, fifty minutes stated and **1 credit**. **Entirely console**, no Cloud Shell and no commands anywhere. Completed by following the lab text.

Manual and last tested both **June 26 2026**, and the catalog card said updated four hours before this run, so this is the freshest lab in these notes by a wide margin.

## Two regions, and they are set in different places

| Setting | Value | Where |
|---|---|---|
| Model **Region** | **Global** | Model settings panel, every task |
| Save prompt **Region** | **us-central1** | the Save prompt dialog |

The lab specifies both and they are not the same. Fourth lab in these notes with two location settings in one flow, after GSP517, GSP541 and GSP524. Set the model region to Global as the tasks say and leave the save dialog on `us-central1`.

## Models used

`gemini-3.5-flash` throughout, except task 3's final comparison which puts **Gemini 2.5 Pro** on the right against **Gemini 3.5 Flash with Thinking level Minimal** on the left. Task 5's image generation uses **Nano Banana 2** at 1:1.

The Pro versus Flash comparison is the only place the lab predicts an outcome: Flash names the general hazard, Fire, while Pro isolates the specific one, the uncertified fire suppression system, and cites more of the supplied guidelines. Worth reading the two outputs rather than skipping ahead, since that contrast is the point of the task.

## Adding an Example clears the system instructions

The trap in task 2, and the lab does flag it, easily missed:

> *"Re-add System Instructions: Since clearing the prompt also cleared the system instructions, paste them again"*

Using **+ → Example** to add a few shot pair resets the System instructions box. So the sequence is: add the example, **then** paste the system instructions back, then the new input and the prompt. Doing it in the intuitive order leaves the model running without its instructions, and the output looks merely mediocre rather than broken.

## The Cloud Run deploy is expected to fail the first time

Task 1 deploys the saved prompt as an app, and the lab documents its own flakiness:

> *"The deployment process may occasionally fail on the first attempt. This typically happens if the underlying permissions for the build service have not fully propagated ... Wait for approximately one minute ... click the Update app button."*

**Retry with Update app, do not rebuild from scratch.** This is the same class of problem as the IAM propagation note in `gotchas.md`, and it is the first time a lab has prescribed the retry itself, which is useful confirmation that waiting is the fix rather than a workaround.

## Task by task

**Task 1**, *Create a prompt application with Agent Studio*. New Chat, rename to `Insurance Risk Summary - Prototype`, system instructions plus the SafeHarbor Warehousing prompt, Save with region `us-central1`, then **Deploy → Cloud Run → Deploy as app**, tick the public deployment acknowledgement, Create App. Then **Deploy → Cloud Run → Open app** and send the Coastal Goods Delivery message through the web interface.

The saved prompt can take a few minutes on first save, which the lab notes.

**Task 2**, *Prompt engineering in Agent Studio*. Three prompts in sequence: `Insurance Claim Data Extraction` zero shot at temperature 0.1, the same with a few shot Example added, then `Insurance Story` for the parameter experiments. Temperature 1.5 then 0.1, output token limit 500 then back to the default **65535**, Top-P 0.8 then 1.0.

**Task 3**, *Compare, evaluate, and manage prompts*. `Insurance Risk Factor Identification` at temperature 0.2, then the three dot menu → **Compare**, which duplicates the prompt into two panes. Three comparisons: modified system instructions, temperature 2.0 against 0.2, then Pro against Flash on the longer guidelines prompt.

**Task 4**, *Analyze images with Gemini in Agent Studio*. `Timetable Image Analysis`, and the image comes from **+ → Import from Cloud Storage → the pre built bucket → `timetable.png`**. Title, description and text extraction, then the percentage of flights departing before 11:30 AM, then the same question at temperature 0.8 to watch the explanation style change.

**Task 5**, *Explore Agent Platform Media Studio*. Left nav → **Generate media → Image**, Nano Banana 2, 1:1, the honeybee prompt, then open a thumbnail.

**The Chirp voice section is optional and unscored.** The checkpoint is satisfied by generating the image, so the Text-to-Speech API enable it may ask for is not needed unless you want to hear it.

## Rate limiting

The lab warns about `429 Quota Exhausted` in task 1 and the advice holds throughout: wait a minute and resubmit. There are roughly a dozen model calls across the five tasks, so pausing between submits costs less than fighting a 429.

## Related files

- `gsp515-explore-generative-ai-gemini-api-challenge.md` and `gsp525-enhance-gemini-model-capabilities-challenge.md` for the same models through the SDK rather than the console.
- `gsp524-analyze-reason-multimodal-data-challenge.md` for multimodal analysis in a notebook, and for the thinking configuration this lab exposes as a Thinking level dropdown.
- `gsp517-genai-apps-gemini-streamlit-challenge.md` and `gsp328-serverless-applications-cloud-run-challenge.md` for deploying a generative app to Cloud Run by hand.
- `pending-labs.md` for GSP1210, the other multimodal Gemini lab.
