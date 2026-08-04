# GSP499, User Authentication: Identity-Aware Proxy

Three tasks, four scored checkpoints. Thirty minutes and it fits, but most of the time is spent watching Cloud Build. Region `us-central1` throughout.

The shape: deploy a trivial Flask app, put IAP in front of it, redeploy it so it reads the identity headers IAP injects, then redeploy again so it cryptographically verifies those headers instead of trusting them.

## This lab is Cloud Run now, not App Engine

Older write-ups of GSP499 describe `gcloud app deploy`, an App Engine region that is permanent once chosen, and an OAuth consent screen that has to be configured before IAP will turn on. **None of that applies.** The current version (manual updated May 2026) is three `gcloud run deploy` calls and a console toggle, with no consent screen step and no irreversible region choice.

Worth checking the manual date before trusting any notes on this one, including these.

## Three source deploys are the runtime

`gcloud run deploy --source .` builds a container with Cloud Build each time, three to five minutes apiece, and there is nothing to overlap them with. Roughly fifteen minutes of the half hour is waiting.

Enabling the APIs first removes one interactive prompt from the first deploy:

```
gcloud config set run/region us-central1
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
```

The **Create an Artifact Registry repository?** prompt can still appear on the first deploy. Answer Y.

## Task 1

```
cd ~
gcloud storage cp gs://PROJECT_ID-bucket/user-authentication-with-iap.zip .
unzip -o user-authentication-with-iap.zip
cd ~/user-authentication-with-iap/1-HelloWorld

gcloud run deploy user-auth-lab --source . --allow-unauthenticated --region=us-central1

gcloud run services describe user-auth-lab --region=us-central1 --format='value(status.url)'
```

The zip lives in a bucket named after the project id. Three subfolders, `1-HelloWorld`, `2-HelloUser`, `3-HelloVerifiedUser`, one per task.

## Enabling IAP is console only

Security, Identity-Aware Proxy, Enable API, Go to Identity Aware Proxy, toggle IAP on for `user-auth-lab`, Turn On.

Then the part that is easy to misread as broken: visiting the app now gives an **access denied** page after signing in. That is the correct intermediate state. IAP is protecting the app and no principal has been authorized yet.

Tick the checkbox next to the service to open the right sidebar, Add Principal, enter the student address, role **Cloud IAP, IAP-Secured Web App User**, Save.

The grant takes about a minute to take effect. If it is still denied after that, the browser is holding a stale IAP cookie:

```
YOUR_SERVICE_URL/_gcp_iap/clear_login_cookie
```

then Use another account and sign in again.

## Task 2, and the spoofing demo

```
cd ~/user-authentication-with-iap/2-HelloUser
gcloud run deploy user-auth-lab --source . --region=us-central1
```

No `--allow-unauthenticated` on this one, and none is needed since the setting carries over from the existing service.

The app now reads two headers IAP injects:

```
X-Goog-Authenticated-User-Email
X-Goog-Authenticated-User-ID
```

Both arrive prefixed with `accounts.google.com:`. The lab then has you **turn IAP off** and curl the app with a forged header:

```
curl -X GET YOUR_URL -H "X-Goog-Authenticated-User-Email: totally fake email"
```

The page renders the fake address, because with IAP bypassed nothing stops a client setting the header itself. That is the entire motivation for task 3.

Note this leaves IAP **off** going into task 3, which matters below.

## Task 3, the audience value is a resource path

The audience is not an OAuth client id, despite the console calling it a Client ID and the lab text calling it `YOUR_CLIENT_ID`. For a Cloud Run service it looks like:

```
/projects/PROJECT_NUMBER/locations/us-central1/services/user-auth-lab
```

Leading slash included. Quote it in the command so the slashes survive.

Get it from the Identity-Aware Proxy page: scroll the resources table **right** to reveal the Actions column, three dots on the `user-auth-lab` row, **Get JWT audience code**.

```
cd ~/user-authentication-with-iap/3-HelloVerifiedUser

gcloud run deploy user-auth-lab --source . \
  --set-env-vars IAP_AUDIENCE="/projects/PROJECT_NUMBER/locations/us-central1/services/user-auth-lab" \
  --region=us-central1
```

The checkpoint scores on the deploy. Everything after it is verification.

## The IAP service agent needs run.invoker

Turn IAP back on, then:

```
gcloud run services add-iam-policy-binding user-auth-lab \
    --member="serviceAccount:service-$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')@gcp-sa-iap.iam.gserviceaccount.com" \
    --role="roles/run.invoker" \
    --region=us-central1
```

Same family as the Pub/Sub and Eventarc service agent problem in `gotchas.md`: a Google managed agent needs an explicit binding before it can reach your resource. Here the agent does exist, because enabling IAP created it, so no `gcloud beta services identity create` is needed. Only the role is missing.

Without this the page shows `None` for both values even with IAP on, since IAP cannot invoke the service.

## How to tell verification actually worked

The verified email arrives **without** the `accounts.google.com:` prefix, because it comes out of the decoded JWT (`X-Goog-IAP-JWT-Assertion`) rather than the plain header. The page shows both side by side, so the missing prefix is the tell.

The signed object supplies the email in `email` and the persistent id in `sub`.
