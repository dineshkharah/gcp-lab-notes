# GSP1063, Finding Data in Google Sheets

Four tasks, **four** checkpoints, fifteen minutes stated. A **Google Workspace lab**, third for the same fictional bakery after GSP1061 and GSP1062.

Workbook kept as `labs/assets/GSP1063.xlsx`, so a re-run can upload it directly.

## The oldest manual in these notes

**Manual last updated February 2024, lab last tested August 2022.** Everything else in this repo is within a year or two of current; this one is close to four years stale, and it shows in two places.

- The instructions say to open the `On the Rise Bakery Bulk Orders` file "that has been created for you", while the checkpoint is named **"Upload a spreadsheet and manipulate data"**. In our run the workbook was uploaded to replace the pre-created one. Keeping the xlsx here means that is a single step next time.
- Task 3's VLOOKUP paragraph ends mid sentence: *"The fourth, and optional parameter,"* and then stops. Nothing follows. It was going to say the fourth argument controls exact versus approximate matching, which is what the `False` in the formula does.

Per the usual rule in `gotchas.md`, where the notes and the lab text disagree the lab text wins, but with a gap this size the console is the better authority than either.

## Task 1, SPLIT then TRANSPOSE

```
B1: =SPLIT(A1, ",")
```

**Double-click** the blue fill handle rather than dragging. It fills to the end of the adjacent data automatically, which matters at a hundred rows.

Then hide column A, and double-click the divider between the D and E labels to autofit the width.

The lab notes `Data > Split text to columns` as the alternative. Worth knowing the difference: that one rewrites the cells in place and needs no helper column, so it is destructive and there is nothing left to re-run. `SPLIT` keeps the source and stays live.

On the **New Order** sheet:

```
A8: =TRANSPOSE(A1:A7)
```

Copy `A8:G8`, go to the Bulk Orders sheet, click **B101**, paste, then use the clipboard dropdown to pick **Paste values only**.

Two things there are load bearing:

- **B101, not A101.** Column A holds the raw comma separated strings and is hidden. The split output starts at B, so that is where a new record has to land.
- **Paste values only.** The transposed cells are a formula pointing at the New Order sheet. A plain paste carries the formula across, where its relative references land somewhere meaningless and the row shows the wrong data or an error.

## Task 2, and why the replacement has to hit the hidden column

`Ctrl+F`, **More options**, Find `Muffin`, Replace with `Blueberry Muffin`, Search **This sheet**, **Replace all**.

**Run it exactly once.** Replacing `Muffin` with `Blueberry Muffin` a second time turns the results into `Blueberry Blueberry Muffin`, because the replacement string contains the search string. If the count in the confirmation looks wrong, undo rather than running it again.

Worth understanding what it actually edited. Everything visible in columns B to H is `SPLIT` output, and formula results cannot be edited directly. So the replacement lands on the source text in **column A**, and hiding a column does not take it out of find and replace scope. The exception is row 101, which was pasted as values and so is edited in place.

Then SUBSTITUTE:

```
I1: Adjusted Delivery Date
I2: =SUBSTITUTE(F2,"Nov-6","Nov-7")
```

`SUBSTITUTE` is a **text** function. It works here only because those delivery dates are stored as text; against real date values it would operate on the underlying serial number and match nothing. And when there is no match it returns the input unchanged, which is why column I is a complete list rather than gappy.

`Nov-16` is safe from this, since it does not contain the literal string `Nov-6`.

## Task 3, two functions with opposite column conventions

This is the one thing from this lab worth carrying elsewhere, because having both in one task invites mixing them up.

**VLOOKUP counts relative to its range.**

```
J2: Georgia Nkosi
K2: =VLOOKUP(J2, G2:I100, 3, False)
```

`Adjusted Delivery Date` is in sheet column I, but the index is `3` because the range starts at G and I is the third column of it. Two further constraints that come with that: the search key must be in the **first column of the range**, so G has to hold the names, and `False` demands an exact match, which is what you want for a name.

**QUERY uses the sheet's own column letters.**

```
Discount!A1: =QUERY('Bulk Orders'!$B$2:$I$100, "select H where E > 500")
```

`H` and `E` here are the real sheet columns H and E, not the seventh and fourth columns of the range. Same task, same source data, opposite convention.

The single quotes around `'Bulk Orders'` are required because the sheet name contains a space. The optional variation is changing `500` to `750`.

## Task 4, and the two error messages

```
J3: Alexander Jorgenson
K3: =VLOOKUP(J3, B2:I100)
```

That is two arguments where three are needed, and the message is:

> Wrong number of arguments to VLOOKUP. Expected between 3 and 4 arguments, but got 2 arguments.

Then:

```
K3: =VLOOKUP(J3, B2:I100, 8)
```

Now the call is well formed and the lookup genuinely fails, so the message changes to:

> Did not find value 'Alexander Jorgenson' in VLOOKUP evaluation.

**That second one is the quiz answer**, since the question asks for the *new* message after the fix. Both display as `#N/A` in the cell, which is the point of the exercise: the cell text tells you nothing and the hover message tells you everything.

```
K4: =ISERROR(K3)
K3: =IFERROR(VLOOKUP(J3, B3:I100, 8), "Record not found")
```

Note the lab's IFERROR version starts the range at **B3**, not B2 as in the two attempts above. Paste it as written.

The thing to actually look at is **K4 flipping from `TRUE` to `FALSE`**. Once `IFERROR` handles the failure, K3 is no longer an error value at all, it is the string `Record not found`, so `ISERROR` stops reporting one. That distinction between an error and a handled error is the lesson.

## Related files

- `gsp1061-use-charts-in-google-sheets.md` and `gsp1062-validate-data-in-google-sheets.md`, the other two labs for this bakery, both with their workbooks under `labs/assets/`.
- `connected-sheets-bigquery.md` for querying real BigQuery data from Sheets, where `QUERY` is replaced by actual SQL.
