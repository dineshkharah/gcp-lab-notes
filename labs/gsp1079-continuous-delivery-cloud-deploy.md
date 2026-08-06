# GSP1079, Continuous Delivery with Google Cloud Deploy

Nine tasks, eight checkpoints. Twenty minutes stated, more like forty in practice. Three GKE clusters and a Skaffold build dominate the clock.

The challenge lab for this badge, GSP393, is this lab with two clusters plus a v2 rebuild and a rollback. See `gsp393-cicd-pipelines-challenge.md`.

## The one thing that decides whether this goes smoothly

**Do not touch the clusters until all three read `RUNNING`.**

The clusters are created `--async` so the lab can move on, and everything up to and including the delivery pipeline works fine while they provision. But `get-credentials` and the namespace apply do not, and their failures are quiet enough to scroll past:

```
ERROR: cluster staging is missing endpoint. Is it still PROVISIONING?
error: context "staging" does not exist
error: ... dial tcp 34.32.235.135:443: connect: connection refused
```

Result in our run: `test` got its `web-app` namespace, `staging` got no kubectl context at all, and `prod`'s namespace apply was refused. The staging rollout then went `FAILED`, because Cloud Deploy had no namespace to deploy into. Nothing in the rollout error points back at a namespace.

Check first, every time:

```
gcloud container clusters list --format="csv(name,status)"
```

Three `RUNNING` lines, then continue. The community script tries to automate this wait and gets it wrong; see the bottom of this file.

## Sequence that overlaps the two slow things

Clusters take five to eight minutes, the Skaffold build about five. They are independent, so start the clusters first and build while they provision.

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
gcloud config set compute/region $REGION
gcloud config set deploy/region $REGION

gcloud services enable container.googleapis.com clouddeploy.googleapis.com \
  artifactregistry.googleapis.com cloudbuild.googleapis.com

gcloud container clusters create test --node-locations=$ZONE --num-nodes=1 --async
gcloud container clusters create staging --node-locations=$ZONE --num-nodes=1 --async
gcloud container clusters create prod --node-locations=$ZONE --num-nodes=1 --async

gcloud artifacts repositories create web-app \
  --description="Image registry for tutorial web app" \
  --repository-format=docker --location=$REGION
```

Checkpoints 1 and 2 score here. The cluster one passes on the clusters **existing**, not on them being ready.

```
cd ~/
git clone https://github.com/GoogleCloudPlatform/cloud-deploy-tutorials.git
cd cloud-deploy-tutorials
git checkout c3cae80 --quiet
cd tutorials/base

envsubst < clouddeploy-config/skaffold.yaml.template > web/skaffold.yaml
cat web/skaffold.yaml

gsutil mb -p $PROJECT_ID gs://${PROJECT_ID}_cloudbuild

cd web
skaffold build --interactive=false \
  --default-repo $REGION-docker.pkg.dev/$PROJECT_ID/web-app \
  --file-output artifacts.json
cd ..
cat web/artifacts.json | jq
```

`cat web/skaffold.yaml` must show the real project id under `googleCloudBuild.projectId`. Still reading `{{project-id}}` means `envsubst` did nothing and the build will fail.

Then the pipeline:

```
cp clouddeploy-config/delivery-pipeline.yaml.template clouddeploy-config/delivery-pipeline.yaml
gcloud beta deploy apply --file=clouddeploy-config/delivery-pipeline.yaml
gcloud beta deploy delivery-pipelines describe web-app
```

`Unable to get target test / staging / prod` in that output is **expected**. The targets do not exist yet.

## Only now, the clusters

```
gcloud container clusters list --format="csv(name,status)"
```

Wait for three `RUNNING`, then:

```
CONTEXTS=("test" "staging" "prod")
for CONTEXT in ${CONTEXTS[@]}; do gcloud container clusters get-credentials ${CONTEXT} --region ${REGION}; kubectl config rename-context gke_${PROJECT_ID}_${REGION}_${CONTEXT} ${CONTEXT} 2>/dev/null; done

