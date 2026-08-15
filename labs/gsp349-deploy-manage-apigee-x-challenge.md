# GSP349, Deploy and Manage Apigee X: Challenge Lab

Five tasks, **five** checkpoints, forty five minutes stated and **5 credits**. About **forty minutes**, and most of that is waiting on Apigee rather than doing anything.

**Half console by mandate, half Cloud Shell.** Tasks 1 and 2 say to use the Apigee UI and the provisioning wizard in so many words. Tasks 3 and 5 say to use the **Apigee API**, meaning `curl` with an access token or the API Explorer. Task 4 is plain `gcloud`.

Manual last updated July 2026, lab last tested September 2025.

## The order that fills the waits

The org provisioning from the wizard's step 3 may still be running when you start, and it is twenty to forty minutes. Nothing else can be hurried, so the order matters more than usual:

1. **Task 1 first**, in the UI. It needs the org but not the instance, so it runs during provisioning.
2. **Task 2** once step 3 shows complete.
3. **Task 3** immediately after, since the NAT address needs the instance.
4. **Task 4**, which needs the load balancer task 2 created.
5. **Task 5** last, and it takes several more minutes on its own.

## Task 1, the two dropdowns that are the actual requirement

Apigee > Admin > Environments > Environments > **Create**.

- Name `staging`
- **Proxy type: Programmable**
- **Deployment type: Proxy**

Those two are called out in the requirements precisely because they default to something else. Everything else on the form is fine as is.

Then **Environment groups > Create**, name `staging-group`, hostname `staging.example.com`, and add `staging` to it.

**Do not attach the group to the instance here.** That is task 5, and the lab says so: *"The staging environment will be attached to the instance in a later task."*

## Task 2, the wizard step with no undo button

Provisioning wizard, step 4, **Access routing**.

- **External** access
- Subnet **api-subnet**
- Wildcard DNS left at **nip.io**

The lab's warning is worth taking literally: *"If you make a mistake, you will need to go Network services > Load balancing to delete the load balancer and refresh the wizard page to try the access routing step again."* There is no edit, only delete and redo.

The wizard writes the `1.2.3.4.nip.io` hostname onto **eval-group** for you. You need that hostname again in task 4, and it is at Management > Environments > Environment Groups.

## Task 3, create and activate are two separate calls

The most common way to get this wrong is treating it as one operation. The address is created in `CREATING`, becomes `RESERVED`, and only then can be **activated** to `ACTIVE`. The task says "create and activate".

Via `curl`:

```
export PROJECT=$(gcloud config get-value project)
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/natAddresses" -d '{"name":"apigee-nat-ip"}' | jq
for i in $(seq 1 20); do STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/natAddresses/apigee-nat-ip" | jq -r '.state // "PENDING"'); [ "$STATE" = "RESERVED" ] && echo "RESERVED" && break; echo "nat address state=$STATE ($i)"; sleep 15; done
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/natAddresses/apigee-nat-ip:activate" | jq
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/natAddresses/apigee-nat-ip" | jq
```

**Or via the API Explorer**, which is what the circulating guide uses and is a legitimate reading of "use the Apigee API". Search the reference for `organizations.instances.natAddresses`, then use the Try this API panel with:

| Call | Field | Value |
|---|---|---|
| `create` | parent | `organizations/PROJECT_ID/instances/eval-instance` |
| `create` | request body | `{"name": "apigee-nat-ip"}` |
| `activate` | name | `organizations/PROJECT_ID/instances/eval-instance/natAddresses/apigee-nat-ip` |

Note the shape difference: **create takes a parent and a body, activate takes the full resource name**. Same two step structure either way.

## Task 4, the rule name and the default rule

**The rule is `rce-stable`**, used as `evaluatePreconfiguredExpr('rce-stable')`. Confirm rather than trust, since the lab tells you to look it up and the newer `rce-v33-stable` naming exists on some builds:

```
gcloud compute security-policies list-preconfigured-expression-sets | grep -i -A2 rce
gcloud compute security-policies create protect-apigee --description="block remote code execution"
gcloud compute security-policies rules create 1000 --security-policy=protect-apigee --expression="evaluatePreconfiguredExpr('rce-stable')" --action=deny-403
gcloud compute backend-services list --global
```

Then attach it to whatever the wizard named the backend service:

```
gcloud compute backend-services update <BACKEND_SERVICE> --security-policy=protect-apigee --global
gcloud compute backend-services describe <BACKEND_SERVICE> --global --format="value(securityPolicy)"
```

**"Allow all traffic except the matching rule" needs no extra work.** A new policy already has a default rule at the lowest priority with action `allow`. Adding an explicit allow would be redundant and could collide on priority.

**You do not need to get a request actually blocked.** The lab says so: *"You do not need to block a request for this progress check to be successful."* Cloud Armor propagation takes minutes, so retrying `curl ...?doc=/bin/ls` before clicking is spent clock.

## Task 5, the attachment, and a wait worth bounding

```
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/attachments" -d '{"environment":"staging"}' | jq
for i in $(seq 1 60); do ATT=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://apigee.googleapis.com/v1/organizations/$PROJECT/instances/eval-instance/attachments" | jq -r '.attachments[]?.environment' | grep -x staging); [ "$ATT" = "staging" ] && echo "STAGING ATTACHED" && break; echo "waiting for staging attachment ($i)"; sleep 15; done
```

The circulating guide's version of this loop is `while : ;` with no bound. Per the unbounded wait note in `gotchas.md`, that spins forever if the attachment call failed, and the dots look identical either way. Five minutes of dots is normal here, so it is exactly the case where an unbounded loop hides a real failure.

## What the community guide is and is not

Unusually honest for one of these: it does not pretend to automate the lab. It gives the two API Explorer paths for task 3, the one string for task 4, and the attach plus wait for task 5, and leaves tasks 1 and 2 to the UI where the lab requires them.

Two things to fix in it:

- **The wait loop is unbounded**, as above.
- **It gives no verification for task 4**, so nothing confirms the policy actually attached to the backend service rather than merely existing. The `describe` line above is the check.

## Related files

- `gsp363-develop-secure-apis-apigee-x-challenge.md`, the other Apigee challenge lab, which comes first in the learning path and is where the proxy editor lives. Also the source of the note that Apigee's api can import a whole proxy bundle.
- `gsp528-ncc-challenge.md` and `gsp322-build-a-secure-network-challenge.md` for the VPC and load balancer side.
- `gsp510-manage-kubernetes-challenge.md` for another lab where the scorer wanted an operation rather than an end state.
