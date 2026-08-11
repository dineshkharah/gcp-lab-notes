# GSP1144, Knowledge Catalog: Qwik Start, Command Line

Four tasks, **four** checkpoints, fifteen minutes stated and **five in practice**. Pure Cloud Shell, two pastes with three checkpoint clicks between them. One of the few labs in these notes that comes in well under its stated time.

The CLI twin of GSP1143, which is the same lab through the console. Different resource types: a **curated** zone here instead of raw, and a **BigQuery dataset** instead of a Cloud Storage bucket. `gsp1143-knowledge-catalog-console.md` carries the console field to flag mapping, which is what lets that lab be run from these same commands.

Manual last updated May 2026, **lab last tested April 2025**, a year apart.

## The ordering trap, which is the only real risk

**Task 4 deletes everything the first three checkpoints score.** Checkpoints 1, 2 and 3 want the lake, zone and asset present; checkpoint 4 wants them gone.

So the run is: one paste, click three checkpoints, second paste, click the fourth. Checkpoints latch once earned, so that works and there is no way back from pasting both blocks first.

## Block one, tasks 1 to 3

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=YOUR_REGION
gcloud config set compute/region $REGION
gcloud services enable dataplex.googleapis.com

for i in 1 2 3 4 5; do gcloud dataplex lakes create ecommerce --location=$REGION --display-name="Ecommerce" --description="Ecommerce Domain" && break; echo "likely a 403 while the api settles, retrying in 30s"; sleep 30; done

gcloud dataplex zones create orders-curated-zone \
    --location=$REGION \
    --lake=ecommerce \
    --display-name="Orders Curated Zone" \
    --resource-location-type=SINGLE_REGION \
    --type=CURATED \
    --discovery-enabled \
    --discovery-schedule="0 * * * *"

bq mk --location=$REGION --dataset orders

gcloud dataplex assets create orders-curated-dataset \
--location=$REGION \
--lake=ecommerce \
--zone=orders-curated-zone \
--display-name="Orders Curated Dataset" \
--resource-type=BIGQUERY_DATASET \
--resource-name=projects/$PROJECT_ID/datasets/orders \
--discovery-enabled
```

`europe-west1` in our run and it changes per run. Every create blocks until its operation finishes, so this is one paste and then waiting, no polling needed.

The retry loop covers the 403 the lab warns about: `Permission 'dataplex.lakes.create' denied` for a minute or two after `services enable` returns, because the permission has not propagated yet. It is **one physical line** so it survives the paste, and bounded at five attempts rather than looping forever on a real error. Same propagation family as the service agent problem in `gotchas.md`, same remedy of waiting rather than assuming the command is wrong.

Then click checkpoints 1, 2 and 3.

## Block two, task 4

Children first, each level refuses to delete while it still has any:

```
gcloud dataplex assets delete orders-curated-dataset --location=$REGION --zone=orders-curated-zone --lake=ecommerce --quiet
gcloud dataplex zones delete orders-curated-zone --location=$REGION --lake=ecommerce --quiet
gcloud dataplex lakes delete ecommerce --location=$REGION --quiet

gcloud dataplex lakes list --location=$REGION
bq ls
```

**`--quiet` on all three.** The lab says "If prompted to confirm, enter Y" three times, and a prompt inside a pasted block takes the next line of the paste as its answer, which is how ARC114 turned an API key into a comment.

The two list commands verify what checkpoint 4 checks: `Listed 0 items.` for the lakes, and `orders` still present in `bq ls`.

## The lab text contradicts itself about data loss

Under Detach an asset:

> This action does delete the underlying data in the BigQuery dataset. It simply removes the BigQuery dataset from being accessible or discoverable using the lake in Knowledge Catalog.

Those two sentences disagree. A `not` has been dropped from the first one. GSP1143's console version of the same paragraph reads "This action does not delete the underlying data", which settles it, and `bq ls` after the teardown confirms it directly: `assets delete` detaches and touches nothing else.

Worth reading rather than skimming, because as written it looks like a warning that the next command destroys data.

## What the three levels are

- **Lake**, the top domain, one per department or data domain. Regional.
- **Zone**, a subdomain inside a lake. **Raw** for untyped files such as Cloud Storage objects, **curated** for typed analytics data such as BigQuery datasets. `--resource-location-type=SINGLE_REGION` is the console's `Data locations: Regional`.
- **Asset**, a pointer to an existing bucket or dataset. **Nothing is copied or moved.** Attaching an empty dataset is fine and useful, because tables created in it later are picked up by discovery on the zone's schedule, hourly here via `--discovery-schedule="0 * * * *"`.

That no movement property is the reason the whole hierarchy can be built and torn down in five minutes: every object is metadata.

## Related files

- `gsp1143-knowledge-catalog-console.md`, the console twin, with the full field to flag mapping and the two entries that are easy to get wrong.
- `gsp1145-aspects-knowledge-catalog-assets.md`, the third in the set, where the hierarchy is only task 1 and the content is aspects.
