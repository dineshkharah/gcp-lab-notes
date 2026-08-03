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

## Two things that cost time in this run

**The task 2 insert silently did not run.** It was the one command in a pasted block that produces no output of its own, so it was easy to miss. Five of the six scored rows were present and the sixth, Richard Nelson, was absent. The task 3 checkpoint appears to look at the rows from tasks 2 and 3 together, so it failed until that row existed. Check with:

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

**The checkpoints lag.** After the data was provably correct, all three still reported failure. Clicking them again a minute later passed all three with no other change. So re-click before diagnosing anything. A theory was forming that the checks wanted a small exact row count and that the Dataflow load had made tasks 3 and 4 unpassable, which would have meant deleting a hundred and fifty thousand rows with partitioned dml. That theory was wrong and the deletion would have been for nothing.

## Task 6, backup

Console only: Backup/Restore in the left menu, Create Backup, database `banking-db`, name `banking-backup-001`, expiration one year. Fifteen minutes, not scored, so there is no need to watch it finish.
