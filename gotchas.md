# Traps that show up again and again

## Service agents do not exist until you create them

The single biggest time sink found so far. Enabling an api does not create its service agent. Granting a role to `service-<project number>@gcp-sa-pubsub.iam.gserviceaccount.com` then fails with:

```
INVALID_ARGUMENT: Service account service-NNN@gcp-sa-pubsub.iam.gserviceaccount.com does not exist.
```

That one line scrolls past while every other grant in the same block reports success, so the policy quietly ends up incomplete. The deploy later dies with "Permission denied while using the Eventarc Service Agent", which reads like a propagation delay and is not one. Retrying does nothing.

Create the agents before granting anything:

```
gcloud beta services identity create --service=pubsub.googleapis.com --project=$PROJECT_ID
gcloud beta services identity create --service=eventarc.googleapis.com --project=$PROJECT_ID
```

An agent created this way does not pick up its own service agent role, so also grant `roles/eventarc.serviceAgent` to the eventarc one. Then wait about two minutes before deploying.

Same pattern turns up whenever a lab says an agent is "created automatically". GSP526 says exactly that about the Privileged Access Manager agent and then asks you to grant it a role. Read the address back out of the create command instead of composing it by hand:

```
export PAM_SA=$(gcloud beta services identity create \
  --service=privilegedaccessmanager.googleapis.com \
  --project=$PROJECT_ID --format='value(email)')
echo $PAM_SA
```

A variable that came from the api cannot be a typo. GSP499 is the one exception found so far: its IAP agent does already exist, and only the `roles/run.invoker` binding is missing.

## Always prove an iam binding landed

Exit status is not enough. Check the policy:

```
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:ROLE_HERE" \
  --format="table(bindings.members)"
```

An empty result means the grant never applied, whatever the command printed.

## The project id in a screenshot is truncated

Qwiklabs project ids end in twelve hex characters. The credentials panel clips the field, so `qwiklabs-gcp-04-32ae3d1e5816` shows as `qwiklabs-gcp-04-32ae3d1e58`. Typing the short version into `gcloud config set project` produced a solid fifteen minutes of `PERMISSION_DENIED` and `CONSUMER_INVALID` errors that looked exactly like missing iam or a disabled api. Use `$DEVSHELL_PROJECT_ID`.

## Commands go missing out of long pasted blocks

Four separate labs had a checkpoint stuck at zero purely because its command never executed, while everything around it worked. `gcloud scheduler jobs run`, `gcloud storage buckets create`, and a Spanner `INSERT` were the ones that vanished. Before investigating config, search the terminal output for the command itself. The commands most likely to be lost are the ones that print little or nothing of their own, since there is no gap in the output to notice.

## Some checkpoints want an exact row count, so claim them in order

On GSP1049 each of the three data checkpoints wanted the table to hold exactly what the lab had created up to that point: two rows, then six, then a hundred and fifty thousand. Loading the csv before claiming the first two made both of them unpassable, and the only way back was deleting rows with partitioned dml, claiming each checkpoint at the right count, then reloading.

This is the concrete reason for the one task at a time rule. It is not tidiness. Racing ahead can put a lab into a state where an earlier checkpoint cannot be earned without destroying work.

Checkpoints latch once earned, which is what makes that kind of recovery safe. The Dataflow checkpoint stayed green while the rows it had checked were being deleted.

## In a two identity lab, check which configuration is active before clicking

GSP647 runs two `gcloud config configurations`, `default` for user 1 and `user2` for user 2. Its zone checkpoint failed with only `Please change the zone as instructed in the lab manual`, even though the zone had been set correctly, because `gcloud init` leaves the configuration it just created active and the scorer read that one instead.

Nothing had to be fixed. Activating `default` and clicking again passed.

```
gcloud config configurations list
```

That prints every configuration with its account, project and zone in one table. Run it before claiming anything in a lab with more than one principal. The other half of the same problem is running a scored command as the wrong identity, which fails in a way that looks like a permissions bug.

## Read the scorer message

The score alone says a checkpoint failed. The message under it says why. One Bigtable checkpoint sat at ten out of twenty with the real reason only in the popup: it wanted a multi region bucket named after the project id, while the bucket had been created regional.

## Bucket location is not changeable

If a checkpoint wants multi region and the bucket is regional, the only fix is delete and recreate. Pass `--location=US` for multi region and a region name like `--location=us-east4` for regional. Also note that a newer region name can be valid for Dataflow and BigQuery while being rejected by Cloud Storage. `us-west8` returned `HTTPError 403: Permission denied on 'locations/us-west8' (or it may not exist)` for a bucket but worked fine as a Dataflow region.

## Order matters for Dataflow

Create the staging bucket before submitting the job. Submitting first makes the job fail during staging and it takes about seven minutes to surface as `Failed`, which is a slow way to learn nothing.

## Publish test messages only once the pipeline is running

Templates that take an `inputTopic` create their own subscription when the pipeline starts. Anything published while the job is still `Pending` is dropped. Wait for `Running`, then publish.

## Dataflow jobs fail on the first run more often than not

The documented remedy works and is worth doing up front rather than after a failure:

```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

Wait about forty five seconds, then submit. Job names can be reused once the old job has terminated.

## Cloud Run functions need gen2 for these scorers

`--gen2` is not optional. A gen1 function deploys and runs correctly and the scorer still refuses it. Also, after a redeploy that follows failed attempts, traffic can stay pinned to an old revision. Fix with:

```
gcloud run services update-traffic FUNCTION_NAME --region=$REGION --to-latest
```

## Thumbnail style functions only fire on new uploads

An object uploaded before the trigger existed will never be processed. Upload again after the function is live.

## Labs get rewritten onto different products, so check the manual date

GSP499 used to be an App Engine lab with `gcloud app deploy`, a permanent App Engine region, and an OAuth consent screen that had to be configured before IAP would turn on. The current version is Cloud Run, with none of those three things in it. Every warning carried over from the old version was wrong, and two of them were warnings about irreversible mistakes that no longer exist.

Every lab page prints **Manual Last Updated** at the bottom. Read it against the notes here. Where they disagree the lab text wins, including against this file.

## Community scripts are a starting point, not an answer

The scripts floating around for these labs are usually built for a different variant of the same lab. Things found wrong in them: source bucket paths from an older lab id, a randomised api id where the lab requires a fixed one, an alert policy display name that does not match, a filename typo that makes the script exit immediately, an entire BigQuery export section for a lab that never mentions BigQuery, and iam grants aimed at the wrong service account. Read them, take the shape, check every literal against the lab text.

## Skip Dataflow for small bulk loads

When a lab says to use Dataflow or a client library to load a few hundred rows, build one multi row insert instead. Five hundred rows of csv turned into a single `INSERT ... VALUES (...),(...)` finished in seconds against Spanner, where the bucket plus manifest plus template route was about twenty five minutes. Five hundred rows across three columns is fifteen hundred mutations, far under the per commit limit.
