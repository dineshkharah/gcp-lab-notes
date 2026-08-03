# Streaming Analytics into BigQuery challenge lab

Five tasks: a bucket, a BigQuery dataset and table, a Pub/Sub topic, a Dataflow job streaming topic to table, and a test message. Fifteen to twenty minutes.

## Three ordering and region traps, all hit in one run

**Create the staging bucket before submitting the Dataflow job.** Submitting first makes the job fail during staging, and it sits at Pending for about seven minutes before showing Failed. That was the whole delay in this run.

**A newer region name can be valid for Dataflow but not for Cloud Storage.** `us-west8` was the lab region. `gcloud storage buckets create --location=us-west8` returned `HTTPError 403: Permission denied on 'locations/us-west8' (or it may not exist)`, while the same region worked fine for the Dataflow job. Create the bucket with `--location=US`.

**Publish the test message only after the job reports Running.** The PubSub to BigQuery template creates its own subscription when the pipeline starts, so anything published while the job is Pending is dropped and never reaches the table.

## The dataset location can differ from the lab region

This lab wanted the dataset in `US` multi region while everything else was regional. `bq mk` without `--location` silently uses the default, so pass it.

## Tasks 1 to 3

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export DATASET=YOUR_DATASET
export TABLE=YOUR_TABLE
export TOPIC=YOUR_TOPIC
export JOB=YOUR_JOB_NAME

gcloud services enable dataflow.googleapis.com

gcloud storage buckets create gs://$PROJECT_ID --location=US

bq mk --location=US -d $DATASET
bq mk --table $PROJECT_ID:$DATASET.$TABLE data:STRING

gcloud pubsub topics create $TOPIC
gcloud pubsub subscriptions create $TOPIC-sub --topic=$TOPIC
```

The subscription is created by hand because `gcloud pubsub topics create` does not add the default subscription that the console checkbox does.

## Task 4, the job

One job, with the exact name the lab gives, using the classic template. No flex template, no second job with a suffix, no `outputDeadletterTable`.

```
gcloud dataflow jobs run $JOB \
  --gcs-location gs://dataflow-templates-$REGION/latest/PubSub_to_BigQuery \
  --region $REGION \
  --staging-location gs://$PROJECT_ID/temp \
  --parameters inputTopic=projects/$PROJECT_ID/topics/$TOPIC,outputTableSpec=$PROJECT_ID:$DATASET.$TABLE
```

Poll for state rather than blocking:

```
gcloud dataflow jobs list --region=$REGION --format="table(name,state,creationTime)"
```

## Task 5

Once the job is Running:

```
gcloud pubsub topics publish $TOPIC --message='{"data": "73.4 F"}'
sleep 60
bq query --use_legacy_sql=false "SELECT * FROM \`$PROJECT_ID.$DATASET.$TABLE\`"
```

The sleep matters. Querying straight away shows an empty table and makes it look like the pipeline is broken.

## Note on the community script

It runs two Dataflow jobs, a flex template one and then a classic one with a suffix appended to the job name, so the name check fails. It also sets `outputDeadletterTable` to the same table as the output, which causes schema conflicts, and creates the dataset without a location.
