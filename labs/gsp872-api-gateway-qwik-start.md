# GSP872, API Gateway qwik start

Six tasks: deploy a Cloud Function backend, test it, create a gateway, secure it with an api key, deploy a second config that enforces the key, then test with and without the key. About thirty five minutes, with roughly fifteen of that being the two gateway operations.

## Skip the download and re upload dance

The lab has you run `cloudshell download` to pull the yaml to your laptop and then upload it through the Create Gateway form. `gcloud api-gateway` reads the file where it already is, so the whole gateway can be built from the shell.

## Export API_ID once

The lab exports a randomised api id, seds it into the yaml, then in task 3 tells you to run the same export again "to once again obtain the API ID". That generates a different random value than the one now written into your spec. Export it once, echo it, and reuse it.

## Tasks 1 and 2, backend

```
export REGION=YOUR_REGION
export PROJECT_ID=YOUR_PROJECT_ID
gcloud config set compute/region $REGION

gcloud services enable apigateway.googleapis.com servicemanagement.googleapis.com \
  servicecontrol.googleapis.com cloudfunctions.googleapis.com run.googleapis.com

git clone https://github.com/GoogleCloudPlatform/nodejs-docs-samples.git
cd nodejs-docs-samples/functions/helloworld/helloworldGet

gcloud functions deploy helloGET \
  --runtime nodejs20 --trigger-http --allow-unauthenticated --region $REGION

curl -s -w "\n" https://$REGION-$PROJECT_ID.cloudfunctions.net/helloGET
```

If the deploy returns `IamPermissionDeniedException`, run it again. The lab warns about that one.

## Task 3, gateway

```
cd ~
export API_ID="hello-world-$(cat /dev/urandom | tr -dc 'a-z' | fold -w 8 | head -n 1)"
echo "API_ID = $API_ID"
```

Write `openapi2-functions.yaml` with the title set to `$API_ID description` and the backend address pointing at the function url, then:

```
gcloud api-gateway apis create $API_ID --display-name="Hello World API"

gcloud api-gateway api-configs create hello-world-config \
  --api=$API_ID --openapi-spec=openapi2-functions.yaml \
  --display-name="Hello World Config" \
  --backend-auth-service-account=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')-compute@developer.gserviceaccount.com

gcloud api-gateway gateways create hello-gateway \
  --api=$API_ID --api-config=hello-world-config \
  --location=$REGION --display-name="Hello Gateway"
```

Gateway id has to be `hello-gateway` because task 6 describes it by that name.

Test it:

```
export GATEWAY_URL=$(gcloud api-gateway gateways describe hello-gateway --location $REGION --format json | jq -r .defaultHostname)
curl -s -w "\n" https://$GATEWAY_URL/hello
```

## Task 4, api key

Enable key support for the managed service first:

```
MANAGED_SERVICE=$(gcloud api-gateway apis list --format json | jq -r .[0].managedService | cut -d'/' -f6)
gcloud services enable $MANAGED_SERVICE
```

The key itself needs the console, because it has to be scoped to that managed service: APIs and Services, Credentials, Create credentials, API key, then under the api restrictions search `hello-world` and tick it.

## Task 5, second config with key enforcement

Same spec plus a `security` block on the path and a `securityDefinitions` section defining `api_key` as an `apiKey` named `key` in the query.

```
gcloud api-gateway api-configs create hello-config \
  --api=$API_ID --openapi-spec=openapi2-functions2.yaml \
  --display-name="Hello Config" \
  --backend-auth-service-account=$PROJECT_ID@$PROJECT_ID.iam.gserviceaccount.com

gcloud api-gateway gateways update hello-gateway \
  --api=$API_ID --api-config=hello-config --location=$REGION
```

Note the service account changes between the two configs. Task 3 asks for the compute engine default, task 5 asks for the Qwiklabs user service account, which is `PROJECT_ID@PROJECT_ID.iam.gserviceaccount.com`. That is deliberate in the lab, not a typo.

The update took about five to eight minutes, longer than the lab's "a few minutes".

## Task 6

```
curl -sL -w "\n" https://$GATEWAY_URL/hello
curl -sL -w "\n" "https://$GATEWAY_URL/hello?key=$API_KEY"
```

First returns a 401 about unregistered callers, second returns `Hello World!`. If the first still succeeds, the new config has not propagated yet.
