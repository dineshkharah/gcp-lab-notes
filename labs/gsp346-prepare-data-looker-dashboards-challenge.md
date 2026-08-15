# GSP346, Prepare Data for Looker Dashboards and Reports: Challenge Lab

Three tasks, **three** checkpoints, forty minutes stated and **no cost**. About **thirty minutes**.

**Entirely the Looker UI.** No Cloud Shell, no Google Cloud console, not even a project id. You sign in to a Looker instance with credentials from the lab panel and never leave it.

The first Looker lab in these notes. Manual last updated December 2025, **lab last tested February 2024**, so expect small ui drift from the screenshots.

## The values are randomised and they are not all the same number

This run: **9** rows for Looks 1 and 2, **6** rows for Look 4, dashboard named **Plane and Helicopter Rental Hub Board**.

The unstarted page renders every one of those as the same placeholder word, `Count`, which makes them look like one value repeated. They are not. Read each row limit off its own task.

**Titles are graded as strings.** With the values above:

- `Top 9 Cities With Most Heliports`
- `Facility Type Breakdown for Top 9 States`
- `States and Cities with Highest Percentage of Cancellations: Flights over 10,000`
- `Top 6 Airports With Smallest Average Distance`

Checkpoint 1 only scores once **all four** Looks exist, so there is no partial credit while you build.

## The LookML shortcut

Rather than clicking fields in the Explore, a refinement in the LookML IDE can pre-build the query and give you a `start_from_here` link into the Explore with the fields already selected. Three of the four Looks come out of this:

```
explore: +airports {
  query: start_from_here {
    dimensions: [city, state]
    measures: [count]
    filters: [airports.facility_type: "HELIPORT^ ^ ^ ^ ^ ^ ^ "]
  }
}

explore: +airports {
  query: start_from_here {
    dimensions: [facility_type, state]
    measures: [count]
  }
}

explore: +flights {
  query: start_from_here {
    dimensions: [aircraft_origin.city, aircraft_origin.state]
    measures: [cancelled_count, count]
  }
}
```

They get you the right fields; the row limit, sort order, pivot, table calculation, hides and the Look title are still done in the Explore afterwards.

**The filter literal is the thing to keep.** `facility_type` is a **fixed width** field, so the stored value is `HELIPORT` followed by trailing spaces. `^` escapes the next character in a Looker filter, so `HELIPORT^ ^ ^ ^ ^ ^ ^ ` is `HELIPORT` plus seven literal spaces. A plain `HELIPORT` filter can match nothing and return an empty table with no error. Now a section in `gotchas.md`, because padded values break exact match filters in BigQuery and Sheets the same way.

## Look 1, and filter without selecting

Explore > **Airports**. Dimensions **City** and **State**, measure **Airports Count**, row limit **9**, sort Airports Count descending, visualization **Table**.

*"The Facility Type should not be included in the visualization"* is satisfied by clicking the **filter icon** beside Facility Type rather than the field name. Select the field and filter it and you get a fourth column, which fails.

## Look 2, the same field selected and pivoted

Explore > **Airports**. Dimensions **State** and **Facility Type**, and **Pivot** the Facility Type. Measure **Airports Count**, row limit **9**, sort descending, Table.

The exact opposite of Look 1 with the same field, which is the point of the pair. Pivoting turns Facility Type into column headers, which is what "the corresponding Facility Types" plural means.

## Look 3, hide rather than remove

Explore > **Flights**. Dimensions **City** and **State** under the **Aircraft Origin** view, not on Flights itself. Measures **Flights Count** and **Cancelled Count**, both of which you need even though neither appears in the result.

Table calculation, named exactly `Percentage of Flights Cancelled`, formatted **Percent (3)**. Both are graded.

```
${flights.cancelled_count}/${flights.count}
```

Filter **Flights Count** `is greater than` `10000`.

Then on the **Cancelled Count** and **Flights Count** columns, gear > **Hide from Visualization**. **Do not deselect them**; the calculation depends on them and removing them breaks it with an unknown field error. Hide versus remove is the whole difficulty of this Look.

Sort the calculation descending.

## Look 4, a custom measure and the only ascending sort

Explore > **Flights**. Search for and select the **Origin and Destination** dimension, which is a single field, which is why the requirement says two columns rather than three.

On the **Distance** dimension, gear > **Aggregate > Average**. Rename the result to exactly `Average Distance (Miles)`, capital M and parentheses included.

Filter that new measure `is greater than` `0`. Not cosmetic: without it the top rows are same airport pairs at zero distance and the result looks nothing like the screenshot.

Sort **ascending**. This is the only ascending sort in the lab and it is easy to skim past. Row limit **6**.

## Task 2, the merge, and the rules Looker guesses

Build the primary query first, then merge into it.

1. Explore > **Flights**: **Aircraft Origin City**, **Aircraft Origin State**, **Aircraft Origin Code**, **Flights Count**. Sort Flights Count descending, row limit **10**. **Run it.**
2. Explore gear, top right > **Merge Results**. Your query becomes the primary.
3. Add a query from the **Airports** explore: **State**, **City**, **Code**, with three filters all set to **Yes**: **Control Tower**, **Is Major**, **Joint Use**.
4. **Check the merge rules Looker guessed.** The lab flags this and it is the likeliest failure: it must pair Aircraft Origin City with Airports City, Aircraft Origin State with Airports State, and Aircraft Origin Code with Airports Code.
5. Row limit **10**, visualization **Bar chart**.
6. Save **to a new Dashboard**. Tile title `Busiest, Major Joint-Use Airports with Control Towers`, dashboard **Plane and Helicopter Rental Hub Board**.

**The dashboard is created here, in task 2**, not in task 3. Task 3 only adds to it.

## Task 3

For each of the four Looks: open it, three dot menu > **Save > To an existing dashboard** > the dashboard > Save.

Then confirm **five tiles**: four Looks plus the merged bar chart. Layout does not matter.

## Related files

- `connected-sheets-bigquery.md` for the same kind of analysis in Sheets against BigQuery, including the `">0"` versus `"1"` version of the padded value problem.
- `arc133-bigquery-workspace-apps-script-challenge.md` for charts and formulas over a BigQuery source.
- `gsp1042-analytics-as-a-service-data-sharing.md`, which still calls Looker Studio by its old Data Studio name, for the other Looker branded product.
