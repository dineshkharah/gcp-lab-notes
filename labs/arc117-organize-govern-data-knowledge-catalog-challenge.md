# ARC117, Organize and Govern Data with Knowledge Catalog: Challenge Lab

Three tasks, **three** checkpoints, twenty five minutes stated and about ten. **Tasks 1 and 2 are one paste**, task 3 is console. No teardown.

The badge closer for GSP1143, GSP1144 and GSP1145. It is GSP1143's hierarchy, a raw zone with a Cloud Storage bucket, plus GSP1145's aspects, with one change that matters.

Manual last updated and last tested June 2026.

## The change that matters: the aspect goes on the zone

GSP1145 attaches its aspect to a **BigQuery table and nine of its columns**. ARC117 attaches its aspect to the **zone**.

That single difference breaks the habit built in the guided lab, because GSP1145's search step has you set `Filters > Systems > BigQuery` before searching. Carry that filter over and searching for `Raw Event Data` returns nothing, which reads like the zone never got catalogued.

The zone entry's system is **Dataplex**, not BigQuery. So in `Discover > Search`, clear the Systems filter or set it to Dataplex, then search `Raw Event Data` and confirm the result row is the zone rather than the lake or the asset. Aspects section, next to `Optional aspects` click Add, filter for `protected raw data`, pick it, set the flag, Save.

The Aspects section only exists on the **entry** page reached through Search. The Manage lakes route to the same zone does not offer it.

## Tasks 1 and 2 from Cloud Shell

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-east4
gcloud config set compute/region $REGION
gcloud services enable dataplex.googleapis.com

for i in 1 2 3 4 5; do gcloud dataplex lakes create customer-engagements --location=$REGION --display-name="Customer Engagements" && break; echo "likely a 403 while the api settles, retrying in 30s"; sleep 30; done

gcloud dataplex zones create raw-event-data \
  --location=$REGION \
  --lake=customer-engagements \
  --display-name="Raw Event Data" \
  --resource-location-type=SINGLE_REGION \
  --type=RAW \
  --discovery-enabled

gcloud storage buckets create gs://$PROJECT_ID \
  --location=$REGION \
  --uniform-bucket-level-access \
  --public-access-prevention

gcloud dataplex assets create raw-event-files \
  --location=$REGION \
  --lake=customer-engagements \
  --zone=raw-event-data \
  --display-name="Raw Event Files" \
  --resource-type=STORAGE_BUCKET \
  --resource-name=projects/$PROJECT_ID/buckets/$PROJECT_ID

gcloud dataplex assets list --location=$REGION --lake=customer-engagements --zone=raw-event-data \
  --format="table(name.basename(),displayName,resourceSpec.name,state)"
```

Translating the challenge's prose into flags, since it states requirements rather than fields:

- **"regional raw zone"** is two flags, `--type=RAW` and `--resource-location-type=SINGLE_REGION`.
- **"regional asset"** needs no flag at all. An asset is regional because the bucket sits in one region and the zone is SINGLE_REGION. A multi region bucket is rejected by that zone, which is the only way this can go wrong.
- The bucket is named after the **project id**, as in GSP1143.
- IDs are the ones the console would derive from the display names: `customer-engagements`, `raw-event-data`, `raw-event-files`.

**Discovery is not mentioned by the challenge.** I enabled it on the zone because GSP1143 explicitly enables it on its raw zone, and the asymmetry favours it: enabling cannot fail a checkpoint that ignores discovery, while omitting could fail one that checks. On an empty bucket it does nothing visible either way.

## Task 3, the aspect type

| Property | Value |
|---|---|
| Display Name | `Protected Raw Data Aspect` |
| Location | `us-east4` |
| Field Display Name | `Protected Raw Data Flag` |
| Type | `Enum` |
| Enum values | `Y`, then `N` |

**`Y` and `N`, not `Yes` and `No`.** GSP1145 uses the long forms and this lab uses the short ones, which is exactly the kind of substitution a challenge lab makes on purpose.

Location has to be `us-east4`. Aspect types are regional and one created elsewhere will not appear in the filter box when attaching, which looks identical to the aspect type having failed to save.

`Is Required` is not specified by the challenge. Ticking it matches the guided lab and is harmless.

Give the checkpoint two or three minutes and refresh the lab page before re-clicking if it comes back red. GSP1145 warns that aspect attachment lags its scorer and the same applies here.

## Related files

- `gsp1143-knowledge-catalog-console.md` for the console to `gcloud` field mapping.
- `gsp1144-knowledge-catalog-command-line.md` for the hierarchy end to end from the CLI.
- `gsp1145-aspects-knowledge-catalog-assets.md` for aspect types and aspects, and the entry versus aspect type versus aspect distinction.
