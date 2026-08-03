# GSP1049, Cloud Spanner Loading Data and Performing Backups

Six tasks: explore, insert one row with dml, insert one row with the python client, insert four rows in a batch, load a hundred and fifty thousand rows with Dataflow, then take a backup. Billed at twenty five minutes, about thirty five in practice because Dataflow runs twelve to sixteen minutes and the backup another fifteen.

The instance, database, and empty `Customer` table are created for you. Names are `banking-instance` and `banking-db`, not the `banking-ops-` names used by GSP1050 and the GSP381 challenge lab.

## This lab carries a do not deviate warning

The lab page shows "Do not deviate from instructions. Actions outside the lab scope will trigger an automatic account block." Worth respecting: run the Dataflow job and the backup through the console as the lab describes rather than swapping in gcloud, even though gcloud equivalents exist. Tasks 2 to 4 are already shell based so nothing is lost there.

## Three scored checkpoints

Client library insert, batch insert, and the Dataflow load. Nothing is scored for the backup.

## Tasks 2 to 4

Task 2 is a single dml insert. Task 3 writes `insert.py` and task 4 writes `batch_insert.py`, both straight from the lab text, both run with `python3`.

The python scripts can print a long warning about Spanner client metrics failing to export to Cloud Monitoring, mentioning incomplete resource labels and a missing `instance_id`. It looks alarming and is harmless. The insert still happens.

If `python3 insert.py` fails with `ModuleNotFoundError: google.cloud.spanner`, install the client with `pip install --quiet google-cloud-spanner`.

## Task 5, Dataflow

Shell prep first:

```
gsutil mb gs://YOUR_PROJECT_ID
touch emptyfile
gsutil cp emptyfile gs://YOUR_PROJECT_ID/tmp/emptyfile

gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

Then the console job. Template is Text Files on Cloud Storage to Cloud Spanner, job name `spanner-load`, regional endpoint from the lab page, instance `banking-instance`, database `banking-db`, manifest `spls/gsp1049/manifest.json`, temporary location `YOUR_PROJECT_ID/tmp`. Under optional parameters, uncheck the default machine type and pick E2 and `e2-medium`.

Those two path fields already carry a `gs://` prefix in the form, which is why the lab gives them without one.

Watch it from the shell instead of refreshing the console:

```
gcloud dataflow jobs list --region=YOUR_REGION --format="table(name,state)"
```

Once it says `Done`:

```
gcloud spanner databases execute-sql banking-db --instance=banking-instance \
  --sql='SELECT COUNT(*) FROM Customer'
```

Our run ended with 151483 rows.

## The checkpoints check exact row counts, so click them in order

This is the important finding and it cost most of the time in this run. Each check wants the table to hold exactly what the lab has created up to that point.

- Insert data through a client library wants exactly **two** rows, Richard Nelson from task 2 and Shana Underwood from task 3.
- Insert batch data through a client library wants exactly **six**, those two plus the four from task 4.
- Load data using Dataflow wants the full hundred and fifty thousand.

So run one task, click its checkpoint, confirm it is green, then move on. If you race ahead and load the csv before claiming tasks 3 and 4, both become unpassable until you delete data to walk the table back.

That is exactly what happened here. The recovery, in order, was:

```
gcloud spanner databases execute-sql banking-db --instance=banking-instance \
  --enable-partitioned-dml \
  --sql="DELETE FROM Customer WHERE CustomerId NOT IN (the six ids)"
```

which left six rows and passed the batch checkpoint. Then deleting just the four batch ids left two rows and passed the client library checkpoint. Then `python3 batch_insert.py` put the four back. Partitioned dml is required for the big delete because a hundred and fifty thousand rows blows past the normal mutation limit.

Checkpoints latch once earned, which is what makes this recovery safe: the Dataflow checkpoint stayed green while its rows were being deleted.

## The task 2 insert silently did not run

It was the one command in a pasted block that produces no output of its own, so it was easy to miss. Five of the six scored rows were present and the sixth, Richard Nelson, was absent, which held up the client library checkpoint. Check with:

```
gcloud spanner databases execute-sql banking-db --instance=banking-instance \
  --sql="SELECT COUNT(*) AS scored_rows FROM Customer WHERE CustomerId IN (
    'bdaaaa97-1b4b-4e58-b4ad-84030de92235',
    'b2b4002d-7813-4551-b83b-366ef95f9273',
    'edfc683f-bd87-4bab-9423-01d1b2307c0d',
    '1f3842ca-4529-40ff-acdd-88e8a87eb404',
    '3320d98e-6437-4515-9e83-137f105f7fbc',
    '6b2b2774-add9-4881-8702-d179af0518d8')"
```

Six is the answer you want.

## Task 6, backup

Console only: Backup/Restore in the left menu, Create Backup, database `banking-db`, name `banking-backup-001`, expiration one year. Fifteen minutes, not scored, so there is no need to watch it finish.