for CONTEXT in ${CONTEXTS[@]}; do kubectl --context ${CONTEXT} apply -f kubernetes-config/web-app-namespace.yaml; done

for CONTEXT in ${CONTEXTS[@]}; do envsubst < clouddeploy-config/target-$CONTEXT.yaml.template > clouddeploy-config/target-$CONTEXT.yaml; gcloud beta deploy apply --file clouddeploy-config/target-$CONTEXT.yaml; done
```

**Three `namespace/web-app created` lines and no errors** is the gate. Anything else means a rollout will fail later.

The clusters are **regional**, created with `--node-locations` while `compute/region` was set, so `get-credentials` takes `--region` and the context name is `gke_PROJECT_REGION_NAME`.

## Release and promotions

```
gcloud beta deploy releases create web-app-001 \
  --delivery-pipeline web-app --build-artifacts web/artifacts.json --source web/
```

The first rollout is slow because Cloud Deploy renders manifests for **all three** targets at release creation, not just the first.

```
gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 \
  --region $REGION --format="table(name.basename(),targetId,state,approvalState)"
```

Re-run that by hand rather than looping, for the reason in the next section.

```
gcloud beta deploy releases promote --delivery-pipeline web-app --release web-app-001 \
  --region $REGION --to-target staging --quiet

gcloud beta deploy releases promote --delivery-pipeline web-app --release web-app-001 \
  --region $REGION --to-target prod --quiet
```

`--quiet` accepts the confirmation prompt, which is what makes these safe inside a pasted block. `--to-target` makes the destination explicit instead of relying on pipeline position, and it let us go straight to prod while the staging rollout was still sitting in `FAILED`.

Prod stops at `PENDING_APPROVAL` because `target-prod.yaml` carries `requireApproval: true`. Derive the rollout name rather than typing it:

```
export PROD_ROLLOUT=$(gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --filter="targetId=prod AND state=PENDING_APPROVAL" --format="value(name.basename())" | head -n 1)
echo "[$PROD_ROLLOUT]"

gcloud beta deploy rollouts approve $PROD_ROLLOUT --delivery-pipeline web-app --release web-app-001 --region $REGION --quiet
```

## Multi line command substitution does not survive a paste

A wait loop written like this looks fine and breaks:

```
until [ "$(gcloud beta deploy rollouts list --delivery-pipeline web-app \
  --release web-app-001 --filter='targetId=test' --format='value(state)' | head -n 1)" = "SUCCEEDED" ]; do
```

Pasted into Cloud Shell it produces:

```
-bash: command substitution: line 73: unexpected EOF while looking for matching `)'
```

The `$( ... )` broken across a line continuation gets mangled, the comparison evaluates against an empty string, and the loop **exits immediately printing its success message**. We got `staging rollout SUCCEEDED` echoed while the rollout was `FAILED`.

Either keep the whole `until` on one physical line, or just re-run the list command by hand. For a handful of checks, by hand is better.

## Two more things that cost time

**A second Cloud Shell tab does not inherit `deploy/region`.** Opening a fresh tab and promoting gave:

```
ERROR: Failed to find attribute [region].
```

Every `gcloud beta deploy` command below takes `--region $REGION` explicitly for that reason. Cheaper than remembering which tab was configured.

**Checkpoints score more loosely than the rollouts do.** `Promote the application to staging` went green at 10 of 10 while that rollout was in `FAILED`. Worth clicking each checkpoint before spending time repairing the thing it covers.

## The community script

Three defects worth knowing:

- **Its cluster wait loop does not work.** It pipes the status list into `while read`, which runs in a subshell, so the `all_running=false` set inside is discarded. The outer variable stays `true` and the loop breaks on the first pass while clusters are still provisioning. The twenty attempt retry wrapper it puts around the namespace apply exists to paper over its own bug.
- **`read` prompts** for zone and region when project metadata does not carry them, which eats the next line of the paste. Same failure as ARC114.
- **It calls `kubectx`**, which is not installed in Cloud Shell by default. The lab uses `kubectl config use-context`.
