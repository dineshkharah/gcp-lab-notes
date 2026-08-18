# ARC100, Store, Process, and Manage Data on Google Cloud: Challenge Lab

Four tasks, **four** checkpoints, fifteen minutes stated and **no cost**. Took most of a 44 minute window, and almost all of that went on one trap described below.

Manual and last tested both April 2025.

## This is the third variant of the same lab

| Lab | Notes |
|---|---|
| `arc101-monitor-and-manage-resources.md` | adds a shared bucket and an alerting policy, older `imagemagick-stream` code |
| `gsp315-store-process-and-manage-data.md` | same three resources, `sharp` code, deployed first try |
| **ARC100** | same three resources, `sharp` code, console flavoured instructions |

Read GSP315 first if this comes up again. It has the working order. What follows is what ARC100 adds.

**Fully Cloud Shell**, despite the course being called Console.

## What this variant changes

**The code blanks are already filled in.** GSP315 hands you `functions.cloudEvent('')` and `const topicName = ""` to complete. ARC100 supplies `functions.cloudEvent('memories-thumbnail-generator', ...)` and `const topicName = "memories-topic-342"` already populated, so the code is paste verbatim.

Values on this run: region `us-east1`, zone `us-east1-d`, bucket `memories-bucket-PROJECT_ID`, topic `memories-topic-342`, function `memories-thumbnail-generator`. Checkpoint weights are **30, 30, 40**, with tasks 3 and 4 sharing the last one.

## The trap that cost the run, and it was in my own command

The deploy failed repeatedly with:

```
[Service] done
[Trigger] failed
The Cloud Storage service account for your bucket is unable to publish to
Cloud Pub/Sub topics in the specified project.
```

The diagnosis was right: the GCS service agent needs `roles/pubsub.publisher`. The **grant** was the problem. Following the advice in `gotchas.md` to read an agent address back from the api rather than composing it, this was used:

```
export GCS_SA=$(gcloud storage service-agent --project=$PROJECT_ID)
```

`gcloud storage service-agent` **does not return a bare email**. It returned a formatted block with a leading newline and indentation:

```
gcs agent=[
  service-642966183496@gs-project-accounts.iam.gserviceaccount.com]
```

So `$GCS_SA` was multi line, `add-iam-policy-binding` rejected it with `unrecognized arguments`, **and that one line's failure scrolled past four successful grants in the same block**. Every subsequent deploy failed identically because the role was never applied.

What worked, using the address the failed command had already revealed:

```
export GCS_SA=service-$PN@gs-project-accounts.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT_ID --member=serviceAccount:$GCS_SA --role=roles/pubsub.publisher --quiet --format="value(etag)"
gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --filter="bindings.members:gs-project-accounts" --format="value(bindings.role)"
```

`roles/pubsub.publisher` on that last line, then the deploy succeeded on the next attempt.

**The lesson is not that composing addresses is fine.** It is that a variable taken from a command has to be **proved usable**, with `echo "[$VAR]"` so that a newline or padding is visible, before anything depends on it. Now a qualification in `gotchas.md` on the advice that produced this.

## A bounded retry is for propagation, not for a missing prerequisite

Five deploy attempts, thirty seconds apart, every one returning the same permission error. **Roughly fifteen minutes bought nothing**, because the cause was static: the binding did not exist.

The retry habit came from GSP523, where a correctly applied binding simply was not usable yet. The distinguishing question is whether `get-iam-policy` actually lists the role:

- **listed but rejected** means propagation, so retry
- **not listed** means the grant failed, so fix the grant and do not retry

One `get-iam-policy` before the first deploy answers it in a second. Also a `gotchas.md` change.

## Three smaller findings

**`[Service] done` with `[Trigger] failed` means the function exists and does nothing.** The code built, the Cloud Run service is live, and `gcloud functions list` shows it. Only the Eventarc trigger is missing, so the function never fires. The final state before the fix said so outright:

```
[ERROR] Eventarc trigger .../triggers/memories-thumbnail-generator-264505 for the
function was not found. The function may not work correctly. Please redeploy.
```

So verify the trigger, not the function:

```
gcloud eventarc triggers list --location=$REGION --format="table(name,destination)"
```

**Failed deploys strand traffic on old revisions.** The retries logged `A new revision will be deployed serving with 0% traffic` and a rollback warning. After a deploy that follows failures:

```
gcloud run services update-traffic $FUNCTION --region=$REGION --to-latest
```

**A Cloud Shell restart wiped every export mid run**, which produced `--trigger-bucket: Invalid value ''` and `gs:///` on the upload. Unset variables interpolate as nothing rather than erroring. Second instance of this in these notes after GSP659; the recovery is re-exporting, and nothing server side is lost. `~/thumbnail` survived, since the home directory persists.

## The working order

Blocks 1 and 2 exactly as in `gsp315-store-process-and-manage-data.md`, with these amendments:

1. Grant `roles/pubsub.publisher` to `service-$PN@gs-project-accounts.iam.gserviceaccount.com` and **verify it with `get-iam-policy` before deploying**. This is the single binding the whole lab turns on.
2. Also grant `roles/iam.serviceAccountTokenCreator` to the Pub/Sub agent, `roles/eventarc.serviceAgent` to the Eventarc agent, and `roles/eventarc.eventReceiver` plus `roles/artifactregistry.reader` to `$PN-compute@developer.gserviceaccount.com`.
3. Create both service agents first with `gcloud beta services identity create` for pubsub and eventarc.
4. Deploy:

```
gcloud functions deploy $FUNCTION --gen2 --runtime=nodejs22 --region=$REGION --source=. --entry-point=$FUNCTION --trigger-bucket=$BUCKET --trigger-location=$REGION --max-instances=1 --quiet
```

5. `update-traffic --to-latest` if any attempt failed, then confirm a trigger exists.
6. Upload a **new** file. `travel.jpg` and `travel2.jpg` were both uploaded before the trigger existed and neither was ever processed; `travel3.jpg` was the one that produced a thumbnail. An object that predates its trigger is invisible to it, which is the standing note from ARC101.

The heredoc for `index.js` must be quoted, `<<'EOF'`, since the file is full of `${bucketName}` and `${fileName}` that bash would otherwise blank.

## Related files

- `gsp315-store-process-and-manage-data.md` and `arc101-monitor-and-manage-resources.md`, the other two variants.
- `gsp080-cloud-run-functions-command-line.md` for a gen2 function whose own `actAs` grant is missing, and for the Pub/Sub agent prompt that defaults to the failing answer.
- `gotchas.md`, the service agents section and the binding that exists section, both of which this run refined.
