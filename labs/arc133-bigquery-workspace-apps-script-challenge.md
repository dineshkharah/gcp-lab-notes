# ARC133, Integrate BigQuery Data and Google Workspace using Apps Script: Challenge Lab

Four tasks, **four** checkpoints, twenty minutes stated and about **fifteen**. **No Cloud Shell at all**, and no Google Cloud console either beyond reading the project id.

Everything happens in `script.google.com` and Google Sheets. The only value needed is the project id.

Manual last updated and **last tested August 2024**, the oldest date in this batch, and nothing in it had rotted.

## What it is

Three of the four tasks are guided labs stitched together. Task 1 is the Apps Script BigQuery advanced service sample; tasks 2 and 3 are Connected Sheets against the Chicago taxi dataset, the same shape as `connected-sheets-bigquery.md` with New York swapped for Chicago; task 4 is typing an address into a cell.

The lab hands you the entire Apps Script file, so there is no code to write anywhere in it.

## Task 1, and the guard that does not guard

```
var PROJECT_ID = "PROJECT_ID";
if (!PROJECT_ID) throw Error('Project ID is required in setup');
```

That check never fires. `"PROJECT_ID"` is a **non empty string and therefore truthy**, so leaving the placeholder in passes the guard and fails later inside `BigQuery.Jobs.query` with a permissions error against a project called `PROJECT_ID`. The line reads like protection and provides none.

Two other things the lab states once and does not repeat:

- **Rename the file to `bq-sheets.gs`** and press Enter. It is `Code.gs` by default.
- **Services → + → BigQuery API → Add**, in the left panel. The advanced service is not on by default, and the lab's own note gives the remedy if it misbehaves: remove the service and add it again.

Then Save, Run, accept the authorization prompts. The execution log prints the URL of the spreadsheet the script created.

One thing to know if it ever breaks: the query uses `[bigquery-public-data:samples.shakespeare]`, **legacy SQL** bracket syntax. If a future gcloud or Apps Script default flips, adding `useLegacySql: true` inside the `request` object next to `query` is the fix. It was not needed here.

## Task 2, three formulas, and the one that answers a different question

New blank spreadsheet, **Data > Data connectors > Connect to BigQuery**, then project > **Public datasets > chicago_taxi_trips > taxi_trips**.

```
=COUNTUNIQUE(taxi_trips!company)
=COUNTIF(taxi_trips!tips,">0")
=COUNTIF(taxi_trips!tips,">0")/COUNTA(taxi_trips!tips)
=COUNTIF(taxi_trips!fare,">0")
```

**The task asks for a percentage and the circulating crib gives a count.** *"Find the percentage of taxi rides in Chicago that included a tip"* against `=COUNTIF(taxi_trips!tips,">0")`, which is how many, not what share. Both cells cost nothing and only one of them can be missing, so put both in.

`">0"` rather than `"1"` for the tip test, for the reason in `connected-sheets-bigquery.md`: these columns hold amounts, not flags, so matching a literal value catches almost nothing.

## Task 3, three charts and the filter the cribs drop

Chart button on the Connected Sheets toolbar, three times.

**Pie, payment types.** Drag `payment_type` to **Label**, `fare` to **Value**, then under Value change **Sum to Count**. Counting a currency column looks odd and is what the guided lab does; the aggregation is what matters, not the column.

**Line, mobile revenue over time.** `trip_start_timestamp` to the **X axis**, tick **Group by** and choose **Year-Month**. `fare` into **Series**. Then **Filter > Add > payment_type**, open the *Showing all items* dropdown, **Filter by condition > Text contains**, value `mobile`. If the chart comes back empty, try `Mobile` capitalised, since that condition can compile to a case sensitive match.

**Line, since 2015.** This is the one the cribs miss. They list both line chart questions and then give **one** procedure, which leaves you with two identical charts and a partial score you cannot explain. Chart 3 is chart 2 plus a second filter:

- **Filter > Add > trip_start_timestamp**
- **Filter by condition > Date is after > exact date `1/1/2015`**, or Filter by values with the pre 2015 years unticked.

Duplicate chart 2 and add the filter rather than rebuilding.

Three **distinct** charts is what the checkpoint counts.

## Task 4 promises Apps Script and does not use it

The heading is *"Use Apps Script to create a new Google Sheets worksheet and enter data"*. The instructions are: open Sheets, click A1, type `76 9th Ave, New York`. No script, no editor, nothing to run.

The checkpoint is the address in A1 of a new spreadsheet. Do not go hunting for the missing step; it is a stale heading, the same species as GSP303's task list belonging to GSP301.

## Related files

- `connected-sheets-bigquery.md`, tasks 2 and 3 against the New York taxi dataset, and the source of the `sheetname!column` idiom and the `">0"` versus `"1"` note.
- `gsp235-apps-script-sheets-maps-gmail.md` for Apps Script with Sheets and the authorization flow.
- `arc126-apps-script-and-appsheet-challenge.md` and `gsp1146-appsheet-no-code-chat-apps.md` for the rest of the Apps Script family.
- `gsp1061-use-charts-in-google-sheets.md` for charts in plain Sheets, without a BigQuery source behind them.
