# GSP327, Engineer Data for Predictive Modeling with BigQuery ML: Challenge Lab

Three tasks, **three** checkpoints, fifteen minutes stated and **5 credits**. Entirely Cloud Shell with `bq query`, four pastes. **Total compute time was about two minutes** across all three tasks, so the fifteen minutes is generous rather than tight.

Manual last updated March 2024, **lab last tested September 2023**. Nearly three years, second stalest in these notes after GSP1063. Nothing had drifted in this run.

Sibling of `gsp374-bigquery-soccer-bqml.md`, the other BQML challenge lab.

## The values are randomised per run

Six of them, and the unstarted lab page renders them as generic words like `Table` and `Number`, so they have to come off the started page:

| Placeholder | Our run |
|---|---|
| destination table | `taxi_training_data_242` |
| target column | `fare_amount_440` |
| `trip_distance >` | `4` |
| `fare_amount >=` | `3` |
| `passenger_count >` | `4` |
| model | `fare_model_408` |

The prediction table name is fixed at `2015_fare_amount_predictions`. Note the lab reused **4** for both the distance and the passenger thresholds; do not assume they differ, and do not assume they match either.

## The two schemas, which the lab makes you find

`historical_taxi_rides_raw` has 19 columns and **1,031,673,361 rows**. `report_prediction_data` has exactly six:

```
pickup_datetime  TIMESTAMP
pickuplon        FLOAT
pickuplat        FLOAT
dropofflon       FLOAT
dropofflat       FLOAT
passengers       INTEGER
```

That six column list is the actual specification for task 1. The cleaned table must expose those names plus the target, because `TRANSFORM()` is stored with the model and **re-applied at prediction time**, so a name that does not match here breaks task 3 rather than task 1. Hence the aliases:

```
pickup_longitude  AS pickuplon
pickup_latitude   AS pickuplat
dropoff_longitude AS dropofflon
dropoff_latitude  AS dropofflat
passenger_count   AS passengers
```

`trip_distance` is deliberately absent from the prediction table, which is why the model has to derive distance from the coordinates.

## Task 1, and why the sampling fraction needs a cap

Count the filtered rows before choosing a fraction. It takes under a second and removes the guess:

```
bq query --use_legacy_sql=false '
SELECT COUNT(*) AS rows_passing_filters
FROM `taxirides.historical_taxi_rides_raw`
WHERE trip_distance > 4 AND fare_amount >= 3
  AND pickup_longitude > -78 AND pickup_longitude < -70
  AND dropoff_longitude > -78 AND dropoff_longitude < -70
  AND pickup_latitude > 37 AND pickup_latitude < 45
  AND dropoff_latitude > 37 AND dropoff_latitude < 45
  AND passenger_count > 4'
```

**17,392,156**, a 1.69% pass rate. So the sampling fraction multiplies that, not the billion:

```
bq query --use_legacy_sql=false "
CREATE OR REPLACE TABLE taxirides.$TABLE_NAME AS
SELECT
  (tolls_amount + fare_amount) AS $FARE_COL,
  pickup_datetime,
  pickup_longitude AS pickuplon,
  pickup_latitude AS pickuplat,
  dropoff_longitude AS dropofflon,
  dropoff_latitude AS dropofflat,
  passenger_count AS passengers
FROM taxirides.historical_taxi_rides_raw
WHERE
  RAND() < 0.05
  AND trip_distance > 4
  AND fare_amount >= 3
  AND pickup_longitude > -78 AND pickup_longitude < -70
  AND dropoff_longitude > -78 AND dropoff_longitude < -70
  AND pickup_latitude > 37 AND pickup_latitude < 45
  AND dropoff_latitude > 37 AND dropoff_latitude < 45
  AND passenger_count > 4
LIMIT 900000
"
```

`0.05` gave **869,447** rows against the 900,000 cap. **Three percent of headroom.** At `0.06` it would have been about 1.04 million and failed the under one million requirement outright. The `LIMIT` is the backstop that makes the fraction safe to be wrong about, and it costs nothing when the fraction happens to be right.

The circulating script uses `RAND() < 0.001` with no cap, which would have produced about 17,000 rows here. Passing, but two orders of magnitude less training data than needed for a comfortable RMSE.

Target is `tolls_amount + fare_amount`, **not** `total_amount`, because total includes tips and tips are not knowable at prediction time. The lab says so and it is the one instruction in the task that changes the answer.

## Task 2, and reading the RMSE correctly

