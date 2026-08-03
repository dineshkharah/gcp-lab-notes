# Analyze BigQuery data in Connected Sheets challenge lab

Entirely in the Google Sheets ui. No Cloud Shell at all. Fifteen to twenty minutes.

Five tasks: connect a BigQuery public dataset, a COUNTIF formula, a pie chart, an extract, and a calculated column.

## Connecting

New blank spreadsheet, then Data, Data connectors, Connect to BigQuery, pick the project, then Public datasets, `new_york_taxi_trips`, `tlc_yellow_trips_2022`, Connect.

The Connected Sheet toolbar then gives you Chart, Pivot table, Function, Calculated column, and Extract. Those five buttons are the whole lab.

## Task 2, count trips with an airport fee

```
=COUNTIF(tlc_yellow_trips_2022!airport_fee, ">0")
```

The column holds `0`, `1.25`, or null, so a walkthrough that uses `"1"` as the criteria matches almost nothing. `">0"` is what the task actually describes. If the scorer rejects it, `"1.25"` is the next thing to try.

## Task 5, calculated column

Use the Calculated column button, not a normal cell.

```
=IF(fare_amount>0, tolls_amount/fare_amount*100, 0)
```

Two things to watch. The lab text says `toll_amount` but the real BigQuery column is `tolls_amount`, plural. And a widely copied crib for this lab uses `tip_amount`, which answers a different question, because that crib is for a variant that asks about tips.
