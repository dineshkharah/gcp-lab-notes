# GSP080, Cloud Run Functions: Qwik Start Command Line

Four tasks plus a quiz, **one** checkpoint, fifteen minutes stated and **no cost**. **Entirely Cloud Shell.** Budget **fifteen**, not five, because the lab's own deploy command does not work as given.

Manual and last tested both August 2026, which did not help.

## The lab's deploy command fails, and the fix is one binding it never mentions

```
ERROR: (gcloud.functions.deploy) ResponseError: status=[403]
Validation failed for trigger .../triggers/nodejs-pubsub-function-129721:
Permission "iam.serviceAccounts.ActAs" denied on
"EndUserCredentials to cloudfunctionsa@PROJECT_ID.iam.gserviceaccount.com"
```

The lab hands you `--service-account cloudfunctionsa@PROJECT_ID.iam.gserviceaccount.com` and **does not grant the student account the role required to use it**. Deploying a function that runs as a service account needs `iam.serviceAccounts.actAs` on that account, and the student has none.

Read the error carefully, because it names two identities and it is easy to fix the wrong one. **`EndUserCredentials` is you**, not the service account. The permission is missing on the caller, so the grant goes to the student address:

```
export ME=$(gcloud config get-value account)
gcloud iam service-accounts add-iam-policy-binding cloudfunctionsa@$PROJECT_ID.iam.gserviceaccount.com --member="user:$ME" --role="roles/iam.serviceAccountUser"
gcloud iam service-accounts get-iam-policy cloudfunctionsa@$PROJECT_ID.iam.gserviceaccount.com --format="table(bindings.role,bindings.members)"
```

Then redeploy with a bounded retry, since a fresh binding is often not usable on the first call:

```
for i in $(seq 1 6); do echo n | gcloud functions deploy nodejs-pubsub-function --gen2 --runtime=nodejs22 --region=$REGION --source=. --entry-point=helloPubSub --trigger-topic cf-demo --stage-bucket $PROJECT_ID-bucket --service-account cloudfunctionsa@$PROJECT_ID.iam.gserviceaccount.com --allow-unauthenticated && echo "DEPLOYED" && break; echo "iam not usable yet ($i), waiting 20s"; sleep 20; done
```

`echo n |` answers the prompt described below, so the loop cannot hang waiting for input.

## The prompt's default answer is the one that errors

Before the trigger validation, the deploy asks:

```
Service account [service-NNN@gcp-sa-pubsub.iam.gserviceaccount.com] is missing
the role [roles/iam.serviceAccountTokenCreator].
Bind the role ... ? (Y/n)?
```

Answering `y` fails immediately:

```
ERROR: status=[400] Service account service-NNN@gcp-sa-pubsub.iam.gserviceaccount.com does not exist.
```

That is the first section of `gotchas.md` playing out. The Pub/Sub **service agent** has not been created in this project yet, so a role cannot be bound to it. Nothing is half applied; the deploy aborts there.

**The prompt is `(Y/n)` with Y capitalised**, so both the intuitive answer and a bare enter take the failing path. The lab does say to select `n` but never says why, and it presents that as a stray note rather than as the thing that decides whether the command runs.

Declining is safe here. The role concerns Pub/Sub **push** authentication, and a gen2 function's Pub/Sub trigger is delivered through Eventarc rather than a push endpoint you own. Same conclusion as GSP363, where a declined `serviceAccountTokenCreator` binding also turned out not to matter.

## The topic name is wrong in the lab's own prose

Task 2 opens with *"you'll set the `--trigger-topic` as `cf_demo`"*, underscore. Every command in the lab uses **`cf-demo`**, hyphen, including the deploy and the publish.

The commands are right. This matters more than a typo normally would, because publishing to the wrong name **succeeds**: `gcloud pubsub topics publish` creates nothing but returns a message id happily if the topic exists, and the underscore version simply has no subscriber. You would get a green publish and no log line, with nothing to indicate why.

```
gcloud pubsub topics list --format="value(name)"
```

`cf-demo` is what should come back.

## The function code needs a quoted heredoc

`index.js` contains:

```
console.log(`Hello, ${name}!`);
```

In an **unquoted** heredoc, bash expands `${name}` to nothing and the deployed function logs `Hello, !` for every message. The file has to go down with `<<'EOF'`, and it is worth proving rather than assuming:

```
grep -n 'Hello, ${name}' index.js
```

The line must come back with `${name}` intact. This replaces the lab's two `nano` sessions, which are the only non command steps in it.

`package.json` has its own version of the problem in reverse: it contains `\"Error: no test specified\"`, and a quoted heredoc preserves those backslashes exactly as JSON needs them. Cheap to confirm before `npm install` produces a confusing error:

```
node -e "JSON.parse(require('fs').readFileSync('package.json'));console.log('package.json parses')"
```

## The npm engine warning is noise

```
npm warn EBADENGINE required: { node: '>=16 <=22' }
npm warn EBADENGINE current: { node: 'v24.19.0', npm: '12.0.2' }
```

Cloud Shell ships Node v24 and `cloudevents` wants 22 or lower. **Irrelevant**, because nothing runs locally: the deployed function runs on `--runtime=nodejs22` server side, and `npm install` here only populates `node_modules` for upload. Same for the three moderate vulnerabilities; do not run `npm audit fix --force`, which would change the dependency the lab pins.

## Check the two referenced resources before deploying

The deploy names a stage bucket and a service account, and a wrong guess about either fails after minutes rather than instantly:

```
gcloud storage ls | grep -- "-bucket" || echo "NO STAGE BUCKET FOUND"
gcloud iam service-accounts list --filter="email:cloudfunctionsa" --format="value(email)"
```

Both are pre created. The bucket is `PROJECT_ID-bucket`.

## The rest

```
gcloud functions describe nodejs-pubsub-function --region=$REGION --format="value(state)"
gcloud pubsub topics publish cf-demo --message="Cloud Function Gen2"
gcloud functions logs read nodejs-pubsub-function --region=$REGION --limit=20
```

`ACTIVE` is the state to see, and the single checkpoint reads the deploy.

**An empty `logs read` is expected**, and the lab says so: logs can take around ten minutes. Nothing is scored on it. Related to the missing log output section in `gotchas.md`, except here the lab is honest about the delay up front.

Quiz answer: **True**.

## Related files

- `gsp081-cloud-run-functions-console.md`, the console twin of this lab.
- `gsp092-monitoring-and-logging-for-cloud-run-functions.md` for the logging side.
- `gsp328-serverless-applications-cloud-run-challenge.md` for `--service-account` and `roles/run.invoker` on Cloud Run services, and `gsp363-develop-secure-apis-apigee-x-challenge.md` for the other declined `serviceAccountTokenCreator` binding.
- `gsp096-pubsub-qwik-start.md` for topics and subscriptions on their own.
