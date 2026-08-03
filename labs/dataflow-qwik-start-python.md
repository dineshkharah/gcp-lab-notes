# Dataflow qwik start, Python

Small lab. Create a bucket, install the Beam sdk inside a container, run the wordcount example remotely, check it succeeded. Fifteen to twenty minutes, mostly the pip install and the job wait.

Task 1 is described through the console but `gcloud storage buckets create` is equivalent and the scorer only checks the bucket.

## Setup and task 1

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export BUCKET=$PROJECT_ID-bucket

gcloud config set compute/region $REGION

gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com

gcloud storage buckets create gs://$BUCKET --location=US

gcloud storage buckets describe gs://$BUCKET --format="value(name,location,location_type)"
```

The lab asks for multi region `us`. A regional bucket fails the check, so confirm the describe prints `US` and `multi-region`.

## Task 2, inside a container

```
docker run -it -e DEVSHELL_PROJECT_ID=$DEVSHELL_PROJECT_ID python:3.12 /bin/bash
```

The lab prose says Python 3.9 while the command pulls 3.12. Use the command, the prose is stale.

Inside the container:

```
pip install 'apache-beam[gcp]'==2.67.0
python -m apache_beam.examples.wordcount --output OUTPUT_FILE
ls
cat OUTPUT_FILE-00000-of-00001
```

## Task 3, remote run, still inside the container

Only `DEVSHELL_PROJECT_ID` crosses into the container, because that is the one passed with `-e`. Region and bucket have to be set again inside, which is where people trip.

```
BUCKET=gs://YOUR_BUCKET_NAME

python -m apache_beam.examples.wordcount --project $DEVSHELL_PROJECT_ID \
  --runner DataflowRunner \
  --staging_location $BUCKET/staging \
  --temp_location $BUCKET/temp \
  --output $BUCKET/results/output \
  --region YOUR_REGION \
  --worker_machine_type=e2-standard-2
```

It streams job messages until the job finishes. `JOB_STATE_RUNNING` with "Starting 1 workers" is normal and takes three to five minutes.

The output file can appear in `results/` while the job still shows Running, because the write stage finishes before teardown. Wait for Succeeded before clicking the checkpoint.

## Quiz

Dataflow `temp_location` must be a valid Cloud Storage url: **True**.
