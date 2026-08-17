# GSP685, bq for Google BigQuery

Six tasks, **six** checkpoints, ten minutes stated and **no cost**. About **three minutes**. Entirely the lab's own terminal.

Fourth of the five required labs in the **Essential Google Cloud CLI Tools** course.

Manual last updated and last tested May 2025.

## Task 6 deletes what checkpoints 3, 4 and 5 score

The only thing in this lab that can cost you points, and it costs three at once.

`bq rm -r babynames` removes the dataset, its table and its data. Checkpoints 3, 4 and 5 all want that dataset present; checkpoint 6 wants it gone. So the run is:

1. blocks 1 to 3
2. click checkpoints 1 through 5
3. block 4
4. click checkpoint 6

Checkpoints latch once earned, so that works. Paste everything at once and three of six are unrecoverable. Now a `gotchas.md` section in its own right, since this is the fifth lab to do it.

## Tasks 1 to 3

```
bq show bigquery-public-data:samples.shakespeare
bq help query | head -30
bq query --use_legacy_sql=false 'SELECT word, SUM(word_count) AS count FROM `bigquery-public-data`.samples.shakespeare WHERE word LIKE "%raisin%" GROUP BY word'
bq query --use_legacy_sql=false 'SELECT word FROM `bigquery-public-data`.samples.shakespeare WHERE word = "huzzah"'
```

Checkpoints 1 and 2. **Both read your job history**, not a result, so the queries have to execute. The `huzzah` one returning nothing is the expected outcome and still scores.

Note the quoting the lab teaches here: the SQL is in **single** quotes so the `"%raisin%"` inside survives untouched. That is the same rule that matters far more in GSP101, where a `!` inside double quotes reaches bash's history expansion.

## Task 4

```
bq ls
bq ls bigquery-public-data: | head -20
bq mk babynames
wget -q https://www.ssa.gov/OACT/babynames/names.zip || wget -q http://www.ssa.gov/OACT/babynames/names.zip
ls -l names.zip
unzip -o -q names.zip
bq load babynames.names2010 yob2010.txt name:string,gender:string,count:integer
bq ls babynames
bq show babynames.names2010
```

Checkpoints 3 and 4.

Two changes from the lab's own commands:

- **`https` first with `http` as a fallback.** The lab gives `http://www.ssa.gov/...`, and plain http to that host is increasingly redirected or refused.
- **`unzip -o`.** Without it a second paste stops at an overwrite prompt, and a prompt inside a pasted block eats the next line.

`bq load` takes a positional schema, `name:string,gender:string,count:integer`, with no flag and no file. About 34,073 rows.

**`bq ls bigquery-public-data:` needs the trailing colon.** Without it `bq` reads the argument as a dataset in the current project and returns nothing useful.

## Task 5

```
bq query "SELECT name,count FROM babynames.names2010 WHERE gender = 'F' ORDER BY count DESC LIMIT 5"
bq query "SELECT name,count FROM babynames.names2010 WHERE gender = 'M' ORDER BY count ASC LIMIT 5"
```

Checkpoint 5. These deliberately omit `--use_legacy_sql=false`, as the lab has them; legacy SQL resolves `babynames.names2010` unquoted without complaint.

The minimum count is 5 because the source omits names with fewer than five occurrences, which is why the "most unusual boys names" query returns a tie rather than a 1.

## Task 6

```
bq rm -r -f babynames
bq ls
```

Checkpoint 6, and **only after 3, 4 and 5 are green**.

**`-f` added.** The lab says "confirm the delete command by typing Y", and that prompt inside a pasted block consumes the next line. `-f` answers it, and this is one of the cases where `--force` style flags are correct rather than risky, because yes is the only answer the task accepts.

The closing `bq ls` should print nothing, which is what checkpoint 6 reads.

## One unused value

The lab panel supplies a region, `us-east4`, and no command in the lab uses it. `bq mk babynames` with no `--location` creates in the US multi region and nothing checks it. Worth noting only so its absence does not look like an omission.

## Related files

- `gsp693-gcloud-cli-beginners-guide.md`, `gsp694-gcloud-for-network-configuration.md` and `gsp695-manage-storage-configuration-gsutil.md`, the earlier labs in this course.
- `gsp1144-knowledge-catalog-command-line.md` for the same teardown ordering trap in a lab where it is easier to miss.
- `gsp327-bigquery-ml-fare-prediction-challenge.md` and `gsp341-create-ml-models-bigquery-ml-challenge.md` for `bq query` at challenge scale, including the digit leading table name that needs a quoted heredoc.
