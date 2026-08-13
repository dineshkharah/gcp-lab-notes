# GSP328, Develop Serverless Applications on Cloud Run: Challenge Lab

Seven tasks, **seven** checkpoints, twenty minutes stated and about **thirteen**. **Entirely Cloud Shell**, two pastes.

Five Cloud Build runs are most of the clock. Nothing is manual; opening the production frontend in a browser at the end is a look, not a step.

Manual last updated December 2025, **lab last tested November 2023**. The staleness shows up in the lab's own snippets rather than in the products.

## Every name is randomised with a suffix, and the suffixes are the point

`public-billing-service-374`, `frontend-staging-service-601`, `private-billing-service-106`, `billing-service-sa-901`, `billing-prod-service-789`, `frontend-service-sa-975`, `frontend-prod-service-741` in this run, all different next time.

On the **unstarted** page every one of these is prose: *"Public billing service"*, *"Private billing service"*, *"Billing service account"*. Names with spaces, which Cloud Run and iam both reject, so anyone planning off the unstarted page has to invent kebab case ids and will get all seven wrong. **Do not prepare this lab from the pre start text.** The started page replaces the prose with real ids.

Two columns are easy to misread even with the real values:

- **"Service Name" in tasks 4 and 6 is not a Cloud Run service.** Task 4 lists service account `billing-service-sa-901` and Service Name `billing-service`; the second is a label, and the Cloud Run service that account gets attached to is `billing-prod-service-789` over in task 5. Same in task 6.
- **The repository is `gcr.io`** in three tables, which is the tag prefix rather than anything to create.

## Two contradictions in the lab text, one of which fakes a pass

**Task 5's verification snippet describes task 3's service.**

```
PROD_BILLING_URL=$(gcloud run services describe private-billing-service-106 ...)
```

Named for production, pointing at staging. It runs, returns `200` with a token, and tells you nothing about the service you just deployed. Only the mismatched suffixes, `-106` against `-789`, make it visible. Corrected form:

```
export PROD_BILLING_URL=$(gcloud run services describe billing-prod-service-789 --platform=managed --region=$REGION --format="value(status.url)")
```

There is a `gotchas.md` section on this now, because a verification that passes against the wrong object is worse than none.

**Task 7's table and its assessment list disagree about authentication.** Table says `unauthenticated`, assessment bullet says "Enable Authentication". The table is right: the production frontend is the public entry point and the architecture has no other way in. The bullet list is boilerplate duplicated from task 5, where authentication genuinely is required.

## Two pastes, and the only ordering constraint

**Task 3 deletes the service checkpoint 1 scores.** That is the entire dependency graph. Everything else is additive and checkpoints latch, so seven tasks collapse into two blocks: tasks 1 and 2, then tasks 3 to 7.

The lab notes that *"Activity Tracking can take some time to register. Wait 30 seconds before retrying"*, so let checkpoints 1 and 2 go green before pasting the delete.

### Block A, tasks 1 and 2

```
export REGION=us-east1
gcloud config set project $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')
export PROJECT=$(gcloud config get-value project)
gcloud config set run/region $REGION
gcloud config set run/platform managed
gcloud services enable run.googleapis.com cloudbuild.googleapis.com containerregistry.googleapis.com
git clone https://github.com/rosera/pet-theory.git
cd pet-theory/lab07

gcloud builds submit --tag gcr.io/$PROJECT/billing-staging-api:0.1 unit-api-billing
gcloud run deploy public-billing-service-374 --image gcr.io/$PROJECT/billing-staging-api:0.1 --region=$REGION --platform=managed --allow-unauthenticated
curl -s -w "\nHTTP %{http_code}\n" $(gcloud run services describe public-billing-service-374 --platform=managed --region=$REGION --format="value(status.url)")

gcloud builds submit --tag gcr.io/$PROJECT/frontend-staging:0.1 staging-frontend-billing
gcloud run deploy frontend-staging-service-601 --image gcr.io/$PROJECT/frontend-staging:0.1 --region=$REGION --platform=managed --allow-unauthenticated
curl -s -w "\nHTTP %{http_code}\n" $(gcloud run services describe frontend-staging-service-601 --platform=managed --region=$REGION --format="value(status.url)")
```

### Block B, tasks 3 to 7

