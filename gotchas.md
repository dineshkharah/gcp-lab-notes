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

## An interactive prompt in a pasted block eats the next line

The mirror image of the problem above. On ARC114 the community script contained `read -p "Enter your API Key: " API_KEY_INPUT`, and pasting the block meant `read` consumed the **next line of the script** as its answer. `API_KEY` became the literal string `# Get API Key`, the real key was then handed to bash as a command, and both curls sent a garbage key into files that were written anyway, containing a 403.

The tell is in the terminal, on the prompt line itself:

```
Enter your Google Cloud API Key: # Get API Key
```

Two habits that prevent it:

- Paste any `export VAR=...` **on its own line**, then verify with brackets around it: `echo "[$API_KEY]"`. Brackets so an empty or whitespace value is visible rather than looking like a blank line.
- Scan a script for `read`, `gunzip` on an existing file, `gcloud ... delete`, and anything else that prompts, and split the paste there.

The score pattern is diagnostic too: on ARC114 the two tasks that used the key both sat at 10 of 25 while the two that did not were full marks. When a subset of checkpoints fails and they share one input, suspect the input.

An editor counts as a prompt. `git pull` and `git merge` open one whenever the merge is not a fast forward, and `ssh-keygen` asks before overwriting an existing key file even when `-N ''` and `-f` have suppressed its other two questions. Both stop a pasted block dead. Cheap insurance at the top of any lab that uses git:

```
git config --global core.editor true
export GIT_MERGE_AUTOEDIT=no
```

## Always give kubectl rollout status a timeout

On GSP053 the lab says `kubectl rollout status` returns immediately after a `rollout pause`. It does not. It blocks forever on `Waiting for deployment ... 1 out of 3 new replicas have been updated`.

That matters more than it sounds, because the command was in the middle of a pasted block. Ctrl+C interrupted the status but the shell kept executing the remaining buffered lines, so `resume` and `undo` raced each other and the deployment finished on the wrong version with no clear record of what had run.

```
kubectl rollout status deployment/NAME --timeout=180s
```

The same applies to anything that can block indefinitely inside a paste. If it can wait forever, bound it, or run it on its own line.

## A multi line command substitution does not survive a paste

A wait loop split across lines looks fine and is not:

```
until [ "$(gcloud beta deploy rollouts list --delivery-pipeline web-app \
  --release web-app-001 --filter='targetId=test' --format='value(state)' | head -n 1)" = "SUCCEEDED" ]; do
```

Pasted into Cloud Shell it gives `-bash: command substitution: unexpected EOF while looking for matching ')'`, the comparison evaluates against an empty string, and the loop **exits immediately printing its success message**. On GSP1079 it echoed `staging rollout SUCCEEDED` while that rollout was in `FAILED`.

Keep an `until` or `while` on one physical line, or drop the loop and re-run the status command by hand. For a handful of checks, by hand is better and you actually read the output.

Related: an async resource is not a ready resource. The same lab creates three GKE clusters with `--async`, and calling `get-credentials` before they are `RUNNING` silently leaves one cluster with no kubectl context, which surfaces much later as a failed deployment with no mention of the real cause.

## A loop over a failed command substitution deletes nothing, silently

On GSP322 this line looked fine and is not valid:

```
gcloud compute firewall-rules list --filter="network:acme-vpc AND sourceRanges:0.0.0.0/0"
```

```
ERROR: Invalid value for field 'filter'. Invalid list filter expression.
```

Compute's list api accepts only a restricted filter grammar, and gcloud translated the compound expression into something it rejects. The `for` loop built on it iterated over an empty list, so the overly permissive rule was never deleted. Two error lines scrolled past inside a wall of successful output and the score came back 90 of 100 with no obvious cause.

For a handful of resources, **list them, read them, act by name.** If a destructive loop matters, echo the names first and confirm the list is not empty before the loop runs.

## Some checkpoints want an exact row count, so claim them in order

On GSP1049 each of the three data checkpoints wanted the table to hold exactly what the lab had created up to that point: two rows, then six, then a hundred and fifty thousand. Loading the csv before claiming the first two made both of them unpassable, and the only way back was deleting rows with partitioned dml, claiming each checkpoint at the right count, then reloading.

This is the concrete reason for the one task at a time rule. It is not tidiness. Racing ahead can put a lab into a state where an earlier checkpoint cannot be earned without destroying work.

