# GSP1062, Validate Data in Google Sheets

Five tasks, **six** checkpoints, twenty minutes stated. A **Google Workspace lab** like GSP1061: no project, no Cloud Shell, no console. Task 2 carries two checkpoints, one for the filter and one for the filter view.

Same fictional bakery as `gsp1061-use-charts-in-google-sheets.md`, and the setup differs in one way worth knowing.

Manual last updated and last tested July 2026.

## Setup: Make a copy is not optional

GSP1061 has you tick a Drive setting so the upload converts. This lab takes a different route: **Open file picker > Upload tab > Browse**, then **File > Make a copy** into My Drive.

That copy step is the one that matters. The uploaded workbook opened straight from the picker is not the file the rest of the lab works on, and the copy in My Drive is. Skipping it leaves you editing something the scorer is not looking at, which is a slow thing to notice because everything else behaves normally.

## Task 1, freeze before sorting

```
View > Freeze > 1 row
```

Then right-click the column C heading and **Sort sheet A to Z**.

The freeze is not cosmetic. `Sort sheet` sorts every row on the sheet including row 1, so without the freeze the header gets sorted into the data as just another value. Freeze first, every time.

Then a new sheet named `Items Sorted By Unit Price`, and in A1:

```
=SORT(Items!A1:Items!C15, Items!B1:Items!B15, FALSE)
```

`Items!A1:Items!C15` looks like a typo and is not. Repeating the sheet name on both sides of the colon is valid and means the same as `Items!A1:C15`.

Three arguments: the range, the column to sort by, and `TRUE` ascending or `FALSE` descending. Sorting by more than one column means appending another column and direction pair on the end.

The lab's warning is worth heeding: the output is a formula range, so **editing any cell inside it gives `#REF!`**. Use `SORT` for output you will not touch.

## Task 2, filter and filter view are different things and score separately

- A **filter** changes what everyone sees, and edits the spreadsheet's state.
- A **filter view** is private to you, sorts and filters without disturbing collaborators, and leaves the underlying data alone.

Both are scored, so the sequence is: create the filter, **click checkpoint 2**, `Data > Remove filter`, create the filter view, save it, **click checkpoint 3**, and only then delete the views. Deleting either one before its checkpoint means recreating it.

**Select cell A1 before `Data > + Create filter view`.** The lab says this at the end of the section rather than the start. The view takes its range from the current selection, so creating it with nothing sensible selected gives a view over the wrong range and the filter icon will not be where the instructions expect.

Saving is `Data > View options > Save view > Save`, which is easy to skip since the view already appears to work.

## Task 3, and the column shift halfway through it

The order the lab gives means the sheet stops matching its own earlier column letters. Worth knowing so you do not try to fix it.

Written first, against the original layout:

```
E1: Truncated Unit Prices
E2: =TRUNC(B2, 4)
D2: =B2*C2      then changed to      =ROUND(B2*C2, 2)
```

Fill both down with the blue handle in the lower right of the cell, then select column D by its gray label and `Format > Number > Currency`.

**Then** the insert: right-click the column B label, `Insert 1 column left`, `B1: Formatted Name`, `B2: =PROPER(A2)`, filled down.

That insert pushes everything right of A across by one, and Sheets rewrites the formulas to match, so `=TRUNC(B2, 4)` becomes `=TRUNC(C2, 4)` on its own and the currency formatting travels with its column. The formulas now disagree with what the lab told you to type and are correct. Leave them.

`PROPER` capitalises the first letter of each word. `TRUNC(x, 4)` cuts to four decimal places without rounding, which is the point of using it rather than `ROUND` for the price labels: the displayed price must not round up.

## Task 4, and what the validation rule actually flags

The **Customers** sheet, which is untouched by task 3's column insert, so its letters are the ones the lab states.

`Insert > Function > Info > ISEMAIL`, then `=ISEMAIL(C2)` in D2, filled down. It returns `TRUE` or `FALSE`.

Then select `D2:D100`, `Data > Data validation > + Add rule`, Criteria **Text contains**, value `True`, Done.

The rule flags what does **not** satisfy it, so the marked cells are the rows where `ISEMAIL` came back `FALSE`, meaning the malformed addresses. Two of them in this dataset. Read the row numbers off your own copy for the unscored quiz between the tasks; the invalid cells carry a red corner marker once the rule is applied, so scanning column D finds them in seconds.

Note that this validation is a slightly odd construction: the honest rule would be Text is valid email on column C itself, which the lab points out is available. Validating a boolean formula's text output in a neighbouring column is the exercise, not the recommendation.

## Task 5, and the conditional formatting reference

```
Apply to range:  C1:C100
Custom formula:  =COUNTIF(C:C,C1)>1
Fill: red
```

The `C1` in the formula matters and pairs with the range starting at `C1`. A custom formula is evaluated relative to the **top left cell of the applied range**, so a range of `C2:C100` would need `C2` in the formula. Getting that pair out of step is the usual reason conditional formatting highlights the wrong rows or none at all.

Then the greeting, in column G:

```
=CONCATENATE("Hello ", A2, ",")
```

which comes out visibly misaligned in G2 because that name carries stray whitespace, then:

```
=CONCATENATE("Hello ", TRIM(A2),",")
```

`TRIM` strips leading, trailing and repeated internal spaces. The lab's own text points at G2 looking different from the rest of the column, which is the tell.

## Remove duplicates does less than the red highlighting suggests

Select column C, `Data > Data cleanup > Remove duplicates`, **Expand to A:D**, tick `Data has header row`, and under Column to analyze pick **Select all**.

With all four columns selected, a row is only a duplicate if **every** one of A to D matches. The conditional formatting from earlier in the task highlights repeated **email addresses** regardless of the rest of the row. So two customers with different names sharing one address stay red and stay in the sheet after the cleanup runs, and that is correct behaviour rather than a failed dedupe.

The dialog reports how many rows it removed, which is the number to sanity check against.

## Related files

- `gsp1061-use-charts-in-google-sheets.md`, the charting lab for the same fictional company, including its source workbook under `labs/assets/`.
- `gsp1063-finding-data-in-google-sheets.md`, the third of the set, covering SPLIT, TRANSPOSE, SUBSTITUTE, VLOOKUP, QUERY and IFERROR.
- `gsp379-functions-formulas-charts-sheets-challenge.md`, the challenge lab, which reuses this lab's PROPER and data validation rule and prefers the Text is valid email criterion over the construction used here.
- `connected-sheets-bigquery.md` for Sheets over BigQuery data.
- `gsp235-apps-script-sheets-maps-gmail.md` for scripting Sheets rather than using built in functions.
