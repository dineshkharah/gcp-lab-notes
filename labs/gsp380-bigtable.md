# GSP380, Create and Manage Bigtable Instances challenge lab

Five tasks: instance with autoscaling, two tables loaded by Dataflow, a replication cluster, backup and restore, then delete everything. Thirty five to forty five minutes because the two Dataflow jobs dominate.

## The two things that cost the most time here

**Delete steps got pasted along with the task before them.** Task 5 deletes every table, the backup, and the instance, which is exactly the state task 4 is scored on. Task 4 had not been confirmed yet, so the whole thing had to be rebuilt. Cleanup belongs in its own paste, sent only after the checkpoint is green.

**A checkpoint sat at ten out of twenty for a reason that was only in the popup message.** It wanted a multi region bucket named after the project id. The bucket existed but was regional, and bucket location cannot be changed, so it had to be deleted and recreated with `--location=US`.

## Instance, tables, and clusters

```
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
export PROJECT_ID=$DEVSHELL_PROJECT_ID

gcloud bigtable instances create ecommerce-recommendations \
  --display-name="ecommerce-recommendations" --cluster-storage-type=SSD \
  --cluster-config=id=ecommerce-recommendations-c1,zone=$ZONE,autoscaling-min-nodes=1,autoscaling-max-nodes=5,autoscaling-cpu-target=60

gcloud bigtable instances tables create SessionHistory \
  --instance=ecommerce-recommendations --column-families=Engagements,Sales

gcloud bigtable instances tables create PersonalizedProducts \
  --instance=ecommerce-recommendations --column-families=Recommendations

gcloud bigtable clusters create ecommerce-recommendations-c2 \
  --instance=ecommerce-recommendations --zone=OTHER_ZONE_SAME_REGION \
  --autoscaling-min-nodes=1 --autoscaling-max-nodes=5 --autoscaling-cpu-target=60
```

Adding the second cluster while Dataflow is importing is safe, so do it during the wait.

Column families come from reading the schema table in the lab: Engagements and Sales for the session table, Recommendations for the products table.

## The Dataflow imports

Source files are under `gs://spls/gsp380/`. Community scripts point at `gs://cloud-training/OCBL377/`, which belongs to an older lab.

```
gcloud dataflow jobs run import-sessions \
  --region=$REGION \
  --gcs-location gs://dataflow-templates-$REGION/latest/GCS_SequenceFile_to_Cloud_Bigtable \
  --staging-location gs://$PROJECT_ID/temp \
  --parameters bigtableProject=$PROJECT_ID,bigtableInstanceId=ecommerce-recommendations,bigtableTableId=SessionHistory,sourcePattern=gs://spls/gsp380/retail-engagements-sales-00000-of-00001,mutationThrottleLatencyMs=0
```

Same shape again for `import-recommendations` against `PersonalizedProducts` and `retail-recommendations-00000-of-00001`.

Both jobs failed on the first run. The documented remedy worked:

```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

Wait about forty five seconds and submit again. Job names can be reused because the old jobs had terminated.

## Answering the two quiz questions from the data

```
cat > ~/.cbtrc <<EOF
project = $PROJECT_ID
instance = ecommerce-recommendations
EOF

cbt read SessionHistory prefix=SESSION_ID_FROM_LAB
cbt read PersonalizedProducts prefix=USER_ID_FROM_LAB
```

Reading the answer out of the data beats guessing from the options. In our run the purchased product for the red session was `red_skirt`.

## Backup, restore, then delete

```
gcloud bigtable backups create PersonalizedProducts_7 \
  --instance=ecommerce-recommendations --cluster=ecommerce-recommendations-c1 \
  --table=PersonalizedProducts --retention-period=7d

gcloud bigtable instances tables restore \
  --source=projects/$PROJECT_ID/instances/ecommerce-recommendations/clusters/ecommerce-recommendations-c1/backups/PersonalizedProducts_7 \
  --destination=PersonalizedProducts_7_restored --destination-instance=ecommerce-recommendations
```

Confirm task 4 is green, then delete the backup first, then the three tables, then the instance. The community script leaves the instance delete commented out, so task 5 never passes with it as written.

## Bigtable reports stale state right after bulk deletes

`gcloud bigtable instances list` returned zero items and `cbt` returned NotFound while the instance still existed. A minute later `instances create` said it already existed. Verify twice before concluding something was destroyed.
