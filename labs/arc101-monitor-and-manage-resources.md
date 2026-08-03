# arc101, Monitor and Manage Google Cloud Resources challenge lab

Four tasks: a bucket shared with a second user, a Pub/Sub topic, a gen2 thumbnail function triggered by the bucket, and an alerting policy. About twenty minutes when it goes smoothly. This run did not, and the cause is the service agent trap in `gotchas.md`.

## Setup and tasks 1 and 2

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export BUCKET=YOUR_BUCKET
export TOPIC=YOUR_TOPIC
export FUNCTION=YOUR_FUNCTION
export USER2=SECOND_USER_EMAIL
export PN=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

gcloud services enable artifactregistry.googleapis.com cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com eventarc.googleapis.com run.googleapis.com \
  logging.googleapis.com pubsub.googleapis.com

gcloud beta services identity create --service=pubsub.googleapis.com --project=$PROJECT_ID
gcloud beta services identity create --service=eventarc.googleapis.com --project=$PROJECT_ID
```

Then the grants, one at a time, and verify all four landed before deploying:

- gcs agent `service-$PN@gs-project-accounts.iam.gserviceaccount.com` gets `roles/pubsub.publisher`
- pubsub agent `service-$PN@gcp-sa-pubsub.iam.gserviceaccount.com` gets `roles/iam.serviceAccountTokenCreator`
- compute sa `$PN-compute@developer.gserviceaccount.com` gets `roles/eventarc.eventReceiver` and `roles/artifactregistry.reader`
- the second user gets `roles/storage.objectViewer`

```
gcloud storage buckets create gs://$BUCKET --location=$REGION
gcloud pubsub topics create $TOPIC
```

## Task 3, the function

Write the topic name straight into `index.js` rather than using `sed -i "16c\..."` on a line number, which silently rewrites the wrong statement if the file shifts by a line. Check the file before deploying:

```
wc -l index.js
node --check index.js && echo "SYNTAX OK"
grep topicName index.js
```

Then deploy:

```
gcloud functions deploy $FUNCTION \
  --gen2 --runtime=nodejs22 --region=$REGION --source=. \
  --entry-point=thumbnail --trigger-bucket=$BUCKET --quiet
```

After a deploy that follows failed attempts, traffic can be left on an old revision, so:

```
gcloud run services update-traffic $FUNCTION --region=$REGION --to-latest
```

Then upload a fresh image. An object uploaded before the trigger existed is never processed.

## Task 4, alerting policy

The display name has to be exactly what the lab asks. One community script creates `Active Cloud Function Instances` while the lab wants `Active Cloud Run Function Instances`, and the scorer matches on the name.

```
export CHANNEL=$(gcloud beta monitoring channels create \
  --display-name="Email alerts" --type=email \
  --channel-labels=email_address=$ALERT_EMAIL --format="value(name)")
```

Then a policy file with `resource.type = "cloud_function"`, metric `cloudfunctions.googleapis.com/function/active_instances`, `COMPARISON_GT`, threshold zero, sixty second alignment on `ALIGN_MAX`, and that channel. Create it with `gcloud beta monitoring policies create --policy-from-file=`.

## What went wrong

Three deploy attempts failed on Eventarc permissions. The real cause was that the gcs agent had no `pubsub.publisher` binding at all: the grant had been issued but never applied, and `get-iam-policy` filtered on `gs-project-accounts` returned nothing. Granting it again with the email written out, plus the token creator grant on the pubsub agent, fixed it. On the next lab the same failure showed its actual error message, which was that the pubsub agent did not exist yet.
