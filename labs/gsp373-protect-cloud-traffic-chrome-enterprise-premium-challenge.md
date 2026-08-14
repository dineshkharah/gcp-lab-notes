# GSP373, Protect Cloud Traffic with Chrome Enterprise Premium Security: Challenge Lab

Four tasks, **four** checkpoints, fifteen minutes stated and **5 credits**. About **twelve minutes**, most of it the first App Engine deploy.

**Tasks 1, 3 and 4 are Cloud Shell. Task 2 is console.** Tasks 3 and 4 are also two clicks each in the console if you prefer, and that is how this run was finished.

Two accounts: an **Owner** and a **Tester**, and unlike ARC119's pair these genuinely differ, because the whole point is that the Tester is denied until a role is granted.

Manual last updated April 2024, **lab last tested October 2023**. The oldest pair in these notes, and it shows: the lab's menu path for task 2 no longer exists.

## The one irreversible command in the lab

`gcloud app create --region=` is **permanent for the life of the project**. No delete, no recreate, no fix. Everything else here can be redone.

Two ways to get it wrong, both covered in `gotchas.md` now:

- **App Engine region names are not Compute region names.** This run's lab said **`europe-west`**, already the App Engine form. Had it said `europe-west1`, `app create` would have rejected it.
- The community script derives the value from `commonInstanceMetadata.items[google-compute-default-region]`, which returns the **Compute** form and is often unset besides.

So guard it, and read the guard's output before continuing:

```
export REGION=europe-west
export TESTER=student-03-90a10716c39e@qwiklabs.net
gcloud config set project $DEVSHELL_PROJECT_ID
export PROJECT=$(gcloud config get-value project)
gcloud services enable iap.googleapis.com appengine.googleapis.com cloudbuild.googleapis.com
gcloud app regions list --format="value(region)" | grep -x "$REGION" && echo "REGION [$REGION] IS VALID FOR APP ENGINE" || echo "STOP: [$REGION] IS NOT AN APP ENGINE REGION"
```

## Task 1, deploy

```
gcloud app create --project=$PROJECT --region=$REGION
git clone https://github.com/GoogleCloudPlatform/python-docs-samples.git
cd python-docs-samples/appengine/standard_python3/hello_world/
gcloud app deploy --quiet
export APP_URL=$(gcloud app describe --format="value(defaultHostname)")
echo "app hostname=[$APP_URL]"
echo "support email=[$(gcloud config get-value core/account)]"
curl -s -o /dev/null -w "HTTP %{http_code}\n" https://$APP_URL
```

`HTTP 200` is task 1 done, and it is also the unprotected state the topics list describes as *"public at first"*.

**The hostname's two letter code is the cheapest proof the region landed right.** `qwiklabs-gcp-03-11942e969a3b.ew.r.appspot.com` for europe-west. A `uc` there would mean us-central, and since the region is permanent that is worth catching immediately rather than after IAP is configured. The community script hardcodes `uc` in a derived `AUTH_DOMAIN` variable, which is wrong everywhere except us-central.

`--quiet` on `app deploy` because it prompts to confirm.

The two `echo` lines produce exactly what the consent screen asks for next.

## Task 2, and the requirement that turned out not to be scored

**The lab's menu path is stale.** It says *"the Security menu of the console"*. It is now **APIs & services > OAuth consent screen**, which redirects to **Google Auth Platform**:

```
https://console.cloud.google.com/auth/overview?project=PROJECT_ID
```

**Get started**, then a four step wizard:

1. **App Information** — app name anything, user support email the Owner account.
2. **Audience** — the lab says External. See below.
3. **Contact Information** — the Owner email again.
4. **Finish** — accept the user data policy, **Create**.

Then stop. *"No scopes and no users"* means do not visit **Data Access**, and leave the test users list under Audience empty.

**Ignore Authorized domains.** It is optional, App Engine domains are handled automatically, and `appspot.com` sits on the public suffix list so the field may reject the value anyway. The lab does not ask for it.

