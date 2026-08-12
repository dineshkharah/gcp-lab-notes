# GSP527, Kickstarting Application Development with Gemini Code Assist: Challenge Lab

Five tasks, **four** checkpoints, one hour stated and **5 credits**, one of the few paid labs in these notes. Task 1 has no checkpoint. Lab access was 2h14m, so there is no time pressure.

Effectively all Cloud Shell. **Nothing scores whether Gemini wrote the code** — the four checkpoints score that the test exists and runs, the backend endpoint returns the right data, the Cloud Function is deployed, and the gateway answers. The only editor step is selecting the Gemini Code Assist project in task 1, which is unscored.

Manual last updated May 2026, lab last tested November 2025.

## The data model, which the lab never states

Everything hinges on this and you have to read it out of the code:

- Firestore collection **`inventory`**
- out of stock means **`quantity == 0`**
- the two out of stock products are **`Wasabi Party Mix`** and **`Jalapeno Seasoning`**

Those two are exactly the **2 items** task 2's test is told to expect. They are created by `initFirestoreCollection()`, which is called from inside the **`/newproducts`** handler rather than at startup. So the count the test depends on is a side effect of a different endpoint having been hit at some point.

Worth verifying rather than assuming, because an unseeded `inventory` makes `/outofstock` correctly return `[]` and the test fail against working code. `gcloud firestore` cannot query documents, so go at the REST API:

```
curl -s -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://firestore.googleapis.com/v1/projects/$PROJECT_ID/databases/(default)/documents:runQuery" \
  -d '{"structuredQuery":{"from":[{"collectionId":"inventory"}],"where":{"fieldFilter":{"field":{"fieldPath":"quantity"},"op":"EQUAL","value":{"integerValue":"0"}}},"select":{"fields":[{"fieldPath":"name"}]}}}'
```

Two documents back means go straight on. An empty result means seed first, by calling `/newproducts`.

## A failing test at task 2 is the expected outcome

The lab frames this as TDD and the ordering follows: task 2 writes a test for `/outofstock`, task 3 implements the endpoint. So `npm run test` at the end of task 2 runs against an endpoint that does not exist yet and **fails**.

That is the point rather than a problem, and checkpoint 2 passes anyway, so do not spend time chasing it. The same test passing is what task 3 is verified by.

## The community script, and what to change in it

Two guides circulate for this lab under different branding and they are byte identical in the code, down to the same `yourproject` placeholder. Four things worth changing:

**The OpenAPI spec templates a gen1 function URL.** It hardcodes `host: us-central1-yourproject.cloudfunctions.net` and the same address under `x-google-backend`. `gcloud functions deploy` has defaulted to gen2 for a while, and gen2 serves from a `run.app` host. It worked in this run with the project substituted, but the assumption is unnecessary since the value can be read back:

```
export FN_URL=$(gcloud functions describe outofstock --region=$REGION --format="value(serviceConfig.uri)")
echo $FN_URL
```

This is the trap already recorded in `arc109-api-gateway-challenge.md`, where the documented `https://NAME-PROJECT_NUMBER.REGION.run.app` shape did not match the real hashed URL. A gateway built on a wrong address deploys cleanly and 404s, ten minutes later.

**`us-central1` is hardcoded in four places**: the function deploy, `host`, `x-google-backend.address`, and both gateway commands.

**It overwrites `functions/index.js` wholesale**, carrying along the `/newproducts` handler and the whole seeding helper. Appending only the `outofstock` handler leaves less to go wrong. Note also that the lab text says `functions/index.ts` while the script writes `index.js`; check which the downloaded folder actually contains.

**`mkdir gateway` runs from inside `functions/`**, because the previous step left you there, so the folder lands at `functions/gateway`. Harmless, but not what the lab describes.

## The names are fixed, not per run

Unusually for a challenge lab, task 5 hands you the identifiers rather than randomising them:

```
export CONFIG_ID=outofstock-api-config
export API_ID=outofstock-api
export GATEWAY_ID=store
export OPENAPI_SPEC=outofstock.yaml
```

The route is `/outofstock` and the function name and entry point are both `outofstock`. Only the project and region change per run.

## Timing

The gateway create is the whole cost. From `arc109-api-gateway-challenge.md` it blocks about **ten minutes**, which is more than every other step put together.

| Task | Time |
|---|---|
| 1 setup and download | 2 min |
| 2 test, mostly `npm install` | 3 to 5 min |
| 3 backend endpoint | 3 min |
| 4 Cloud Function deploy | 3 to 5 min |
| 5 API Gateway | 12 to 15 min |

## Related files

- `arc109-api-gateway-challenge.md`, the same gateway over a Cloud Function pattern, with the URL derivation and the ten minute create.
- `gsp872-api-gateway-qwik-start.md`, the guided version of the gateway half.
- `gsp081-cloud-run-functions-console.md` and `gsp092-monitoring-and-logging-for-cloud-run-functions.md` for the function deployment side.
