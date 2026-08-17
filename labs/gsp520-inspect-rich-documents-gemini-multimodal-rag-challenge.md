# GSP520, Inspect Rich Documents with Gemini Multimodality and Multimodal RAG: Challenge Lab

Two tasks, **four** checkpoints, thirty five minutes stated and **5 credits**. **Entirely JupyterLab on Agent Platform Workbench, zero Cloud Shell.**

Manual last updated and last tested **July 2026**, the freshest dates of any lab in these notes. Nothing in it is stale, which matters for the approach below.

## What the started page fills in

| Thing | Value |
|---|---|
| Workbench instance | `generative-ai-jupyterlab` |
| notebook | `inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb` |
| kernel | **`Python 3 (Local)`** |

**The kernel is not the same as its sibling notebook labs.** GSP515, GSP517 and GSP525 all take plain `Python 3`. This one wants `Python 3 (Local)`, and picking the wrong entry in the Select Kernel dialog is a failure that surfaces later as imports not resolving.

Project ID and Location are pre configured in the Getting Started section, so the `global` location trap from GSP517 does not apply here.

## Two setup steps, not one

The Getting Started section, **and then the four cells in Setup and requirements** before task 1. The second is easy to miss because it reads like part of the preamble. Skipping it makes task 1 fail with nothing obviously wrong.

## The checkpoint mapping is uneven

| Checkpoint | Notebook sections it covers |
|---|---|
| 1 | Image understanding across multiple images |
| 2 | Similarities/Differences between images |
| 3 | **four sections**: Generate a video description, Extract tags of objects throughout the video, Ask more questions about a video, Retrieve extra information beyond the video |
| 4 | all of task 2 |

**Save the notebook before every Check my progress.** The lab states it once and means it for all four: *"Save the notebook script before clicking on the Check my progress button for every task."* An unsaved run scores nothing.

**Checkpoint 1's message is asynchronous, not latched.** It reads *"Please run the recommended cells for the section 'Image Understanding across multiple images'. If already done, please wait."* A red result there is a reason to wait and re click rather than to go looking for a fault, which is the opposite of the GSP306 style latched failure.

The 429 the lab warns about is real and the place to expect it is checkpoint 3's four sections plus task 2's metadata build, which calls Gemini once per image across a fourteen page 10K split into Part 1 and Part 2. Wait a minute and rerun the single cell rather than restarting the section.

## How this one was actually completed

Not by filling the TODOs. The circulating helper replaces the notebook wholesale, from the Jupyter terminal:

```
rm inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb
curl -LO https://raw.githubusercontent.com/.../inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb
```

Then in the replacement notebook, the first two cells run one at a time, the third selected, and Run All Below from there. 100/100.

This works here for one specific reason worth naming: **the lab's manual and the notebook file name are current as of July 2026**, so a pre filled copy of that same notebook still lines up with what the four checkpoints read. The same trick against a lab whose notebook has been revised imports cleanly and scores nothing, which is the caution already in `gotchas.md` about bundles built for an earlier variant.

## The one dangerous thing in that helper

**The `rm` runs before the `curl`.** The lab's notebook is the only artifact the entire lab is scored on, there is no copy of it anywhere in the instance, and nothing in JupyterLab brings it back. If the download fails for any reason, a renamed repo path, a rate limit, no egress, the lab is over and needs a fresh instance.

Reverse the order and verify before destroying anything:

```
curl -LO https://raw.githubusercontent.com/.../inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb -o replacement.ipynb
python -c "import json,sys; json.load(open('replacement.ipynb')); print('valid notebook')"
mv inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb original.ipynb.bak
mv replacement.ipynb inspect_rich_documents_w_gemini_multimodality_and_multimodal_rag.ipynb
```

The `json.load` matters because a failed `curl -L` against GitHub returns an HTML 404 page with a 200 status, saved under the notebook's name. JupyterLab then refuses to open it and the original is already gone.

Keeping the original as `original.ipynb.bak` also gives the fallback of filling the TODOs by hand, which is the whole point of not deleting it.

## What the notebook contains, for the manual route

Task 2's helper functions are **provided**, so its fills are about calling them correctly rather than writing retrieval logic:

`get_similar_text_from_query`, `print_text_to_text_citation`, `get_similar_image_from_query`, `print_text_to_image_citation`, `get_gemini_response`, `display_images`.

The metadata the build step produces, which those functions read:

- text side: `text`, `text_embedding_page`, `chunk_text`, `chunk_number`, `text_embedding_chunk`
- image side: `img_desc`, `mm_embedding_from_text_desc_and_img`, `mm_embedding_from_img_only`, `text_embedding_from_image_description`

Task 2's five sections in order: Build metadata of documents containing text and images, Create a user query, Get all relevant text chunks, Create context text, Pass context to Gemini.

Two data notes. The Terms of Service document is **text only**, so the image side helpers do nothing against it; the 10K is the one with tables, charts and graphs. And the video is handed to you as an **https** url, `https://storage.googleapis.com/spls/gsp520/google-pixel-8-pro.mp4`, used in three of checkpoint 3's four sections. Gemini on Vertex takes a `gs://` file uri for media, so a cell feeding this to `Part.from_uri` needs `gs://spls/gsp520/google-pixel-8-pro.mp4` instead. That went untested on this run because the replacement notebook already handled it; read the surrounding cell before pasting the url as given.

## Related files

- `gsp525-enhance-gemini-model-capabilities-challenge.md` and `gsp515-explore-generative-ai-gemini-api-challenge.md` for the same SDK filled by hand.
- `gsp517-genai-apps-gemini-streamlit-challenge.md` for the location variable and for a slow download that looked hung.
- `gsp523-multimodal-vector-search-bigquery-challenge.md` for multimodal embeddings and vector search done in BigQuery instead.
- `genai129-deploy-agent-with-adk-challenge.md` for editing a provided file without destroying it, which is the same lesson as the `rm` above.