**The finding: `Internal` was selected in step 2 and checkpoint 2 passed anyway.** The task text says *"Configure an external-facing OAuth Consent screen"*, so External is what is specified, but the scorer evidently checks that a brand exists rather than its audience type.

Two consequences worth writing down:

- **External is still the right answer** on a rerun. If checkpoint 2 comes back red, the audience type is the first thing to change.
- More interestingly, it suggests task 2 may be automatable after all. `gcloud iap oauth-brands create --application_title=... --support_email=...` can only produce an **internal** brand for a project inside an organisation, which is why this was written off as console only. If internal satisfies the scorer, that command may satisfy it too. **Untested here**, so treat it as the thing to try first next time rather than as a working step.

## Tasks 3 and 4 from Cloud Shell

```
gcloud iap oauth-brands list
gcloud iap web enable --resource-type=app-engine --project=$PROJECT
gcloud projects add-iam-policy-binding $PROJECT --member=user:$TESTER --role=roles/iap.httpsResourceAccessor
gcloud projects get-iam-policy $PROJECT --flatten="bindings[].members" --filter="bindings.role:roles/iap.httpsResourceAccessor" --format="value(bindings.members)"
```

`oauth-brands list` first, because `web enable` fails without a brand and that error reads like IAP being unavailable rather than the consent screen being absent.

**`roles/iap.httpsResourceAccessor` is the role id behind the console's IAP-secured Web App User label.** The lab gives the display name and the checkpoint reads the id, the usual mismatch.

## Tasks 3 and 4 in the console, which is what was actually used

Faster than the block above, and worth recording because it is four clicks.

**Security > Identity-Aware Proxy**, Applications tab. There is a single **App Engine app** row.

- **Task 3:** flip the **IAP** toggle on that row and confirm the dialog.
- **Task 4:** tick the row's checkbox, open the info panel, **Add principal**, the Tester email, role **Cloud IAP > IAP-secured Web App User**, Save.

The console prints *"Policy updated. It may take a few minutes for these changes to become active"*, which is a real delay and not a warning of failure.

**Granting the role here scopes it to the IAP resource; granting it in the block scopes it to the project.** Either satisfies the checkpoint. Do one, not both.

## The browser verification

Not scored, and the only part of the lab that shows what is being taught.

1. Open the app URL as the **Owner**. Loads, after an OAuth prompt the first time.
2. In a **second incognito window** signed in as the **Tester**, open the same URL. Expect *"You don't have access"*. That is IAP working.
3. After the binding, reload the Tester window. It loads. Give it a minute for the policy to propagate; one failed reload is not a failure.

Two windows rather than two tabs, since signing in as the Tester in the same profile ends your Owner console session.

## The community script

It automates task 1 and prints two console links, so tasks 2, 3 and 4 are untouched. Five problems, the first two serious:

- **`REGION` from Compute metadata**, feeding the one irreversible command in the lab.
- **`AUTH_DOMAIN` hardcodes `uc.r.appspot.com`**, wrong outside us-central.
- `details.json` writes the author's channel name in as the app name.
- A `read -p` subscribe prompt inside a script, which per `gotchas.md` is a prompt waiting to eat a pasted line.
- **It ends with a `remove_files` function that `rm`s every file in `~` starting `gsp`, `arc` or `shell`.** Nothing in this lab creates such files. Read the end of a script before running the start of it.

## Related files

- `gsp499-iap-user-authentication.md`, the guided IAP lab, which was rewritten from App Engine onto Cloud Run. GSP373 is still App Engine, so the old App Engine warnings apply here rather than there.
- `gsp190-iam-custom-roles.md` and `gsp647-iam-permissions-with-gcloud.md` for role ids against console labels.
- `gsp322-build-a-secure-network-challenge.md` for the bastion and IAP tunnel form of restricting access.
