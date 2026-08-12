# GSP341, Create ML Models with BigQuery ML: Challenge Lab

Four tasks, **four** checkpoints at 25 each, twenty minutes stated and **5 credits**. All SQL, so either the BigQuery console or `bq query` from Cloud Shell. A clean run is about **twelve minutes of runtime**.

**Nothing is randomised.** Dataset `ecommerce`, models `customer_classification_model`, `improved_customer_classification_model`, `finalized_classification_model`, all fixed. Tasks 1 and 4 hand you the model SQL verbatim; tasks 2 and 3 are the ones you write.

Third BQML lab in these notes after `gsp327-bigquery-ml-fare-prediction-challenge.md` and `gsp374-bigquery-soccer-bqml.md`.

Manual last updated and last tested June 2026.

## The thing that cost an hour: do not run ahead of checkpoint 3

**Read this before starting.** Checkpoint 3 refused for over an hour on a first instance where the work was demonstrably correct, and passed first time on a second instance where the only difference was doing the tasks in order.

What the first instance looked like, from `bq ls --models ecommerce`:

```
customer_classification_model            15:18:10
finalized_classification_model           15:26:02      <- task 4, built first
improved_customer_classification_model   15:29:44      <- task 3, built second
```

Task 4's model existed before checkpoint 3 had ever been earned. Six things were then tried and **none** recovered it:

- `pageviews` included, and excluded
- `labels` and `input_label_cols` for the label option
- hard refresh of the lab page, repeatedly
- **deleting `finalized_classification_model`** so the dataset held exactly the two models the task describes
- rebuilding the entire chain in order inside the same instance, create, evaluate, create, evaluate
- a fresh `ML.EVALUATE` against the current version of the model after each rebuild

Every attempt produced a working model at `roc_auc` around **0.909**, quality `good`, evaluated on the correct held out window, and every click returned `It doesn't look like you've completed this step yet.`

A fresh lab instance, same six queries, run strictly in order with each checkpoint clicked green before the next command, passed **on the first click**.

Causation is one pair of runs rather than proof, so treat it as the working rule rather than the explanation: **build nothing belonging to a later task until the current checkpoint is green.** And once a checkpoint has refused correct work several times, restarting the lab is a cheaper move than the seventh variation. Twelve minutes against the hour spent proving the SQL was already right.

That inverts the rule elsewhere in `gotchas.md`. Earned checkpoints latch and stay green, which is what makes destructive recovery safe. Here a **failed** checkpoint appeared to latch as well, and no correction of state cleared it.

## What the two evaluations should return

Useful as a sanity check, since these numbers are stable run to run:

| Model | roc_auc | quality |
|---|---|---|
| `customer_classification_model`, 2 features | **0.7238** | `decent` |
| `improved_customer_classification_model`, 9 features | **0.9095** | `good` |

If your improved model lands near 0.909, the SQL is right and any checkpoint problem is elsewhere. Training took 89 seconds for the first model and 112 to 138 for each of the other two.

## The dataset has to be in the US

```
bq mk --location=US -d ecommerce
```

`data-to-insights.ecommerce.web_analytics` lives in the US multi region and BigQuery will not join across locations. A dataset created elsewhere fails at `CREATE MODEL` rather than at `mk`, which makes it look like a model problem.

## The circulating script omits a whole model

Its task 4 gives only the `ml.PREDICT` query and **never creates `finalized_classification_model`**, which the lab supplies right there in the task. Running it as written gives `Not found: Model ecommerce.finalized_classification_model`.

It also uses `input_label_cols` for the improved model and `labels` elsewhere. Both work; the deprecated one emits `Found deprecated training option(s): labels` and nothing else. Neither had any effect on scoring.

## Two diagnostic dead ends, recorded so they are not retried

**`INFORMATION_SCHEMA.MODEL_OPTIONS` is not readable by the student account:**

