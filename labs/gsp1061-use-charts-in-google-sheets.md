# GSP1061, Use Charts in Google Sheets

Four tasks, **four** checkpoints, fifteen minutes stated. A **Google Workspace lab**: no project, no Cloud Shell, no console. The credentials are a Workspace student account and everything happens in Drive, Sheets and Slides.

Files kept: `labs/assets/GSP1061.xlsx` is the source workbook the lab has you download, and `labs/assets/GSP1061.pptx` is the Slides deliverable from task 4. Uploading the xlsx straight from here skips the download step on a re-run.

Manual last updated and last tested July 2026.

## Tick the conversion setting before uploading, not after

The one step with a consequence. In Drive, **Settings > Convert uploads to Google Docs editor format** has to be checked **before** the `.xlsx` goes up.

It applies at upload time only. Ticking it afterwards does nothing to a file already in Drive, and the fix is deleting the upload and doing it again. An unconverted workbook stays an `.xlsx` opened in Sheets' Office editing mode, where the chart editor and `SPARKLINE` are not the same thing the lab is describing.

## Task 1, pie then column

Locations sheet. Click the **gray column label** for column E to select the whole column, then `Insert > Chart`.

Sheets picks a chart type from the data's shape, so what appears may not be a pie. Set **Pie chart** in the Chart type dropdown, then immediately switch to **Column chart**, which is what the rest of the task customises. The pie exists only to be looked at.

Customize tab:

- **Chart & axis titles**, title to `Locations by Continent`
- **Series**, next to **Format data point** click **Add**, pick `Continent: South America`, colour purple
- **Gridlines and ticks**, select **Vertical axis** in the first dropdown, then tick **Minor gridlines**

That last one is the only fiddly bit. Gridlines and ticks applies to one axis at a time and defaults to the horizontal one, so selecting Vertical axis first is what makes the Minor gridlines checkbox do what the lab wants.

## Task 2, and the chart that lands on top of the other one

`Sales - South America` sheet, select `A1:M3`, `Insert > Chart`. A line chart appears. Change Chart type to **Combo chart**.

Then select `A1:M3` again and insert a second chart. **It is placed directly on top of the first one.** The lab mentions this in passing and it reads like nothing happened. Drag it clear before touching the Chart editor, otherwise you cannot tell which chart the editor is bound to and it is easy to convert the combo chart into the scatter chart and end up with one chart instead of two. The checkpoint wants both.

Then Chart type to **Scatter chart**.

## Task 3, and why the sparkline range is C:N

Right-click the gray label for column B, **Insert 1 column left**. That is what makes the ranges in the formulas look wrong at first glance: the twelve months of data were in `B:M` and are now in `C:N`, and the new empty column B is where the inline charts go.

```
=SPARKLINE(C2:N2)
```

Progress bar against the annual goal, first quarter over target:

```
=SPARKLINE(SUM(C3:E3)/2400000,{"charttype", "bar"; "max", 1; "min", 0; "color1", "blue"})
```

`C3:E3` is January to March for the same reason. Then change `"blue"` to `"green"`, which the lab asks for as a separate step and is what the checkpoint is likely to look at.

The options array syntax is worth remembering because it is unusual:

- a **comma** separates an option name from its value
- a **semicolon** separates one option from the next

So `{"charttype", "bar"; "max", 1}` is two options, not four values. Getting those two separators the wrong way round is the only way this formula fails.

`data` is the sole required argument, and it takes a literal array too: `=SPARKLINE({500,100,200,400,300})`.

## Task 4, publish a chart rather than the document

`File > Share > Publish to web`. The dropdown defaults to **Entire document** and the lab wants **a single chart** picked from that list instead. Publish, then OK on the confirmation.

The published page is **restricted to the lab domain**, so the URL only resolves while signed into the student account and only for the life of the lab. Nothing leaks and nothing is worth saving.

Then Slides, opened in the **student account**, which is what the incognito instruction is for. Blank presentation named `On the Rise Bakery`, Layout **Title Only**, title text `On the Rise Bakery`, then `Insert > Chart > From Sheets`, pick the workbook, and import the **column chart** from task 1.

A chart imported this way stays **linked** to the sheet. The lab's note is worth keeping: anyone who can open the presentation sees the linked chart even with no access to the underlying spreadsheet.

## The quiz answer

Unscored, in between tasks 2 and 3. January sales from all locations, showing how each continent contributes to the total: **pie chart**. It is contribution of parts to a whole, which is the one thing a pie does better than a column chart, and there are few enough continents to stay under the five slice guideline the lab states in task 1.

## Related files

- `connected-sheets-bigquery.md` for Sheets driven from BigQuery data rather than a local workbook.
- `gsp235-apps-script-sheets-maps-gmail.md` for scripting Sheets instead of charting it.
