# GSP695, Manage Storage Configuration using gsutil

Seven tasks, **four** checkpoints, ten minutes stated and **no cost**. About **five minutes**. Entirely the lab's own terminal.

Third of the five required labs in the **Essential Google Cloud CLI Tools** course.

Manual last updated **May 2026**, lab last tested **May 2024**. Two years apart, and the gap is visible: the lab has been converted to `gcloud storage` almost everywhere and one `gsutil` command was left behind.

## Checkpoints are tasks 1, 2, 4 and 6

Tasks 3, 5 and 7 have none. That matters most for **task 5**, which is the one likely to fail.

## The commands

```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
cd training-data-analyst/blogs
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET=${PROJECT_ID}-bucket

gcloud storage buckets create --default-storage-class=MULTI_REGIONAL gs://${BUCKET}
gcloud storage buckets describe gs://${BUCKET} --format="value(name,location,storageClass,uniformBucketLevelAccess.enabled)"
```
Checkpoint 1.

```
gcloud storage cp -r endpointslambda gs://${BUCKET}
gcloud storage ls gs://${BUCKET}/*
```
Checkpoint 2.

```
mv endpointslambda/Apache2_0License.txt endpointslambda/old.txt
rm endpointslambda/aeflex-endpoints/app.yaml
gcloud storage rsync -d -r endpointslambda gs://${BUCKET}/endpointslambda
gcloud storage ls -r "gs://${BUCKET}/endpointslambda/**"
```
Checkpoint 3. **Read that final listing before clicking.** See below.

```
gcloud storage cp --storage-class=NEARLINE ghcn/ghcn_on_bq.ipynb gs://${BUCKET}
gcloud storage ls -L gs://${BUCKET}/ghcn_on_bq.ipynb | head -20
```
Checkpoint 4.

The `buckets describe` in block 1 is worth the second: it confirms the storage class landed **and** reports whether uniform bucket level access is on, which is what decides task 5.

## The rsync vanished out of the pasted block

The one thing that went wrong, and it is instructive because everything looked right.

`mv`, `rm` and `rsync` were pasted together. Afterwards:

```
$ ls endpointslambda/
README.md  aeflex-endpoints  lambdafunctioninline.py  old.txt
$ ls endpointslambda/aeflex-endpoints/
main.py  openapi.yaml  requirements.txt
```

Rename done, `app.yaml` gone. Local state perfect. So the obvious conclusion was that the sync had worked and the checkpoint was wrong.

The bucket said otherwise:

```
$ gcloud storage ls -r "gs://${BUCKET}/endpointslambda/**"
.../endpointslambda/Apache2_0License.txt
.../endpointslambda/aeflex-endpoints/app.yaml
...
$ gcloud storage ls "gs://${BUCKET}/endpointslambda/old.txt"
ERROR: (gcloud.storage.ls) One or more URLs matched no objects.
```

Byte for byte the original upload. The `rsync` between the `rm` and the listing **never ran**, which is the missing command problem in `gotchas.md`, now with a fifth instance. Re-running it alone fixed it immediately.

**The generalisable bit: a correct source proves nothing about whether the transfer ran.** Where a block makes a local change and then pushes it somewhere, list **both** sides afterwards. Three commands localised the failure in one step:

```
gcloud storage ls "gs://${BUCKET}/endpointslambda/old.txt"
gcloud storage ls "gs://${BUCKET}/endpointslambda/Apache2_0License.txt"
gcloud storage ls "gs://${BUCKET}/endpointslambda/aeflex-endpoints/app.yaml"
```

The checkpoint wants the first to succeed and the other two to match nothing.

**`-d` is what earns the checkpoint**, not the rename. Without it the rename reads as an addition and the old name stays in the bucket, so `Apache2_0License.txt` would persist and the scorer would still complain.

## Task 5 is the stale command, and it is unscored

```
gsutil -m acl set -R -a public-read gs://${BUCKET}
```

Same command that failed on GSP364. Uniform bucket level access refuses object ACLs outright. Everything else in this lab is `gcloud storage`; this one line was not converted.

There is **no checkpoint for task 5**, so a failure costs nothing. If you want the public read anyway, the modern equivalent works whether or not uniform access is on:

```
gcloud storage buckets add-iam-policy-binding gs://${BUCKET} --member=allUsers --role=roles/storage.objectViewer
curl -s -o /dev/null -w "public read: HTTP %{http_code}\n" http://storage.googleapis.com/${BUCKET}/endpointslambda/old.txt
```

`200` means public, `403` means public access prevention is enforced, in which case skip it.

## Task 7, and one paste hazard

```
gcloud storage ls -L -r gs://${BUCKET} | head -60
```

The lab pipes to `| more`, which is interactive and would leave you pressing space inside a pasted block. `head` gives the same information without stopping.

Expect `ghcn_on_bq.ipynb` at **NEARLINE** and everything under `endpointslambda/` at **MULTI_REGIONAL**. That contrast is the whole point of tasks 6 and 7, and note the storage class is a per object property, not just a bucket default.

## Related files

- `gsp693-gcloud-cli-beginners-guide.md` and `gsp694-gcloud-for-network-configuration.md`, the first two labs in this course.
- `gsp364-managed-prometheus-challenge.md` for the same `gsutil acl set` failure inside a challenge lab, where it did cost a checkpoint.
- `arc111-cloud-storage-data-protection-challenge.md` and `gsp315-store-process-and-manage-data.md` for buckets in other contexts.
