# GSP363, Develop and Secure APIs with Apigee X: Challenge Lab

Five tasks, **five** checkpoints, twenty five minutes stated and **5 credits**. The stated time is not real by any route.

**The lab is written to be done entirely in the Apigee proxy editor. It does not have to be.** Apigee has an import API that accepts a proxy bundle as a zip, so the whole of tasks 1, 2, 4 and 5 can be delivered from Cloud Shell in one call. That is the difference between roughly ninety minutes of hand written XML and roughly twenty five minutes of waiting.

Manual last updated and last tested March 2026.

## What each route costs

The console route, which is what the lab describes:

| Task | Realistic | What eats it |
|---|---|---|
| 1 | 15 min | service account, iam, proxy wizard, the GoogleAccessToken XML, first deploy, ssh and test |
| 2 | 30 to 40 min | property set, two conditional flows, three AssignMessage policies, one JavaScript policy |
| 3 | 15 min | api product with operation quota, developer, app, two preflow policies |
| 4 | 8 min | one MessageLogging policy, plus the log delay |
| 5 | 8 min | FaultRules in the target endpoint, plus one AssignMessage |

Plus a deploy after every task at one to three minutes each, and the lab's own warning that you may get 502s for a few minutes after deploying.

The bundle route is **Apigee runtime provisioning, fifteen to twenty minutes, and then about five minutes of api calls.** The provisioning wait dominates either way, which is why the first block should be pasted before you read anything.

## Cloud Shell is only three things in the lab's own text

Worth noticing, because it sets expectations wrongly. The lab uses the shell for the `serviceAccountTokenCreator` binding, the runtime readiness loop, and `gcloud compute ssh apigeex-test-vm`. Everything else it describes as clicking.

**The tests genuinely must run from `apigeex-test-vm`.** `eval.example.com` resolves only on the internal network, so a `curl` from Cloud Shell fails with a dns error that looks like the proxy is broken.

## Block 1, iam and the provisioning wait

Paste this first and read the rest of the lab while it runs.

```
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud services enable translate.googleapis.com apigee.googleapis.com
gcloud iam service-accounts create apigee-proxy --display-name="Apigee Proxy Service Access" 2>/dev/null || echo "service account already exists"
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:apigee-proxy@$PROJECT_ID.iam.gserviceaccount.com" --role="roles/logging.logWriter" --quiet
gcloud iam service-accounts add-iam-policy-binding apigee-proxy@$PROJECT_ID.iam.gserviceaccount.com --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-apigee.iam.gserviceaccount.com" --role="roles/iam.serviceAccountTokenCreator"
gcloud iam service-accounts get-iam-policy apigee-proxy@$PROJECT_ID.iam.gserviceaccount.com --format="yaml(bindings)"
for i in $(seq 1 120); do STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/instances/eval-instance" | jq -r '.state // "PENDING"'); [ "$STATE" = "ACTIVE" ] && echo "INSTANCE ACTIVE" && break; echo "instance state=$STATE ($i)"; sleep 15; done
for i in $(seq 1 60); do ATT=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/instances/eval-instance/attachments" | jq -r '.attachments[]?.environment' | grep -x eval); [ "$ATT" = "eval" ] && echo "ENV ATTACHED, ORG READY" && break; echo "waiting for eval attachment ($i)"; sleep 15; done
```

Both waits are **bounded**, unlike the lab's `while : ;` version and unlike both circulating scripts. Per the unbounded wait note in `gotchas.md`, a `while` loop over a resource that failed to appear prints reassuring dots forever.

**On the `serviceAccountTokenCreator` binding.** The lab gives it to you as an explicit copy button, and **both circulating scripts omit it**, which is why it is in the block above. I expected its absence to fail task 1 outright, since the `GoogleAccessToken` element depends on the Apigee service agent being able to mint tokens for `apigee-proxy`. In practice the lab completed without that command being run separately, so either the lab project pre grants it or the checkpoint does not exercise a live backend call. Treat it as the **first thing to check if task 1 is red**, not as a guaranteed blocker.

## Block 2, get the bundle and prove it is one

```
curl -L "<bundle-url>" -o translate-v1.zip
file translate-v1.zip
unzip -o -q translate-v1.zip -d bundle && find bundle -type f | sort
```

Two reasons this is its own block rather than folded into the import.

**`file` must say Zip archive.** A `drive.google.com/uc?export=download` url often returns an html interstitial instead of the file, and the circulating script pipes that straight into the import endpoint as `application/octet-stream`. The failure is a confusing import error rather than a download error.

**The `find` listing is the only way to check the bundle before trusting it.** The checkpoints inspect the revision's actual xml against exact names:

`AM-BuildTranslateRequest`, `AM-BuildTranslateResponse`, `AM-BuildLanguagesRequest`, `JS-BuildLanguagesResponse`, `VA-VerifyKey`, `Q-EnforceQuota`, `ML-LogTranslation`, `AM-BuildErrorResponse`, plus the `translate` and `getLanguages` flow names, and a `language.properties` set with `output=es` and `caller=en`.

Per the community scripts note in `gotchas.md`, these bundles circulate between channels without anyone rechecking the literals against the current manual. A bundle built for an earlier variant imports cleanly and fails the checkpoints.

**The url is a third party Drive link and will eventually die.** The console route below is the fallback, and it is the reason the task table above is worth keeping.

## Block 3, import, deploy, and the distribution objects