```
gcloud run services delete public-billing-service-374 --region=$REGION --platform=managed --quiet

gcloud builds submit --tag gcr.io/$PROJECT/billing-staging-api:0.2 staging-api-billing
gcloud run deploy private-billing-service-106 --image gcr.io/$PROJECT/billing-staging-api:0.2 --region=$REGION --platform=managed --no-allow-unauthenticated
export BILLING_URL=$(gcloud run services describe private-billing-service-106 --platform=managed --region=$REGION --format="value(status.url)")
curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Bearer $(gcloud auth print-identity-token)" $BILLING_URL

gcloud iam service-accounts create billing-service-sa-901 --display-name="Billing Service Cloud Run"

gcloud builds submit --tag gcr.io/$PROJECT/billing-prod-api:0.1 prod-api-billing
gcloud run deploy billing-prod-service-789 --image gcr.io/$PROJECT/billing-prod-api:0.1 --region=$REGION --platform=managed --no-allow-unauthenticated --service-account=billing-service-sa-901@$PROJECT.iam.gserviceaccount.com
export PROD_BILLING_URL=$(gcloud run services describe billing-prod-service-789 --platform=managed --region=$REGION --format="value(status.url)")
curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Bearer $(gcloud auth print-identity-token)" $PROD_BILLING_URL

gcloud iam service-accounts create frontend-service-sa-975 --display-name="Billing Service Cloud Run Invoker"
gcloud run services add-iam-policy-binding billing-prod-service-789 --region=$REGION --platform=managed --member=serviceAccount:frontend-service-sa-975@$PROJECT.iam.gserviceaccount.com --role=roles/run.invoker
gcloud run services get-iam-policy billing-prod-service-789 --region=$REGION --platform=managed

gcloud builds submit --tag gcr.io/$PROJECT/frontend-prod:0.1 prod-frontend-billing
gcloud run deploy frontend-prod-service-741 --image gcr.io/$PROJECT/frontend-prod:0.1 --region=$REGION --platform=managed --allow-unauthenticated --service-account=frontend-service-sa-975@$PROJECT.iam.gserviceaccount.com
export PROD_FRONTEND_URL=$(gcloud run services describe frontend-prod-service-741 --platform=managed --region=$REGION --format="value(status.url)")
echo $PROD_FRONTEND_URL
curl -s -w "\nHTTP %{http_code}\n" $PROD_FRONTEND_URL

gcloud run services list --platform=managed --region=$REGION --format="table(metadata.name,spec.template.spec.serviceAccountName,status.url)"
```

Three deliberate orderings inside block B:

- **The `builds submit` between creating `billing-service-sa-901` and referencing it** doubles as propagation time. A brand new service account can be rejected as nonexistent by a deploy that names it immediately, and the ninety second build covers the gap without a `sleep`.
- **The invoker binding goes on the billing service**, not the frontend and not the project. `roles/run.invoker` is permission to call one specific service, and "Bind Account to Service" in task 6's assessment is that binding. The `get-iam-policy` immediately after is the standing habit from `gotchas.md` of proving an iam binding landed.
- **`--quiet` appears once**, on the delete, where the only possible answer is yes. See below for why it must not appear on the deploys.

The closing `services list` is the whole lab in one table: four services, `billing-service-sa-901@` on the production billing service, `frontend-service-sa-975@` on the production frontend, blank for the two staging services.

## The community script fails three checkpoints on an absent flag

Worth recording because the mistake is invisible and the script reports success.

Every deploy in it reads `gcloud run deploy $NAME --image ... --quiet` with **no authentication flag at all**. `gcloud run deploy` asks *"Allow unauthenticated invocations?"* when the flag is missing, and `--quiet` answers **no**. So tasks 1, 2 and 7, all three of which require unauthenticated, come out private, return 403 to their endpoint tests, and the script prints a green tick for each.

Four other things absent from it: the task 3 delete, `--service-account` on any deploy despite creating both accounts, the `run.invoker` binding, and any `curl`. That is the whole of requirement 3 of the lab's three stated definition of done items, the secure access one. Both display names also have the word "Account" inserted, giving `Billing Service Account Cloud Run` where the lab says `Billing Service Cloud Run`. And `REGION` is read from `commonInstanceMetadata.items[google-compute-default-region]`, often unset in these projects, which combined with `--quiet` means the deploys fail with no prompt to catch it.

Optimistically 3 of 7. The four it misses are the ones the challenge exists to test.

## Why the endpoint tests are in the blocks

Three of the seven assessments end with *"Service should respond when the endpoint is accessed"*, which is a request rather than a state. Per the action versus end state note in `gotchas.md`, a scorer phrased that way may be reading that the request happened. Each `curl` costs a second, so there is no reason to find out the hard way.

A `403` from either authenticated service **without** the bearer header is correct, not a failure; that is what `--no-allow-unauthenticated` is for. With the header both return `200`.

## Related files

- `gsp081-cloud-run-functions-console.md` and `gsp092-monitoring-and-logging-for-cloud-run-functions.md` for Cloud Run functions rather than services.
- `gsp304-docker-image-to-kubernetes-challenge.md` for the same build and tag and push cycle against `gcr.io`, deployed to GKE instead.
- `gsp1131-artifact-registry-qwik-start.md` for the Artifact Registry form of the registry half.
- `gsp647-iam-permissions-with-gcloud.md` and `gsp190-iam-custom-roles.md` for service accounts and role bindings on their own.
