# GSP525, Enhance Gemini Model Capabilities: Challenge Lab

Four tasks, **four** checkpoints, twenty five minutes stated and **5 credits**. **Entirely JupyterLab on Agent Platform Workbench, zero Cloud Shell.**

The twenty five minutes is the reading estimate, not the window. The page still read 2h 8m remaining at 100/100.

Manual last updated May 2026, lab last tested October 2025.

## What the started page fills in

The unstarted text leaves three things as unrendered templates, and all three matter:

| In the unstarted text | Actual value |
|---|---|
| Workbench instance name | `workbench-notebook` |
| notebook file | `enhance-gemini-model-capabilities.ipynb` |
| `model_name`, used in all three topic lines | **Gemini 3.5 Flash** |

Kernel is `Python 3`.

## Six TODO cells, and one config object behind all of them

The lab reads as three separate capabilities but it is one pattern repeated. Every feature, code execution, Google Search grounding, and the response schema, is configured in a `GenerateContentConfig` handed to `generate_content`. Get task 2 right and tasks 3 and 4 are variations.

| Task | Cell heading | Fill |
|---|---|---|
| 2 | 1. Define the code execution tool | `Tool(code_execution=ToolCodeExecution())` |
| 2 | 2. Define the prompt with the code to be executed | an f string naming the data and asking for code, see below |
| 3 | 1. Define the Google Search tool | `Tool(google_search=GoogleSearch())` |
| 3 | 2. Define the prompt with grounding | the prompt, with **Nike Air Jordan XXXVI** as the product |
| 3 | 3. Generate a response with grounding | `GenerateContentConfig(tools=[google_search_tool])` |
| 4 | 5. Construct the search query | `f"{model} price at {retailer}"` |
| 4 | 6. Use Response Schema to extract the data | the `generate_content` call, `response_mime_type` plus `response_schema` |

The task 2 prompt has to do two things, reference the data variable and explicitly ask for the code to be run:

```
f"""what is the average price of sneakers in {sneaker_prices}
Generate and run code for the calculation."""
```

**Asking for the calculation is not enough.** The tool is what permits execution but the prompt is what triggers it, so *"Generate and run code"* is the part that makes the response carry an executed result rather than a code block to read.

## The cell numbering in task 4 is the tell

Task 3 has cells 1, 2, 3. Task 4 has cells **5 and 6**. Cell 4 is never mentioned in the lab instructions, which means it is a provided cell, and given what task 4 needs it is the schema definition itself.

So the schema is handed to you. Cell 6's job is to reference it, not to author it, which is the difference between one line and twenty:

```
response_mime_type="application/json",
response_schema=<the name bound in cell 4>,
```

Read cell 4 before writing cell 6. Anything scored here is about naming the object the notebook already built.

## Imports

Everything in the table above lives in `google.genai.types`, not on the client and not in `google.genai` directly:

```
from google.genai.types import Tool, ToolCodeExecution, GoogleSearch, GenerateContentConfig
```

Task 1's own cell handles the SDK install and the `genai.Client`, and it is scored on its own, so run it and click before touching anything else. **Check what the location variable is set to while you are in there.** If it is left at `global` that is the GSP517 404 waiting to happen, and all four checkpoints go through the same api.

## The community helper

Correct on the four cells it covers, and worth having for the exact `Tool(...)` shapes. Five things about it:

- **Its title belongs to a different lab entirely.** It is headed *Establish Hybrid Network Connectivity with NCC* with a matching video badge, and the body under that is GSP525. Now a note in `gotchas.md`, because a title that names the wrong lab is a reason to read a helper rather than a reason to discard it.
- **It covers four of the six TODO cells.** Missing are task 3 cell 2, the grounded prompt, and task 4 cell 6, the schema call. Both are the cells where something has to be written rather than copied.
- **Every snippet is fenced as `bash` and all of it is Python.** Cosmetic, but it means nothing in it was ever run as given.
- **`f"{model} price at {retailer}"` assumes both names already exist in the notebook.** `model` in particular is a name that commonly holds the model id in these notebooks, so confirm what it is bound to in the surrounding cells before trusting the interpolation.
- No imports, so the four type names it hands you have to be traced back to `google.genai.types` yourself.

## Related files

- `gsp515-explore-generative-ai-gemini-api-challenge.md` for `Tool` and `FunctionDeclaration` in the same SDK, and `Part.from_uri`.
- `gsp517-genai-apps-gemini-streamlit-challenge.md` for the location variable defaulting to `global`.
- `genai129-deploy-agent-with-adk-challenge.md` for editing notebook and source cells without breaking them, and for why reading the file first matters.
- `pending-labs.md` for GSP520, the remaining notebook lab in this family.
