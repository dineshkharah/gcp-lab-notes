# GSP1143, Knowledge Catalog: Qwik Start, Console

Four tasks, **four** checkpoints, fifteen minutes stated and about ten. Console lab by title, but **every step has a `gcloud` equivalent** and running tasks 3 and 4 from Cloud Shell is faster than the console.

Manual last updated May 2026, last tested April 2026.

## The one thing that can lose this lab

**Task 4 asks you to delete everything the first three checkpoints score.** Checkpoints 1, 2 and 3 want the lake, zone and asset to exist. Checkpoint 4 wants them gone.

Click 1, 2 and 3 before deleting anything. They latch once earned, so create, click three times, then tear down works fine and there is no way back if you script the whole lab and start clicking afterwards.

This is the same trap as GSP1049 with the row counts, except here the lab text itself instructs the destruction, which makes it easier to walk into.

## The product has three names

The console calls it **Knowledge Catalog**, under Navigation menu > View all products > Analytics. It was Dataplex Universal Catalog until recently. But the API, the CLI, and every IAM permission are still **dataplex**:

- Search the console marketplace for `Cloud Dataplex API` to enable it
- `gcloud services enable dataplex.googleapis.com`
- `gcloud dataplex lakes|zones|assets ...`
- the permission in the 403 is `dataplex.lakes.create`

So the console name and the command name do not match and neither is wrong. Searching the console for "Knowledge Catalog" finds the product, searching for the API by that name finds nothing.

## Console to CLI mapping

The console wizard fields translate exactly. Worth having, because the CLI version of this lab, GSP1144, uses different resource types and does not spell these out.

| Console field | Flag |
|---|---|
| Display Name | `--display-name` |
| ID (left as default) | the positional argument, derived from the display name |
| Region | `--location` |
| Zone Type, Raw or Curated | `--type=RAW` or `--type=CURATED` |
| Data locations, Regional | `--resource-location-type=SINGLE_REGION` |
| Enable metadata discovery | `--discovery-enabled` |
| Discovery settings, Inherit | **omit every discovery flag** |
| Type, Storage bucket | `--resource-type=STORAGE_BUCKET` |
| Bucket chosen in Browse | `--resource-name=projects/PROJECT/buckets/BUCKET` |

Two of those are easy to get wrong.

**The ID is not the display name.** The console derives it, so `temperature raw data` becomes `temperature-raw-data` and that is what every later `--zone=` needs. From the CLI the ID is the positional argument and the display name is a separate flag, and both have to match what the console would have produced in case the scorer checks either. Read them back rather than assuming:

```
gcloud dataplex lakes list --location=$REGION --format="table(name.basename(),displayName,state)"
gcloud dataplex zones list --location=$REGION --lake=sensors --format="table(name.basename(),displayName,state)"
```

**Inherit means the absence of a flag.** Passing `--discovery-enabled` on the asset sets an asset level override instead of inheriting the zone's setting, which is not what the lab asks for.

## Tasks 3 and 4 from Cloud Shell

Task 3, bucket named after the project id plus the asset:

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-east1

gcloud storage buckets create gs://$PROJECT_ID \
  --location=$REGION \
  --uniform-bucket-level-access \
  --public-access-prevention

gcloud dataplex assets create measurements \
  --location=$REGION \
  --lake=sensors \
  --zone=temperature-raw-data \
  --display-name="measurements" \
  --resource-type=STORAGE_BUCKET \
  --resource-name=projects/$PROJECT_ID/buckets/$PROJECT_ID
```

`--public-access-prevention` is the `Public access will be prevented` confirmation dialog. Uniform access is the default for new buckets now but stating it keeps the result identical to the console's.

Task 4, after the first three checkpoints are green. Children first, each level refuses while it still has any:

```
gcloud dataplex assets delete measurements --location=$REGION --zone=temperature-raw-data --lake=sensors --quiet
gcloud dataplex zones delete temperature-raw-data --location=$REGION --lake=sensors --quiet
gcloud dataplex lakes delete sensors --location=$REGION --quiet

gcloud dataplex lakes list --location=$REGION
gcloud storage ls
```

**`--quiet` on all three.** Each prompts for confirmation, and a prompt inside a pasted block eats the next line as its answer. It also covers the lake deletion, where the console makes you type the word `delete` into a box.

The two list commands verify what checkpoint 4 checks. `Listed 0 items.` from the first, and the bucket still present in the second, because `assets delete` only detaches.

## The 403 right after enabling the api

If tasks 1 and 2 are done from the CLI, `lakes create` can fail with `Status code: 403. Permission 'dataplex.lakes.create' denied` for a minute or two after `services enable` returns. The lab warns about it. Bounded retry on one physical line, so it survives a paste:

```
for i in 1 2 3 4 5; do gcloud dataplex lakes create sensors --location=$REGION --display-name="sensors" && break; echo "likely a 403 while the api settles, retrying in 30s"; sleep 30; done
```

Same family as the service agent propagation problem in `gotchas.md`, and the same remedy: give it time rather than assuming the command is wrong.

## Related files

- `gsp1144-knowledge-catalog-command-line.md` is this lab through the CLI, with a curated zone and a BigQuery dataset instead of a raw zone and a bucket. Same four checkpoints and the same delete last ordering trap, and it runs in five minutes.
- `gsp1145-aspects-knowledge-catalog-assets.md` builds the same hierarchy again in its task 1 and then does the part that actually uses it, attaching aspects to a table and its columns.
- `arc117-organize-govern-data-knowledge-catalog-challenge.md` is the challenge lab, this lab's raw zone and bucket plus an aspect on the zone.
