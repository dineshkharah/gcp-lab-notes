# GSP526, Privileged Access with IAM challenge lab

Six tasks, six checkpoints. Twenty minutes and it fits comfortably, because every PAM operation is instant. The challenge lab at the end of the Privileged Access with IAM badge, alongside GSP190, GSP647 and GSP499.

Two personas on one project:

- **Cymbal Systems Admin**, the primary user. Enables PAM, creates and updates the entitlement, requests the grant.
- **Cymbal Security Lead**, the secondary user. Approves the grant, then revokes it.

Both accounts share one password.

## Sign the second user in before starting

Open a **second, separate incognito window** and sign the Security Lead in immediately, while task 1 is running. Everything from task 4 onward alternates between the two windows, and logging in with a grant already pending burns clock for no reason.

Keep the windows strictly separate. Signing the Lead into the primary window replaces the session and the Admin has to log back in.

## Task 1 is where the lab actually bites

The task text says the service agent is "created automatically upon activation". Often it is not there when you go looking for it, which is the same trap as the Pub/Sub and Eventarc agents in `gotchas.md`. Force it into existence and read its address back from the same command:

```
export PROJECT_ID=YOUR_PROJECT_ID
gcloud config set project $PROJECT_ID

gcloud services enable privilegedaccessmanager.googleapis.com

export PAM_SA=$(gcloud beta services identity create \
  --service=privilegedaccessmanager.googleapis.com \
  --project=$PROJECT_ID --format='value(email)')
echo $PAM_SA

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${PAM_SA}" \
  --role="roles/privilegedaccessmanager.serviceAgent"

gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:privilegedaccessmanager" \
  --format="table(bindings.role,bindings.members)"
```

`identity create` both creates the agent and prints its email, which beats constructing `service-NUMBER@gcp-sa-pam.iam.gserviceaccount.com` by hand. The address goes into a variable so a typo cannot make the binding land on a non existent principal, which fails silently.

The `get-iam-policy` at the end is the proof the binding is real. Do not skip it.

This is the only task worth doing in Cloud Shell. PAM's gcloud surface is thin and the rest of the lab is described as wizard fields, so the console is the shorter path from here.

## Task 2, the wizard

Navigation menu, IAM & Admin, Privileged Access Manager. Location defaults to `global`.

| Field | Value |
| --- | --- |
| Entitlement name | `pam-entitlement` |
| Role | Compute Admin |
| Maximum duration | 10 hours |
| Requester principal | the **primary** user |
| Justification required from requesters | Not required |
| Approver principal | the **secondary** user |
| Number of approvers required | 1 |

Three ways to get this wrong:

- **10 hours, not 4.** Task 3 is the change to 4. Entering 4 here loses task 2 and there is no way to score it afterwards without recreating the entitlement.
- **Requester and approver are easy to swap.** Both addresses are `student-02-` followed by a hash and differ only in the hash. Paste them, do not type them, and check which is which against the lab page.
- **Justification: Not required.** The form tends to default the other way, and task 4 has you supply a justification anyway, which makes the wrong setting feel correct.

The task text says to fill the parameters "in the specified order", which is worth honouring: the approver fields do not appear until approvals are switched on.

## Task 3

**Entitlements for all users** tab, not My Entitlements. Edit `pam-entitlement`, maximum duration to 4 hours, save.

The edit walks the same multi step form, so confirm the role is still Compute Admin and the approver count is still 1 after saving.

## Task 4, one window each

Primary window: **My Entitlements** tab, `pam-entitlement`, Request grant, duration **4 hours** (matching the new maximum, higher is rejected), any justification text, submit.

Secondary window: find the pending grant and Approve.

The approver view does not live update. If the Lead sees no pending grant, refresh before concluding the request failed.

Then **wait one to two minutes** before clicking Check my progress. The lab says so in the task text: the scorer reads approval event logs and they lag behind the action. A failure inside the first minute means nothing.

## Task 5 belongs to the Lead

Revoke the active grant **as the secondary user**. That is the dual control point the whole lab is built around, and doing it as the Admin is the likely way to lose the checkpoint.

A grant that sits on `Revoking` settles by itself.

Get this checkpoint green before task 6. Deleting the entitlement removes what tasks 4 and 5 are scored against.

## Task 6

Delete `pam-entitlement` from the **Entitlements for all users** tab, in the primary window.

Either account can do it, since the scorer only checks that the entitlement is gone, but the Admin is the persona doing cleanup and the log review reads more naturally there.

For the audit log half, Logging, Logs Explorer:

```
resource.type="audited_resource"
protoPayload.serviceName="privilegedaccessmanager.googleapis.com"
```

The full lifecycle should be visible: `CreateEntitlement`, `UpdateEntitlement`, `CreateGrant`, `ApproveGrant`, `RevokeGrant`, `DeleteEntitlement`. Widen the time range to the last hour if it comes back empty.