```
Access Denied: Table ecommerce:INFORMATION_SCHEMA.MODEL_OPTIONS:
User does not have permission to query table
```

So the stored training options cannot be inspected from inside the lab.

**The `Labels` column in `bq ls --models` is resource labels, not the ML label column.** It is empty for every model whether training succeeded or not, so it says nothing.

What does work is `ML.FEATURE_INFO`, which lists exactly what a model was trained on:

```
bq query --use_legacy_sql=false 'SELECT input FROM ML.FEATURE_INFO(MODEL `ecommerce.improved_customer_classification_model`)'
```

Nine rows for the improved model: `latest_ecommerce_progress`, `bounces`, `time_on_site`, `pageviews`, `source`, `medium`, `channelGrouping`, `deviceCategory`, `country`. The label is absent from that list, which is how you confirm it was treated as the label rather than a feature.

## Two wrong leads worth not repeating

**A `pageviews` feature mismatch between the create and evaluate queries.** There isn't one. `IFNULL(totals.pageviews, 0) AS pageviews` and a bare `totals.pageviews` both produce a column named `pageviews`, so the feature lists agree either way and the community script was never broken there.

**Deriving the evaluate query from the model query with `sed`.** Swapping the date range is tempting, but stripping the leading `CREATE`/`OPTIONS` lines by line number also removes the `WITH all_visitor_stats AS (` opener if the count is off by one, producing a silently malformed file. Write both queries out in full; the duplication is cheaper than the failure.

## The four required features in task 3

The lab lists them in prose and they map to:

| Lab wording | Column |
|---|---|
| how far through checkout | `MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress` |
| where the visitor came from | `trafficSource.source`, `trafficSource.medium`, `channelGrouping` |
| device category | `device.deviceCategory` |
| geographic | `IFNULL(geoNetwork.country, "") AS country` |

Plus `bounces` and `time_on_site` carried over from the first model, and `pageviews`, which the lab does not ask for but the canonical version includes. `UNNEST(hits) AS h` is what makes `latest_ecommerce_progress` reachable, and it is why the improved model takes two minutes to train where the first takes ninety seconds.

Note also that task 2's checkpoint is titled "Confirm that both machine learning models have been evaluated" while only one model exists at that point. It passes anyway.

## The run that works

Six files, then six commands. Write all six first:

`t1.sql` and `t4.sql` are the lab's own SQL for the first and finalized models. `t2.sql` is `ML.EVALUATE` on the first model over **20170501 to 20170630**, the two months deliberately held out of the nine month training window. `t3.sql` is the improved model over the same nine months. `t3_eval.sql` is `t3.sql`'s inner query with the dates swapped to the eval window, wrapped in `ML.EVALUATE`. `t4_predict.sql` is `ml.PREDICT` over **20170701 to 20170801**, the final month.

Then, one at a time, each checkpoint green before the next:

```
bq query --use_legacy_sql=false < t1.sql          # checkpoint 1
bq query --use_legacy_sql=false < t2.sql          # checkpoint 2
bq query --use_legacy_sql=false < t3.sql
bq query --use_legacy_sql=false < t3_eval.sql     # checkpoint 3
bq query --use_legacy_sql=false < t4.sql
bq query --use_legacy_sql=false < t4_predict.sql  # checkpoint 4
```

Note there is **no checkpoint between `t3.sql` and `t3_eval.sql`**. Task 3 is scored on the model and its evaluation together, so clicking after the create and before the evaluate is exactly the mistake that starts this file.

Quoted heredocs for all six, because the SQL is full of backticks, `#` comments and single quotes.

## Related files

- `gsp327-bigquery-ml-fare-prediction-challenge.md`, the other BQML challenge lab, where `ML.EVALUATE` returning `mean_squared_error` rather than RMSE is the equivalent trap.
- `gsp374-bigquery-soccer-bqml.md`, the third, where randomised constants are the difficulty.
