# ARC119, Create a Secure Data Lake on Cloud Storage: Challenge Lab

Four tasks, **four** checkpoints, twenty minutes stated and about **ten**. **One paste covers tasks 1 to 3**, task 4 is console.

Manual last updated and last tested June 2026, and the page said *Updated 6 hours ago*, so this is the freshest lab in these notes.

## The tasks are drawn from a pool, so this file is one draw

The lab page says so outright: *"For each time when you start the lab, you get different tasks."* Before the lab starts, all four task bodies read `Dynamically selected task will show up here...`, so nothing can be prepared from the unstarted page beyond enabling the apis.

The draw recorded here was: a Cloud Storage bucket, a lake plus a raw zone, an entry group, and a tag template. Expect other draws to pull from the same badge, meaning the GSP1143, GSP1144 and GSP1145 material, so `arc117-organize-govern-data-knowledge-catalog-challenge.md` and `gsp1144-knowledge-catalog-command-line.md` are the files to open regardless of which draw appears.

## Two usernames, and they do not matter

The lab details panel gives **Username 1** and **Username 2**, and the task bodies say *"Sign into the project as User 1"* and *"Sign into the project as User 2."*

**In this draw there is no reason to switch.** Both students are project level identities, no task grants or tests a permission, and every checkpoint verifies that a resource exists rather than who created it. The whole thing ran in one Cloud Shell as User 1 and scored.

The two identity note in `gotchas.md` is about labs where a **grant** is scored, and there checking the active configuration before clicking genuinely matters. A draw that includes an IAM task would move this lab into that category, so the instruction is worth reading rather than assuming it is always decoration.

## Tasks 1 to 3 from Cloud Shell

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-east1
export BUCKET=$PROJECT_ID-bucket
gcloud config set compute/region $REGION
gcloud services enable dataplex.googleapis.com datacatalog.googleapis.com

gcloud storage buckets create gs://$BUCKET --location=$REGION --uniform-bucket-level-access

for i in 1 2 3 4 5; do gcloud dataplex lakes create customer-lake --location=$REGION --display-name="Customer-Lake" && break; echo "likely a 403 while the api settles, retrying in 30s"; sleep 30; done

gcloud dataplex zones create public-zone \
  --location=$REGION \
  --lake=customer-lake \
  --display-name="Public-Zone" \
  --type=RAW \
  --resource-location-type=SINGLE_REGION \
  --discovery-enabled \
  --labels=domain_type=source_data

gcloud dataplex assets create customer-bucket \
  --location=$REGION \
  --lake=customer-lake \
  --zone=public-zone \
  --display-name="Customer Bucket" \
  --resource-type=STORAGE_BUCKET \
  --resource-name=projects/$PROJECT_ID/buckets/$BUCKET \
  --discovery-enabled

gcloud dataplex entry-groups create custom-entry-group --location=$REGION --display-name="Custom entry group"
```

The retry loop on `lakes create` is the same 403 propagation window as GSP1144 and ARC117: `services enable` returns before `dataplex.lakes.create` is usable.

Four inferences the task text forces:

- **The bucket is `PROJECT_ID-bucket`.** ARC117's draw used the bare project id and this one appends `-bucket`. The pool substitutes names, so read it off the task rather than off these notes.
- **"Regional" is `--location=us-east1`.** Not `US`. A multi region bucket is refused by a `SINGLE_REGION` zone, so the asset step catches that mistake for you, which is a reason to create the asset before clicking anything.
- **The asset belongs to task 1, not task 2.** Task 1's last bullet is *"Use the same bucket you have created in the above step to attach as an asset to the zone"*, which cannot run until task 2's zone exists. So the tasks are not independent in the order they are numbered, despite the setup section claiming they are.
- **`ID: Leave the default value`** means use the id the console would derive from the display name, lowercased with the spaces or capitals flattened: `customer-lake`, `public-zone`.

The label `domain_type=source_data` goes on the **zone**, which is where the task text places the sentence. It scored there.

## The entry group id cannot be the name the lab gives

*"Entry Group Name: Custom entry group"*, with spaces, which no id accepts. So it is a display name and the id has to be invented: `custom-entry-group` for `gcloud dataplex`, and note that the older `gcloud data-catalog entry-groups` rejects hyphens and would need `custom_entry_group`. The dataplex form worked.

This is the same shape as *ID: Leave the default value* two tasks earlier, except here the lab does not tell you an id exists at all.

## Task 4 is a tag template, and the console has no tag templates

**The finding worth keeping from this lab.**

The task specifies a tag template name, a **tag template ID**, a location, and two typed fields, one Text and one Enum with values Yes and No. There is no tag templates page in the current console. The page that takes exactly those inputs is:

**Knowledge Catalog > Metadata types > Aspect types > Create**

Creating it there with the task's values cleared checkpoint 4. The old vocabulary still exists in the api, `gcloud data-catalog tag-templates create` runs fine, so the rename reached the ui and stopped. See the rename note in `gotchas.md`: when a lab names a console page that is not there, find the page whose **input fields** match rather than assuming the lab is describing a retired product.

The task text goes on to describe attaching the tag to the bucket's Cloud Storage entry and then searching by template. Creating the aspect type was sufficient for the checkpoint here, so those steps read as narrative rather than as scored work in this draw. Doing them anyway costs a couple of minutes and cannot hurt; if the checkpoint were to come back red, that is where to look next, and the trap would be ARC117's, a stale **source system** filter hiding the bucket entry because the entry sits under Cloud Storage rather than Dataplex or BigQuery.

## Related files

- `arc117-organize-govern-data-knowledge-catalog-challenge.md`, the closest neighbour, and the source of the lake plus raw zone plus bucket asset block reused here.
- `gsp1144-knowledge-catalog-command-line.md` for the hierarchy from the CLI, and for what lakes, zones and assets each mean.
- `gsp1143-knowledge-catalog-console.md` for the console field to flag mapping, useful when a draw has to be clicked.
- `gsp1145-aspects-knowledge-catalog-assets.md` for aspect types and aspects, which is what task 4 turns out to be.
- `cloud-storage-three-bucket-lab.md` for bucket creation flags on their own.
