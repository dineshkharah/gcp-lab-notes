# GSP1050, Spanner Defining Schemas and Understanding Query Plans

Six tasks: load three tables by hand, load a fourth with supplied python, query it, add a column and write to it, add two secondary indexes, then read query plans. Fifteen minutes billed, about twenty in practice. Four scored checkpoints, twenty five points each.

The instance, database, and all four tables are created for you. Names are `banking-ops-instance` and `banking-ops-db`, matching the GSP381 challenge lab rather than the `banking-instance` used by GSP1048 and GSP1049.

## Follow this lab as written

Two reasons, both from the lab itself. It carries the "Do not deviate from instructions" warning, and in task 4 it prints the gcloud ddl equivalent with an explicit "Do not issue this command" note. So the column and the indexes go through `snippets.py`, not through `gcloud spanner databases ddl update`, even though that would produce the same schema.

## Task 1, three inserts in Spanner Studio

Spanner Studio, new sql editor tab, run the three insert statements from the lab, clicking Clear between each. Three rows, four rows, nine rows.

Watch one value: the Insurance portfolio short name is `Ins` here. The GSP381 challenge lab uses `Insurance` for the same row. Easy to carry the wrong one over if you run them back to back.

## Task 2, the helper code

```
mkdir python-helper
cd python-helper

wget https://storage.googleapis.com/cloud-training/OCBL373/requirements.txt
wget https://storage.googleapis.com/cloud-training/OCBL373/snippets.py

pip install -r requirements.txt
pip install setuptools

python snippets.py banking-ops-instance --database-id banking-ops-db insert_data
```

If pip refuses with `externally-managed-environment`, add `--user` to both pip lines.

`snippets.py` holds every function the rest of the lab uses. The pattern is always the same: `python snippets.py INSTANCE --database-id DATABASE FUNCTION_NAME`.

## The rest, in order, clicking each checkpoint as it comes

```
python snippets.py banking-ops-instance --database-id banking-ops-db query_data

python snippets.py banking-ops-instance --database-id banking-ops-db add_column
python snippets.py banking-ops-instance --database-id banking-ops-db update_data
python snippets.py banking-ops-instance --database-id banking-ops-db query_data_with_new_column

python snippets.py banking-ops-instance --database-id banking-ops-db add_index
python snippets.py banking-ops-instance --database-id banking-ops-db read_data_with_index
python snippets.py banking-ops-instance --database-id banking-ops-db add_storing_index
python snippets.py banking-ops-instance --database-id banking-ops-db read_data_with_storing_index
```

The four checkpoints are the three table load, the Campaigns load, the added column, and the first secondary index. `update_data`, the reads, and the storing index are not scored but are worth running, since the storing index read is the point of that section: a plain index read cannot return `MarketingBudget`, and the `STORING` variant can.

## Task 6, query plans

Three queries in Spanner Studio, clicking the Explanation tab after each: an inner join, an aggregate with a group by, and a co-located join across the interleaved `Category` and `Product` tables. Nothing scored, it is a reading exercise about distributed unions and cross applies.

## Ran clean

No surprises in this one. It is the least error prone of the four Spanner labs because almost everything is a supplied function invoked by name.
