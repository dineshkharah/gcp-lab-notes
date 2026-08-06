# GSP393, Implement CI/CD Pipelines on Google Cloud challenge lab

Seven tasks. Twenty five minutes stated, closer to forty five. All Cloud Shell.

This is `gsp1079-continuous-delivery-cloud-deploy.md` with **two** clusters instead of three, plus two things that lab does not have: a v2 rebuild and a rollback. Read that file first; everything below assumes it.

## Differences from GSP1079

| | GSP1079 | GSP393 |
| --- | --- | --- |
| clusters | `test`, `staging`, `prod` | `cd-staging`, `cd-production` |
| artifact repo | `web-app` | `cicd-challenge` |
| pipeline stages | three, unmodified template | two, template edited with sed |
| extra iam | none | `clouddeploy.jobRunner` and `container.developer` on the compute default sa |
| extra tasks | none | modify `app.go`, release `web-app-002`, roll back |

The pipeline and target templates ship with `staging` / `prod` names and have to be sed'd to `cd-staging` / `cd-production`. Check the results:

```
cat clouddeploy-config/delivery-pipeline.yaml
```

Exactly two stages, `cd-staging` then `cd-production`, no `test` line.

## The same gate as GSP1079, and it is the whole lab

**Both clusters must read `RUNNING` before `get-credentials`.** Async creation means a cluster that is still provisioning silently gets no kubectl context and no `web-app` namespace, and the failure only surfaces much later as a failed rollout that never mentions namespaces.

One physical line, so it pastes safely:

```
until [ "$(gcloud container clusters list --format='value(status)' | sort -u)" = "RUNNING" ]; do gcloud container clusters list --format="csv(name,status)"; sleep 20; done; echo "BOTH RUNNING"
```

Provisioning took about six minutes in our run. Then the second gate: **two `namespace/web-app created` lines with no errors.**

## Single line until loops are the way to batch this lab

A wait split across lines breaks on paste (see `gotchas.md`). Written on one line it works, which means the whole lab can be run as four large pastes rather than a dozen small ones, with the waits built in.

The pattern used throughout:

```
until [ "$(gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --filter='targetId=cd-staging' --format='value(state)' | head -n 1)" = "SUCCEEDED" ]; do gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --format="table(name.basename(),targetId,state)"; sleep 20; done; echo "STAGING SUCCEEDED"
```

Every `gcloud beta deploy` call carries `--region $REGION` explicitly, because a second Cloud Shell tab does not inherit `deploy/region`.

## Task 6, the change the scripts skip

The lab says to edit line 24 of `web/leeroy-app/app.go` to read `leeroooooy app v2!!`. Patch by **pattern, not line number**, so the tab indentation survives:

```
sed -i 's/leeroooooy app!!/leeroooooy app v2!!/' web/leeroy-app/app.go
sed -n '20,28p' web/leeroy-app/app.go
```

Original is `fmt.Fprintf(w, "leeroooooy app!!\n")`. Note there is **no `v1`** in the original, so any sed written as `s/v1/v2/` silently does nothing.

Then rebuild and cut `web-app-002`. Skaffold reuses the cached `leeroy-web` and rebuilds only `leeroy-app`, tagging it **`c3cae80-dirty`** because the working tree is now modified. That suffix turns out to be the most useful thing in the lab.

## The rollback, and how to prove it worked

```
gcloud deploy targets rollback cd-staging --delivery-pipeline=web-app --region=$REGION --quiet
```

`targets rollback` creates a **new rollout under release `web-app-001`**, not a new release. So waiting on "release 001 staging succeeded" matches the original rollout immediately and returns instantly. Count instead:

```
until [ "$(gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --filter='targetId=cd-staging AND state=SUCCEEDED' --format='value(name)' | wc -l)" -ge 2 ]; do gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --format="table(name.basename(),targetId,state)"; sleep 20; done; echo "ROLLBACK SUCCEEDED"
```

Then the proof:

```
kubectl config use-context cd-staging
kubectl get deployment leeroy-app -n web-app -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Ends in **`:c3cae80`** after a successful rollback, and `:c3cae80-dirty` if it did not take. Clean tag is v1, dirty tag is v2.

## The community script

Four problems, one of them substantive:

- **It never modifies `app.go`.** It re-clones the repo and rebuilds identical code, so `web-app-002` is byte identical to `web-app-001` and the rollback rolls back to something indistinguishable. Task 6 is the point of the second half of this lab and the script skips it.
- Its second `git clone` into an existing directory fails, so the "reset to tutorial base" section does nothing.
- One sed writes to `clouddeploy-config/felivery-pipeline.yaml`, a typo. Harmless only because the whole block is redone afterwards.
- `read` prompts for zone and region, which eat the next line of a paste.

It also hardcodes the approval target as `web-app-001-to-cd-production-0001`. Correct in our run, but derive it:

```
export PROD_ROLLOUT=$(gcloud beta deploy rollouts list --delivery-pipeline web-app --release web-app-001 --region $REGION --filter="targetId=cd-production AND state=PENDING_APPROVAL" --format="value(name.basename())" | head -n 1); echo "[$PROD_ROLLOUT]"
```
