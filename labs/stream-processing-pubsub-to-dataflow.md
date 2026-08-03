# Stream Processing with Cloud Pub/Sub and Dataflow qwik start

Not a challenge lab, full instructions given. Five tasks: project resources, review the code, start the pipeline, watch it, clean up. Twenty five to thirty minutes. This is the source lab for the streaming data lake challenge lab.

Everything runs from Cloud Shell. Task 4 says to use the Dataflow console but `gcloud dataflow jobs list` and `gcloud storage ls` cover it, and the lab offers that as the alternative anyway.

## Python beats the Java path

The lab offers both. `mvn compile exec:java` pulls the whole Beam Java dependency tree and takes five minutes before it even submits. Python is one `pip install -r requirements.txt` and a single script.

## App Engine region names differ from compute region names

Cloud Scheduler needs an App Engine app first. For most regions the App Engine region is the same string, but `us-central1` becomes `us-central` and `europe-west1` becomes `europe-west`. Getting this wrong blocks scheduler creation entirely. `us-east4` needed no translation.

## Setup and task 1

```
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET_NAME="${PROJECT_ID}-bucket"
export TOPIC_ID=my-id
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
export AE_REGION=APP_ENGINE_REGION

gcloud config set compute/region $REGION

gcloud services disable dataflow.googleapis.com --project $PROJECT_ID --force
gcloud services enable dataflow.googleapis.com --project $PROJECT_ID

gcloud storage buckets create gs://$BUCKET_NAME
gcloud pubsub topics create $TOPIC_ID
gcloud app create --region=$AE_REGION
gcloud services enable cloudscheduler.googleapis.com

gcloud scheduler jobs create pubsub publisher-job --schedule="* * * * *" \
  --topic=$TOPIC_ID --message-body="Hello!" --location=$REGION

gcloud scheduler jobs run publisher-job --location=$REGION
```

The api restart can return a 429 on `serviceusage.googleapis.com` because lab provisioning has used up the mutate quota for the minute. Wait a minute and run just that pair again.

`jobs run` is its own scored checkpoint and is easy to lose in a long paste. Confirm it with:

```
gcloud scheduler jobs describe publisher-job --location=$REGION \
  --format="value(state,status,lastAttemptTime)"
```

## Task 3, the pipeline

```
cd ~
git clone https://github.com/GoogleCloudPlatform/python-docs-samples.git
cd python-docs-samples/pubsub/streaming-analytics

python3 -m venv env
source env/bin/activate
pip install -q -U pip
pip install -q -r requirements.txt

python PubSubToGCS.py \
  --project=$PROJECT_ID \
  --region=$REGION \
  --input_topic=projects/$PROJECT_ID/topics/$TOPIC_ID \
  --output_path=gs://$BUCKET_NAME/samples/output \
  --runner=DataflowRunner \
  --window_size=2 \
  --num_shards=1 \
  --temp_location=gs://$BUCKET_NAME/temp \
  --worker_machine_type=e2-standard-2 \
  --worker_disk_type=compute.googleapis.com/projects/$PROJECT_ID/zones/$ZONE/diskTypes/pd-standard
```

`worker_disk_type` needs the full resource url. The bare string `pd-standard` gets ignored or rejected, which is why the lab hands you the zone.

The command runs in the foreground and streams job messages forever, because it is a streaming pipeline. It looks stuck and is not. Watch progress from a second tab.

## Task 4, checking output

```
gcloud storage ls gs://$BUCKET_NAME/samples/
```

Files appear as `output-14:36-14:38-0`, one per two minute window, starting about four minutes after workers finish startup.

`gcloud dataflow jobs describe` does not include the `environment` block in its default view, so it cannot be used to confirm the machine type. Use the `Worker configuration: e2-standard-2 in ZONE` line in the job messages instead.