```
bq query --use_legacy_sql=false "
CREATE OR REPLACE MODEL taxirides.$MODEL_NAME
TRANSFORM(
  * EXCEPT(pickup_datetime),
  ST_Distance(ST_GeogPoint(pickuplon, pickuplat), ST_GeogPoint(dropofflon, dropofflat)) AS euclidean,
  CAST(EXTRACT(DAYOFWEEK FROM pickup_datetime) AS STRING) AS dayofweek,
  CAST(EXTRACT(HOUR FROM pickup_datetime) AS STRING) AS hourofday
)
OPTIONS(input_label_cols=['$FARE_COL'], model_type='linear_reg') AS
SELECT * FROM taxirides.$TABLE_NAME
"
```

Ninety seconds to train on 869k rows.

Three things about that `TRANSFORM()`:

- `* EXCEPT(pickup_datetime)` passes the label and the five real features through while dropping the raw timestamp, which is useless as a numeric feature but needed to derive the two below.
- `euclidean` does most of the work, and it exists because `trip_distance` is not available at prediction time.
- `dayofweek` and `hourofday` are **cast to STRING on purpose**, so BQML treats them as categorical. Left numeric, the model would learn that hour 23 is "greater than" hour 1.

**`ML.EVALUATE` does not return RMSE.** It returns `mean_squared_error`, so the gate needs an explicit square root:

```
bq query --use_legacy_sql=false "
SELECT SQRT(mean_squared_error) AS rmse, mean_absolute_error, r2_score
FROM ML.EVALUATE(MODEL taxirides.$MODEL_NAME)"
```

Ours: **rmse 8.226**, mae 4.835, r2 0.596. Under the 10 gate with room to spare.

This is one of the few checkpoints in these notes that is a **quality threshold rather than an existence check**, so it can fail on a correctly created object. Read the number before clicking.

## Task 3, and the digit that forces a heredoc

`2015_fare_amount_predictions` starts with a digit, so it is **not a valid unquoted BigQuery identifier** and needs backticks. But backticks inside a double quoted shell string are command substitution. A quoted heredoc solves both at once, since nothing inside it expands:

```
cat > predict.sql <<'EOF'
CREATE OR REPLACE TABLE `taxirides.2015_fare_amount_predictions` AS
SELECT *
FROM ML.PREDICT(
  MODEL `taxirides.fare_model_408`,
  (SELECT * FROM `taxirides.report_prediction_data`)
)
EOF

cat predict.sql
bq query --use_legacy_sql=false < predict.sql
```

The model name is hardcoded rather than a variable, because the same quoting that protects the backticks also blocks expansion. `cat` the file first to confirm it survived the paste.

No `TRANSFORM()` clause here and none is wanted. It is stored with the model, so `ML.PREDICT` re-derives euclidean distance, day of week and hour of day from the raw prediction table automatically. Output column is `predicted_fare_amount_440`, and we got 74,362 rows in about a second.

## The model predicts negative fares, and still scores 100

The interesting part of this lab, and it is not a mistake in the SQL.

```
predicted_fare_amount_440
-7.502629752620123
-2.0393360989401117
 0.007611570879817009
-0.11465346394106746
-0.7528064483776689
```

The lab's own cleaning rules restrict training to `passenger_count > 4` **and** `trip_distance > 4`, so the model only ever saw five or six passenger rides of more than four miles. `report_prediction_data` is overwhelmingly **single passenger short hops**. The model is extrapolating a linear fit far outside the regime it was fitted on, and a linear regression will happily run through zero into negative prices.

Which is why RMSE 8.23 and negative predictions are both true at once: `ML.EVALUATE` measures a held-out slice of the **training** distribution, not the serving one. Textbook train and serve skew, except here it is baked into the instructions rather than being an error.

Checkpoint 3 passes anyway, because it checks the table exists with predictions in it and nothing about whether they are sensible. Worth knowing so the negative numbers do not send you back to rebuild task 1: nothing upstream is wrong.

If you wanted a model that actually worked, you would drop the two restrictive filters and keep only the fare minimum and the coordinate bounds. That would fail task 1's checkpoint.

## Related files

- `gsp374-bigquery-soccer-bqml.md`, the other BQML challenge lab, also run entirely with `bq`, where the per run constants were the whole difficulty too.
- `gsp341-create-ml-models-bigquery-ml-challenge.md`, the third, where nothing is randomised and the difficulty is a checkpoint that refuses correct work if you run ahead of it.
- `gsp1041-bigquery-authorized-views-data-sharing.md` and the rest of that family for the BigQuery sharing side.
