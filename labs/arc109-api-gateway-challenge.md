# ARC109, Get Started with API Gateway challenge lab

Three tasks: a gen2 Cloud Run function, an API Gateway in front of it, and a Pub/Sub topic the function publishes to. About twenty five minutes, ten of which is the gateway create.

## Fixed names the scorer checks

Function `gcfunction`, api id `gcfunction-api`, config name `gcfunction-api`, display name `gcfunction API` everywhere, topic `demo-topic`, compute default service account, and the lab region.

## Use helloHttp as the entry point from the start

Task 3 replaces the code with a version that registers `functions.http('helloHttp', ...)`. Deploying task 1 with a different entry point means the task 3 redeploy needs flag changes too. Starting with `helloHttp` makes it a pure code swap.

## Create the topic before the redeploy

The task 3 code calls `pubsub.topic('demo-topic')` at module load, so the topic has to exist first. Create it in the first block, along with a subscription, since the lab wants the default subscription option kept on.

## Task 1

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID

gcloud services enable apigateway.googleapis.com servicemanagement.googleapis.com \
  servicecontrol.googleapis.com cloudfunctions.googleapis.com run.googleapis.com \
  pubsub.googleapis.com

gcloud pubsub topics create demo-topic
gcloud pubsub subscriptions create demo-topic-sub --topic=demo-topic
```

`package.json` needs both the functions framework and the pubsub client, so write the task 3 version of it now and only swap `index.js` later.

First version of `index.js`:

```
const functions = require('@google-cloud/functions-framework');

functions.http('helloHttp', (req, res) => {
  res.status(200).send('Hello World!');
});
```

```
gcloud functions deploy gcfunction \
  --gen2 --runtime=nodejs22 --region=$REGION --source=. \
  --entry-point=helloHttp --trigger-http --allow-unauthenticated --quiet

gcloud functions describe gcfunction --region=$REGION --format="value(serviceConfig.uri)"
```

## Task 2, gateway from the command line

The lab prints a backend address in the form `https://gcfunction-PROJECT_NUMBER.REGION.run.app`. The real deployed url can be the hashed form instead, for example `https://gcfunction-3cbpx2hcca-uk.a.run.app`. Read it from `describe` rather than templating the handout's guess.

```
export FN_URL=$(gcloud functions describe gcfunction --region=$REGION --format="value(serviceConfig.uri)")
export PN=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
```

Write `openapispec.yaml` with `x-google-backend.address` set to `$FN_URL`, then:

```
gcloud api-gateway apis create gcfunction-api --display-name="gcfunction API"

gcloud api-gateway api-configs create gcfunction-api \
  --api=gcfunction-api --openapi-spec=openapispec.yaml \
  --display-name="gcfunction API" \
  --backend-auth-service-account=$PN-compute@developer.gserviceaccount.com

gcloud api-gateway gateways create gcfunction-api \
  --api=gcfunction-api --api-config=gcfunction-api \
  --location=$REGION --display-name="gcfunction API"
```

The gateway create blocks about ten minutes.

## Task 3

Swap `index.js` for the version in the lab that publishes to `demo-topic`, redeploy with the same flags, then call the gateway endpoint:

```
export GATEWAY_URL=$(gcloud api-gateway gateways describe gcfunction-api \
  --location=$REGION --format="value(defaultHostname)")

curl -s -w "\n" https://$GATEWAY_URL/gcfunction
```

Expect `Message sent to Topic demo-topic!`. Messages take about five minutes to show in the subscription.

## Related files

`gsp527-gemini-code-assist-challenge.md` builds the same gateway over a Cloud Function, and the circulating script for it templates the function url instead of reading it back, which is the mistake this file's task 2 section exists to prevent.