```
export REV=$(curl -s -X POST "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/apis?action=import&name=translate-v1" -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/octet-stream" --data-binary @translate-v1.zip | jq -r '.revision')
echo "imported revision=[$REV]"
curl -X POST "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/environments/eval/apis/translate-v1/revisions/$REV/deployments?serviceAccount=apigee-proxy@$PROJECT_ID.iam.gserviceaccount.com" -H "Authorization: Bearer $(gcloud auth print-access-token)"

cat > translate-product.json <<'EOF'
{
  "name": "translate-product",
  "displayName": "translate-product",
  "approvalType": "auto",
  "attributes": [{"name": "access", "value": "public"}],
  "environments": ["eval"],
  "operationGroup": {
    "operationConfigs": [
      {
        "apiSource": "translate-v1",
        "operations": [{"resource": "/", "methods": ["GET", "POST"]}],
        "quota": {"limit": "10", "interval": "1", "timeUnit": "minute"}
      }
    ],
    "operationConfigType": "proxy"
  }
}
EOF
curl -X POST "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/apiproducts" -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" -d @translate-product.json
curl -X POST "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/developers" -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" -d '{"email":"joe@example.com","firstName":"Joe","lastName":"Developer","userName":"joe"}'
curl -X POST "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/developers/joe@example.com/apps" -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" -d '{"name":"translate-app","apiProducts":["translate-product"]}'
export KEY=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/developers/joe@example.com/apps/translate-app" | jq -r '.credentials[0].consumerKey')
echo "api key=[$KEY]"
```

Four things here that the circulating scripts get wrong:

- **The revision is captured from the import response** rather than hardcoded to `revisions/1`. A failed import then produces one visible error instead of a deploy against a revision that does not exist.
- **`-s` is off** on the import, deploy, product, developer and app calls. Silence on five api calls is how a script reports success it did not have.
- **The api product is created after the proxy is deployed.** One of the two scripts creates it first, with `"apiSource": "translate-v1"` pointing at a proxy that does not yet exist.
- **The consumer key is extracted.** Both scripts create the app and discard the response, so `$KEY` never exists and no test in tasks 3, 4 or 5 can run. The key is at `.credentials[0].consumerKey`.

The service account goes on the **deployment**, as a query parameter, not in the bundle.

## Block 4, the four tests that prove four tasks

Give the deploy two to three minutes first, per the lab's 502 warning.

```
export TEST_VM_ZONE=$(gcloud compute instances list --filter="name=('apigeex-test-vm')" --format="value(zone)")
gcloud compute ssh apigeex-test-vm --zone=$TEST_VM_ZONE --quiet --force-key-file-overwrite --command="curl -s -k -X GET 'https://eval.example.com/translate/v1/languages' -H 'Key: $KEY' | head -c 200; echo"
gcloud compute ssh apigeex-test-vm --zone=$TEST_VM_ZONE --quiet --command="curl -s -k -X POST 'https://eval.example.com/translate/v1?lang=de' -H 'Content-Type:application/json' -H 'Key: $KEY' -d '{\"text\": \"Hello world!\"}'; echo"
gcloud compute ssh apigeex-test-vm --zone=$TEST_VM_ZONE --quiet --command="curl -s -k -X POST 'https://eval.example.com/translate/v1?lang=invalid' -H 'Content-Type:application/json' -H 'Key: $KEY' -d '{\"text\": \"Hello world!\"}'; echo"
gcloud logging read 'logName:"translate"' --limit=5 --format="value(textPayload)"
```

Expected, in order, and each one maps to a checkpoint:

- a json array of languages, which is task 2's `getLanguages` flow
- `{"translated":"Hallo Welt!"}`, task 2's `translate` flow
- the rewritten error naming the `lang` query parameter, task 5
- `de|Hello world!|Hallo Welt!` in the logs, task 4

The key being **required** is task 3, which the lab's own two failing curls demonstrate: no key and a bogus key must both be rejected.

`--quiet` on each ssh call because the first `gcloud compute ssh` of a session generates a key pair and prompts for a passphrase. `--force-key-file-overwrite` on the first one only, as the lab suggests.

## If you take the console route

The single most useful sentence in the lab is in its own Save errors note: when a save fails with `Could not save new revision`, use the **Save dropdown, then Save as new revision**, and the error message tells you what is actually invalid. The plain Save button does not. With this volume of hand written xml you will need that repeatedly.

Two structural details worth having in advance, because they are the ones people get wrong:

- **`getLanguages` takes a path with no verb.** The lab explains why in a parenthesis: the flow rewrites the request verb from GET to POST, so a condition on `request.verb` equal to GET stops matching before the response policies run.
- **Task 2's variable names feed task 4.** `language`, `text` and `translated` are created in `AM-BuildTranslateRequest` and `AM-BuildTranslateResponse`, and `ML-LogTranslation` logs those exact names. Get them wrong in task 2 and task 4 logs empty fields with nothing obviously broken.

`FaultRules` in task 5 goes in the **TargetEndpoint** `default.xml`, not the ProxyEndpoint. The lab says so twice, which suggests it is a common mistake.

## Related files

- `gsp349-deploy-manage-apigee-x-challenge.md`, the other Apigee challenge lab, covering the org, environments, the provisioning wizard and Cloud Armor rather than proxies and policies.
- `gsp872-api-gateway-qwik-start.md` and `arc109-api-gateway-challenge.md` for API Gateway, the much simpler product that solves an overlapping problem.
- `gsp647-iam-permissions-with-gcloud.md` for service accounts and role bindings on their own.
- `gsp092-monitoring-and-logging-for-cloud-run-functions.md` for the Cloud Logging side of what `ML-LogTranslation` writes.
