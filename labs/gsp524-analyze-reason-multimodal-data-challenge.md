# GSP524, Analyze and Reason on Multimodal Data with Gemini: Challenge Lab

Five tasks, **eight** checkpoints, twenty five minutes stated and **5 credits**. **Entirely JupyterLab on Agent Platform Workbench, zero Cloud Shell.** Window was 2h1m at 100/100, so the twenty five minutes is a reading estimate.

Manual last updated May 2026, lab last tested October 2025.

**Took two attempts, and the first was abandoned at 30/100 for a reason that no longer exists.** Read the next section before anything else.

## The blocker was fixed upstream between attempts

The first run stalled because the notebook shipped:

```python
MODEL_ID = "gemini-2.0-flash-001"
```

which 404s in the lab's own region:

```
ClientError: 404 NOT_FOUND. Publisher model
`projects/PROJECT/locations/europe-west1/publishers/google/models/gemini-2.0-flash-001`
was not found or your project does not have access to it.
```

**Checkpoint 1 passed anyway**, because it only verifies the imports and the client, so the lab reads as working until the first real call in task 2. The lab was set aside rather than worked around.

On the second attempt the notebook ships **`gemini-2.5-flash`** and nothing 404s. Nobody fixed it locally; the artifact changed.

**The signal that something had changed was not the manual date**, which is still May 18 2026 on both runs. It was the lab text's model wording:

| First attempt | Between | Second attempt |
|---|---|---|
| started page: "Gemini 3.5 Flash" | unstarted page: "Gemini **Flash**" | started page: "Gemini 3.5 Flash" |

So the template briefly rendered a generic string, and the notebook's `MODEL_ID` moved to `2.5`. Now a caution in `gotchas.md`: the manual date does not track the files a lab ships.

**Still worth one cheap check before task 2**, since this could regress:

```python
print("MODEL_ID=[", MODEL_ID, "] LOCATION=[", LOCATION, "]")
```

If it is not a model the region serves, fix it **at its definition in task 1's cell**, not in a patch cell below, because eight sections read it and rerunning the setup would restore the broken value.

## Eight checkpoints, three code shapes

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

Every one of the ten fills is one of three shapes: a prompt string, a `generate_content` call, and for the second half of each task the same call plus `config=config`.

**`config` is pre defined in the notebook**, holding the thinking configuration, as is the `print_thoughts()` helper. Neither needs writing, which is why the thinking cells are no harder than the initial ones.

## What each modality is passed as

This is the part that was unknown after the first attempt, and it is simpler than expected because the setup section pulls the media down to local disk.

**Text**, a variable interpolated into an f string:

```python
contents=prompt
```

**Images**, a pre built list appended to the prompt:

```python
contents=[prompt] + image_parts
```

**Audio**, a single pre built part:

```python
contents=[thinking_mode_prompt, audio_part]
```

`image_parts` and `audio_part` are constructed by the notebook's own setup cells, so **there is no `Part.from_uri` anywhere and none of the `gs://` versus `https` trap from GSP520**.

**One naming oddity in the starter**: task 4's *initial* cell, the non thinking one, still calls its prompt variable `thinking_mode_prompt`. Keep the name the cell uses rather than tidying it, since the checkpoint reads the cell.

## Task 5 is a pipeline, not a section

Tasks 2, 3 and 4 each end by writing a file:

```python
with open('analysis/text_analysis.md', 'w') as f:
    f.write(thinking_model_response.text)
```

and likewise `analysis/image_analysis.md` and `analysis/audio_analysis.md`. **Task 5 reads all three back**, concatenates them under headings, and asks for a report over the combination.

So the sections are not independent. Skipping one, or running task 5 before an earlier section has written its file, fails with a `FileNotFoundError` or an empty report rather than anything that looks like a bad prompt.

**And checkpoint 8 is triggered by an upload, not by the notebook.** The last cell is pre written:

```python
!gcloud storage cp analysis/final_report.md gs://{PROJECT_ID}-bucket/analysis/final_report.md
```

Generating the report is not enough; the object has to land in the bucket. Second instance in these notes of a checkpoint whose scoring surface is Cloud Storage rather than the running artifact, after GSP541.

## The prompts that scored

Nothing subtle, but the checkpoints do read what was asked for, so each prompt names the specific extractions the task describes.

Task 2 initial: sentiment per item, key themes across quality, fit, style, customer service and pricing, and frequently mentioned products or features.

Task 2 deep dive: drivers of positive and negative sentiment, brand perception impact, three areas for improvement, three takeaways.

Task 3 initial: identify the apparel items, describe colour, design, fit and style, and name style trends across the set.

Task 3 deep dive: hypothesise the target audience per image, analyse how visual elements carry the brand message, compare against industry trends, recommend campaign and product actions.

Task 4 initial: transcribe with speaker identification, sentiment, key themes, overall perception.

Task 4 deep dive: satisfaction signals, factors shaping perception, risks to reputation, three data driven recommendations.

Task 5: summarise sentiment, identify themes and trends, cover style preferences and usage patterns, evaluate the audio against the product image, and recommend marketing and positioning changes, formatted as a Markdown report.

## Two operational notes

**Edit the existing TODO cells rather than adding new ones**, and run top to bottom. The checkpoints read cell contents, and a correct answer in a cell the scorer does not look at scores nothing.

**A kernel restart wipes every earlier section's variables while leaving the files intact.** So after a restart, rerun task 1 and then only the section you are on; the `analysis/*.md` files task 5 needs survive.

## Related files

- `gsp520-inspect-rich-documents-gemini-multimodal-rag-challenge.md` and `gsp525-enhance-gemini-model-capabilities-challenge.md`, the sibling notebook labs.
- `gsp515-explore-generative-ai-gemini-api-challenge.md` for the same SDK with `Part.from_uri` and function declarations.
- `gsp517-genai-apps-gemini-streamlit-challenge.md` for the location variable defaulting to `global`.
- `gsp523-multimodal-vector-search-bigquery-challenge.md` for multimodal work done in BigQuery instead.