Checkpoints latch once earned, which is what makes that kind of recovery safe. The Dataflow checkpoint stayed green while the rows it had checked were being deleted.

The blunter version of this is a lab whose **last task deletes what the earlier checkpoints score**. GSP1143 and GSP1144 both build a lake, a zone and an asset, score one checkpoint on each, and then have you tear all three down for a fourth. Scripting the whole lab and clicking afterwards leaves three checkpoints with nothing to find. Whenever a lab's final task is cleanup, claim everything before running it.

## In a two identity lab, check which configuration is active before clicking

GSP647 runs two `gcloud config configurations`, `default` for user 1 and `user2` for user 2. Its zone checkpoint failed with only `Please change the zone as instructed in the lab manual`, even though the zone had been set correctly, because `gcloud init` leaves the configuration it just created active and the scorer read that one instead.

Nothing had to be fixed. Activating `default` and clicking again passed.

```
gcloud config configurations list
```

That prints every configuration with its account, project and zone in one table. Run it before claiming anything in a lab with more than one principal. The other half of the same problem is running a scored command as the wrong identity, which fails in a way that looks like a permissions bug.

The console version of the same trap is a lab that hands out two or three sets of credentials with a console button for each, as GSP1041 does with three. **Close one console before opening the next**, which is what those labs tell you to do and it is not a formality. A second student account signed in over a live first one produces `Access Denied` on an object you just granted yourself, which reads like a broken grant rather than a browser session.

## Refresh the lab page before rebuilding anything

On ARC130 the api key checkpoint failed with `Please create an API key` while the key existed, was enabled, and was visible in the console. Nothing was wrong. After finishing the two later tasks and **refreshing the lab page**, that checkpoint and one other both went green on their own.

Two lessons. Clicking the same button again is not a retry, it usually returns the same cached verdict, so change something first: refresh the page, or produce a new signal the scorer can see. And a failing checkpoint is not proof the work is wrong.

Sometimes the lab admits this itself. GSP1145 warns twice that its aspect checkpoints take a few minutes to come good. Where a lab says that, a red checkpoint is the expected **first** result and waiting is the whole fix. Read the note before rebuilding the thing it is about.

The other half of that run: the Google Docs and Apps Script checkpoint also scored **without a document ever being created**, off the api traffic the earlier tasks had already generated. Worth clicking every checkpoint after a refresh before hand building the slowest task in a lab.

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

A rename does not have to reach the whole product. Knowledge Catalog, in GSP1143, is called that only in the console; the api is still `dataplex.googleapis.com`, the commands are still `gcloud dataplex`, and the permission in its error messages is still `dataplex.lakes.create`. So the console name and the command name disagree and neither is a typo. When a lab name and a command prefix do not match, check whether the product was renamed before assuming the lab is stale.

The date is also worth reading across labs that share tasks. GSP1041 and GSP1042 have three identical tasks, and their manuals are two months apart, so the same step is described with different menu labels in each. And the lag cuts the other way too: GSP1042 still calls the product Data Studio, renamed Looker Studio in 2022, which means searching the lab text for the name in the console finds nothing.

## Community scripts are a starting point, not an answer

The scripts floating around for these labs are usually built for a different variant of the same lab. Things found wrong in them: source bucket paths from an older lab id, a randomised api id where the lab requires a fixed one, an alert policy display name that does not match, a filename typo that makes the script exit immediately, an entire BigQuery export section for a lab that never mentions BigQuery, and iam grants aimed at the wrong service account. Read them, take the shape, check every literal against the lab text.

**Read the end of the script before running the start of it.** The ARC122 one finishes with an interactive `Would you like to cleanup resources? (y/N)` that deletes the api key and runs `gsutil -m rm -r` over the whole bucket, including the two response files the checkpoints score. A stray keystroke at that prompt undoes the lab. Scripts also tend to `exit 1` on failures that are not fatal, like an object acl call that a uniform access bucket refuses, which stops the run before any scored work happens.

## Skip Dataflow for small bulk loads

When a lab says to use Dataflow or a client library to load a few hundred rows, build one multi row insert instead. Five hundred rows of csv turned into a single `INSERT ... VALUES (...),(...)` finished in seconds against Spanner, where the bucket plus manifest plus template route was about twenty five minutes. Five hundred rows across three columns is fifteen hundred mutations, far under the per commit limit.
