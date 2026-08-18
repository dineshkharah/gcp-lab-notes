# GSP659, Deploy Your Website on Cloud Run

Six tasks, **five** checkpoints, twenty minutes stated and **no cost**. About **fifteen**, of which twelve are two Cloud Builds and `setup.sh`. **Pure Cloud Shell**, including the one part written as console clicks.

Manual and last tested both **August 2026**. Lab window was 59 minutes, which is comfortable but not idle time.

## The console step is one gcloud command

The lab has you create the Artifact Registry repository through the console. There is no reason to:

```
gcloud artifacts repositories create monolith-demo --repository-format=docker --location=$REGION --description="monolith demo"
```

Same name, format and location the lab specifies. This page carries the **Do not deviate from instructions** banner, and doing the lab's own step through the CLI instead of the UI is inside scope, as is dropping the unscored `npm start` preview, which removes work rather than adding any.

**The repository must exist before `builds submit`.** The lab says why and it is worth keeping: pushing an image does not create a repository, and the Cloud Build service account has no permission to create one. A missing repo fails the build with a permissions error rather than a not found, which sends you looking in the wrong place.

## The destructive step is the command directly below its own checkpoint

**Task 4 deploys with `--concurrency 1`, checkpoint 3 scores that value, and the lab's very next command sets concurrency back to 80.** Nothing is labelled cleanup; the restore reads as the next paragraph of the lesson. Click checkpoint 3 before running it.

Seventh instance of this shape in these notes and the tightest one. The gap between earning a checkpoint and destroying it is a single command.

## Every deploy needs --allow-unauthenticated

The lab says *"When asked to allow unauthenticated invocations to [monolith] type Y"*. In a pasted block that prompt eats the following line, per the standing note in `gotchas.md`. Passing the flag means nothing prompts, on all four deploys.

## The five blocks

Read the region off the lab page. `us-east4` on this run.

**Block 1**, tasks 1 and 2, claims checkpoint 1. Roughly eight minutes.

```
export REGION=us-east4
export PROJECT_ID=$(gcloud config get-value project)
gcloud services enable artifactregistry.googleapis.com cloudbuild.googleapis.com run.googleapis.com
gcloud artifacts repositories create monolith-demo --repository-format=docker --location=$REGION --description="monolith demo"
gcloud auth configure-docker $REGION-docker.pkg.dev --quiet
git clone https://github.com/googlecodelabs/monolith-to-microservices.git ~/monolith-to-microservices
cd ~/monolith-to-microservices && ./setup.sh
cd ~/monolith-to-microservices/monolith && gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:1.0.0
gcloud artifacts docker images list $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith
```

`builds submit` has to run from `~/monolith-to-microservices/monolith`, since it uploads the current directory and expects the `Dockerfile` there.

**Block 2**, task 3, claims checkpoint 2.

```
gcloud run deploy monolith --image $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:1.0.0 --region $REGION --allow-unauthenticated
export URL=$(gcloud run services describe monolith --region $REGION --format='value(status.url)')
curl -s -o /dev/null -w "site HTTP %{http_code}\n" $URL
echo $URL
```

**Block 3**, task 4, claims checkpoint 3. Stop here.

```
gcloud run deploy monolith --image $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:1.0.0 --region $REGION --allow-unauthenticated --concurrency 1
gcloud run revisions list --service monolith --region $REGION --format="table(metadata.name,spec.containerConcurrency)"
```

Two revisions, newest at concurrency `1`. Reading it from the table beats the lab's route of opening the Revisions tab and its own note that you may have to navigate away and back for current data.

**Blocks 4 and 5 can be run as one**, claiming checkpoints 4 and 5 together. Deploying `2.0.0` does not disturb the `2.0.0` image sitting in the registry, which is what checkpoint 4 reads.

```
gcloud run deploy monolith --image $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:1.0.0 --region $REGION --allow-unauthenticated --concurrency 80
cd ~/monolith-to-microservices/react-app/src/pages/Home && mv index.js.new index.js
grep -c "Fancy Fashion" ~/monolith-to-microservices/react-app/src/pages/Home/index.js
cd ~/monolith-to-microservices/react-app && npm run build:monolith
cd ~/monolith-to-microservices/monolith && gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:2.0.0 && gcloud run deploy monolith --image $REGION-docker.pkg.dev/$PROJECT_ID/monolith-demo/monolith:2.0.0 --region $REGION --allow-unauthenticated
gcloud run services describe monolith --region $REGION --format='value(spec.template.spec.containers[0].image)'
curl -s $URL | grep -o "Fancy Fashion &amp; Style Online" || echo "text not found, check the build"
```

Two deliberate details in that block.

**`grep -c` before the build.** It should print `1`. A `0` means the `mv` did not land, and everything after it would spend five minutes building and shipping the old bundle. Cheap check in front of the expensive step.

**`&&` between the build and the deploy.** Chained rather than newline separated, so a failed build cannot hand the deploy an image tag that does not exist.

The final `curl` is the verification that matters. The describe proving the tag is `2.0.0` only shows what was asked for; grepping the served HTML for the new headline proves `npm run build:monolith` actually regenerated the static files rather than the container shipping the previous bundle under a new tag.

## Cloud Shell needed a manual gcloud auth login

On this run the first block failed on credentials and `gcloud auth login` was needed by hand.

**Cause is a race, not a fault.** Cloud Shell raises an **Authorize** dialog the first time a command needs credentials, and a block pasted the instant the shell opens runs ahead of it. `gcloud auth login` is the same authorization done manually.

Nothing already done has to be redone, because the repository, the build and any deploy are server side. What does not survive is the shell's own state, so re-export after a reauth:

```
gcloud config get-value project
echo "REGION=$REGION PROJECT_ID=$PROJECT_ID"
```

Empty variables with a correct project means only the exports were lost. `$URL` is the one to watch, since the last line of the merged block silently prints nothing if it is unset.

## Related files

- `gsp328-serverless-applications-cloud-run-challenge.md` for Cloud Run as a challenge, including the auth flags and `roles/run.invoker`.
- `gsp304-docker-image-to-kubernetes-challenge.md` and `gsp305-scale-out-update-containerized-app-challenge.md` for the same build and rolling update on GKE instead.
- `gsp1131-artifact-registry-qwik-start.md` for Artifact Registry on its own.
- `gsp081-cloud-run-functions-console.md` and `gsp092-monitoring-and-logging-for-cloud-run-functions.md` for the functions side of Cloud Run.
