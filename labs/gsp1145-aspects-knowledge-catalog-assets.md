# GSP1145, Create and Add Aspects to Knowledge Catalog Assets

Four tasks, **four** checkpoints, twenty five minutes stated. **Task 1 is scriptable, tasks 2 to 4 are console only** and worth doing there.

Third lab in the Knowledge Catalog set, and the only one with new content. `gsp1143-knowledge-catalog-console.md` and `gsp1144-knowledge-catalog-command-line.md` are both just the lake, zone, asset hierarchy; this one is about **aspects**, which is the part that does something.

No teardown task, so none of the delete last ordering trap that the other two have. Manual last updated and last tested June 2026, the freshest of the three.

## What an aspect actually is

Worth getting straight before clicking, because the lab uses "aspect", "aspect type" and "entry" as if they were interchangeable.

- An **entry** is a catalog record for something real: a BigQuery table, a column, a bucket.
- An **aspect type** is a reusable **template**. It defines named fields and their types, and lives in one region.
- An **aspect** is an **instance** of an aspect type attached to a particular entry.

So this lab defines one template, `Protected Data Aspect`, with a single required Enum field `Protected Data Flag` accepting `Yes` or `No`, and then stamps it onto one table and nine of its columns. Task 4 searches by aspect type, which is the payoff: the metadata you attached is now a query dimension.

## Task 1 from Cloud Shell

The `customers` BigQuery dataset is **pre-created** in this lab, unlike GSP1143 and GSP1144 where you make the storage yourself. So task 1 is three creates and nothing else.

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=europe-west4
gcloud config set compute/region $REGION
gcloud services enable dataplex.googleapis.com

for i in 1 2 3 4 5; do gcloud dataplex lakes create orders-lake --location=$REGION --display-name="Orders Lake" && break; echo "likely a 403 while the api settles, retrying in 30s"; sleep 30; done

gcloud dataplex zones create customer-curated-zone \
  --location=$REGION \
  --lake=orders-lake \
  --display-name="Customer Curated Zone" \
  --resource-location-type=SINGLE_REGION \
  --type=CURATED

gcloud dataplex assets create customer-details-dataset \
  --location=$REGION \
  --lake=orders-lake \
  --zone=customer-curated-zone \
  --display-name="Customer Details Dataset" \
  --resource-type=BIGQUERY_DATASET \
  --resource-name=projects/$PROJECT_ID/datasets/customers

gcloud dataplex assets list --location=$REGION --lake=orders-lake --zone=customer-curated-zone \
  --format="table(name.basename(),displayName,resourceSpec.name,state)"
```

**No discovery flags on either the zone or the asset**, which is different from GSP1143. This lab's property tables say to leave the remaining properties at their defaults and never mention metadata discovery for the zone, and the asset's setting is `Inherit`, which from the CLI means omitting the flags rather than passing them.

IDs are what the console derives from the display names: `orders-lake`, `customer-curated-zone`, `customer-details-dataset`. Click the checkpoint once the last line reports `ACTIVE`.

## Task 1 is almost decoupled from the rest of the lab

Worth noticing, because it changes how you debug tasks 3 and 4.

The entry the aspects get attached to is found by **Filters > Systems > BigQuery**, then searching `customer_details`. That entry comes from BigQuery's own catalog integration, which registers every table in the project automatically. It is **not** produced by the lake, the zone, the asset, or by zone discovery.

So the hierarchy built in task 1 exists to satisfy checkpoint 1 and nothing more. Tasks 2 through 4 would work in an empty project with just the dataset. Practical consequence: if `customer_details` does not turn up in the search, the problem is the search filter or the BigQuery table, never the lake.

## Task 2, the aspect type

`Manage Metadata > Metadata Types > Aspect types > Create`.

| Property | Value |
|---|---|
| Display Name | `Protected Data Aspect` |
| Location | `europe-west4` |
| Field Display Name | `Protected Data Flag` |
| Type | `Enum` |
| Is Required | ticked |
| Enum values | `Yes`, then `No` |

**Location has to match the region used everywhere else.** An aspect type is regional, and one created in the wrong region will not appear in task 3's Filter box or task 4's Aspect Types filter, which looks exactly like the aspect type failing to save.

Each enum value needs its own **Add an enum value** then **Done**. Two values means going round that loop twice before **Save**.

The CLI has `gcloud dataplex aspect-types create`, but it wants the field definitions as a JSON metadata template file, so for one enum field the console is less work.

## Task 3 applies the aspect twice

Easy to do half of this and think it is finished. Both halves are needed:

1. **On the entry.** Aspects section, next to `Optional aspects` click Add, filter for `protected data aspect`, pick it, set `Protected Data Flag` to `Yes`, Save.
2. **On the columns.** Schema tab, tick these nine, then Add aspect, pick the same type, `Yes`, Save.

```
zip
state
last_name
country
email
latitude
first_name
city
longitude
```

Nine, and the table has more columns than that, so this is a selection rather than select all. Tick them against the list rather than by eye.

Column level aspects are the reason not to script task 3. Doing it through the API means patching the entry with aspect keys addressed per column path, which is a lot of JSON for what is two clicks here.

## Both of the last checkpoints lag

The lab says so twice, once for the aspect type and once for the aspect attachment: *"It can take a few minutes ... before the progress check returns a successful message."*

Take that literally. A red checkpoint here is the expected first result, and the fix is waiting, not rebuilding. Clicking again immediately returns the same cached verdict, so give it two or three minutes, refresh the lab page, then click. See the refresh note in `gotchas.md`.

## Task 4 is verification only

`Discover > Search`, `Filters > Aspect Types`, tick `Protected Data Aspect`, OK. The `customer_details` table comes back, and its Schema tab shows the aspect on the nine columns.

No checkpoint of its own beyond what task 3 already scored, and nothing is created. It is the demonstration that the metadata added by hand is now searchable, which is the point of the whole lab.

## Related files

- `gsp1143-knowledge-catalog-console.md` for the console to `gcloud` field mapping used in the block above.
- `gsp1144-knowledge-catalog-command-line.md` for the same hierarchy end to end from the CLI, including its teardown.
- `arc117-organize-govern-data-knowledge-catalog-challenge.md`, the challenge lab, where the aspect goes on a **zone** rather than a table and its columns, so this lab's BigQuery search filter has to be cleared.
