# gsp315, Store, Process, and Manage Data on Google Cloud challenge lab

Same shape as arc101, a bucket, a topic, and a gen2 thumbnail function, but the lab supplies newer function code that uses `sharp` instead of `imagemagick-stream` and registers with `functions.cloudEvent`. Deployed first try here because the service agent lesson from arc101 was applied up front.

## The two blanks in the supplied code

The lab hands you `index.js` with `functions.cloudEvent('')` and `const topicName = ""`. The first takes the function name, which is also the entry point. The second takes the topic name.

## Order that worked

```
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
export PROJECT_ID=$DEVSHELL_PROJECT_ID
export BUCKET=YOUR_BUCKET
export TOPIC=YOUR_TOPIC
export FUNCTION=YOUR_FUNCTION
export PN=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

gcloud services enable artifactregistry.googleapis.com cloudfunctions.googleapis.com \
  cloudbuild.googleapis.com eventarc.googleapis.com run.googleapis.com \
  logging.googleapis.com pubsub.googleapis.com

gcloud storage buckets create gs://$BUCKET --location=$REGION
gcloud pubsub topics create $TOPIC

gcloud beta services identity create --service=pubsub.googleapis.com --project=$PROJECT_ID
gcloud beta services identity create --service=eventarc.googleapis.com --project=$PROJECT_ID
```

Then the four grants from arc101, plus `roles/eventarc.serviceAgent` on the freshly created eventarc agent, which does not come with it automatically. Verify each, wait two minutes, then deploy.

```
gcloud functions deploy $FUNCTION \
  --gen2 --runtime=nodejs22 --region=$REGION --source=. \
  --entry-point=$FUNCTION --trigger-bucket=$BUCKET --trigger-location=$REGION \
  --max-instances=1 --quiet
```

Result was `[Trigger] done` and `allTrafficOnLatestRevision: true` on the first attempt, so no traffic fix was needed.

Upload the test image after the function is live and expect `NAME_64x64_thumbnail.jpg` next to the original.

## Note on the community script for this lab

It writes the old imagemagick version of the function, not the `sharp` version the lab supplies, and its iam section repeats the arc101 mistakes.
