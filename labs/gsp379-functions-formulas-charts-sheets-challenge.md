# GSP379, Use Functions, Formulas, and Charts in Google Sheets: Challenge Lab

Five tasks, **five** checkpoints, fifteen minutes stated. A **Google Workspace lab**: no project, no Cloud Shell, no console.

The badge closer for GSP1061, GSP1062 and GSP1063, and it draws on all three plus two things none of them teach: descriptive statistics and pivot tables.

Files kept: `labs/assets/GSP379.xlsx` is the challenge workbook and `labs/assets/GSP379.pptx` is the Slides deliverable from task 3.

Manual last updated February 2024, **lab last tested June 2023**. Stale, like GSP1063, and the wording shows it.

## The two quiz answers

Both unscored, both worth having.

- Function that checks a valid email address: **`ISEMAIL`**. The other three options do not exist.
- Functions used to calculate range: **`MAX`** and **`MIN`**. There is no `RANGE` function in Sheets, which is the point of asking. Range is `MAX(...) - MIN(...)`.

## Task 1, and a self contradicting instruction worth resolving before you type

Column B wants `=PROPER(...)` on the first names, same as GSP1062's task 3.

The email half contradicts itself across two sentences. It says **"verify that column E contains a valid email address"**, so column E holds the addresses, and then says to **"use that function in column E"**. Both cannot be true. Writing `ISEMAIL` formulas into column E overwrites the very addresses the second half of the task then asks you to build a validation rule against.

Decide before typing, because the destructive reading is not recoverable without re-uploading. The reading that keeps the data is the one GSP1062 already demonstrates: `ISEMAIL` referencing the address column from the **next free column**, and the data validation rule applied to **column E** itself.

The task explicitly wants **both** methods, so one without the other leaves the checkpoint short:

1. the `ISEMAIL` formula column
2. a data validation rule. `Data > Data validation > + Add rule`, and note that Sheets has a **Text is valid email** criterion, which is the honest rule here. GSP1062's `Text contains True` construction was validating a boolean's text output and was an exercise rather than a recommendation.

## Task 2, and doing the three steps in the right order

Three separate things:

- **Sort by column G**, the birthdays.
- **Update column I**, October 5 to October 10. Either find and replace or `SUBSTITUTE`, as in GSP1063. If you use find and replace, run it once.
- **Cell A18**, a function retrieving a manager's email address.

**Sort before writing A18.** `Sort sheet A to Z` sorts every row on the sheet, so a value already sitting in A18 gets carried off into the data. Freeze row 1 first for the same reason, per GSP1062.

For A18, `VLOOKUP` only searches rightward from the first column of its range, so the range has to **start at the role column** and the index counts from there:

```
=VLOOKUP("Manager", D2:E17, 2, FALSE)
```

Read the actual column letters off your own copy. `QUERY` is the alternative and takes sheet column letters rather than relative positions, which is the distinction recorded in `gsp1063-finding-data-in-google-sheets.md`.

## Task 3, chart of roles then a presentation named exactly Staff Roles

Selecting the single roles column and `Insert > Chart` is enough. Sheets aggregates a lone text column into counts by category on its own, so a column or pie chart of staff per role appears without building a helper table first.

Then a **new Slides presentation** with the chart embedded, `Insert > Chart > From Sheets`, and the filename **`Staff Roles`** exactly. The filename is the part a scorer can check most easily, so it is the part worth getting character perfect.

Mechanics of the embed are in `gsp1061-use-charts-in-google-sheets.md`, including that the chart stays linked to the sheet and is visible to anyone who can open the presentation regardless of their access to the workbook.

## Task 4, descriptive statistics

Over `C2:C101`, into the table at `E3:F8`:

```
=AVERAGE(C2:C101)
=MEDIAN(C2:C101)
=MAX(C2:C101)-MIN(C2:C101)
=STDEV(C2:C101)
```

Two things to check rather than assume.

**The table spans six rows and the task names four statistics.** Read the labels in column E and fill what is actually there rather than filling four cells and stopping; the extra rows are most likely `MAX` and `MIN` in their own right, which is also why the quiz answer is those two.

**`STDEV` is the sample standard deviation, `STDEVP` is the population one.** For a hundred sampled orders `STDEV` is the expected answer, and it is the one to use unless the label says population.

## Task 5, the pivot table default is the trap

`Insert > Pivot table`, and in the dialog choose **New Sheet** for Insert to, as the lab's note says.

- **Rows**: `Item`
- **Values**: `Customer Rating`, **Summarize by AVERAGE**

That last setting is the whole task. Sheets defaults the value field to **SUM**, so a pivot table built by accepting the defaults shows total rating per item, which is a meaningless number and not what the task asked for. Change the Summarize by dropdown before clicking away.

## Related files

- `gsp1061-use-charts-in-google-sheets.md` for charts and the Slides embed.
- `gsp1062-validate-data-in-google-sheets.md` for `PROPER`, sorting, freezing and data validation rules.
- `gsp1063-finding-data-in-google-sheets.md` for `SUBSTITUTE`, find and replace, `VLOOKUP` and `QUERY`.
