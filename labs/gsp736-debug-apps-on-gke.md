# GSP736, Debug Apps on Google Kubernetes Engine

Six tasks, **three** checkpoints, twenty minutes stated and about forty in practice. Cloud Shell plus a lot of console.

Checkpoints: **Deploy an application** after task 2, **Create a logs-based metric** after task 4, **Create an alerting policy** after task 5. Task 6, the actual fix, is **unscored** — which is a shame, because it is the point of the lab.

## What this lab actually is

A guided debugging exercise on a deliberately broken microservices app. The bug is real and the diagnosis path is the content: alert fires, logs narrow it to a service, the service turns out to be crash looping, the container logs reveal why, and a one line manifest change fixes it.

Worth doing properly rather than rushing, since nothing here is a lookup.

## Setup

The `central` cluster is prebuilt but starts `PROVISIONING`. Same rule as every GKE lab in these notes: **wait for `RUNNING` before touching it.**

```
until [ "$(gcloud container clusters list --format='value(status)' | sort -u)" = "RUNNING" ]; do gcloud container clusters list --format="csv(name,status)"; sleep 20; done; echo "RUNNING"

gcloud container clusters get-credentials central --zone YOUR_ZONE
kubectl get nodes
```

Then the app:

```
git clone https://github.com/xiangshen-dk/microservices-demo.git
cd microservices-demo
kubectl apply -f release/kubernetes-manifests.yaml
kubectl get pods
```

Twelve pods, all must be `Running` before the checkpoint. The external ip takes a while to be assigned:

```
export EXTERNAL_IP=$(kubectl get service frontend-external | awk 'BEGIN { cnt=0; } { cnt+=1; if (cnt > 1) print $4; }')
echo $EXTERNAL_IP
curl -o /dev/null -s -w "%{http_code}\n" http://$EXTERNAL_IP
```

`200` means ready. An empty `$EXTERNAL_IP` just means the load balancer is still coming up; re-run the export.

## Task 4, the logs-based metric

Logs Explorer, **Show query** enabled:

```
resource.type="k8s_container"
severity=ERROR
labels."k8s-pod/app": "recommendationservice"
```

Actions, Create Metric, name it **`Error_Rate_SLI`**. The name matters, the alerting policy in task 5 looks for it.

**No results is the correct outcome here.** The load has not been generated yet so there are no errors to count. An empty query result looks like a broken filter and is not.

Note the lab prose says "errors from the frontend pod" while the query filters `recommendationservice`. The query is what counts.

## Task 5, the alerting policy

- untick **Active** in the metric picker
- Kubernetes Container, Logs-Based Metric, `logging/user/Error_Rate_SLI`
- Rolling window function: **Rate**
- Threshold: **0.5**
- **Disable** Use notification channel
- Name: `Error Rate SLI`

No notification channel in this lab, which is unusual for the monitoring badge labs and is deliberate; the incident appears in the console instead of an inbox.

## Generating the failure

Locust, via the `loadgenerator-external` endpoint. **300 users, hatch rate 30.** Host is the `frontend-external` url **without the port**.

The alert takes up to five minutes to fire. The Failures tab fills with 500s well before that.

## The diagnosis, which is the actual lesson

The chain, in order:

1. Alert fires on `Error_Rate_SLI`, errors are from **recommendationservice**
2. Errors say it could not connect to a downstream service, so recommendationservice is the victim, not the cause
3. Both frontend and recommendationservice call **productcatalogservice**, which is the common dependency
4. Kubernetes Engine, Workloads, `productcatalogservice` shows the pod **crash looping**
5. Container logs show it parsing the product catalog repeatedly, plus `panic: runtime error: invalid memory address or nil pointer dereference`

Then find it in the source:

```
grep -nri 'successfully parsed product catalog json' src
```

`src/productcatalogservice/server.go:237`. The reload is gated on `reloadCatalog`, which is set from the `ENABLE_RELOAD` environment variable. Confirm from the logs rather than assuming, by adding to the query:

```
jsonPayload.message:"catalog reloading"
```

An `Enable catalog reloading` entry proves the feature is on.

The generalisable bit: **the service that logs the errors is usually not the one that is broken.** Follow the dependency graph to the shared downstream, then check whether that pod is healthy before reading its logs.

## Task 6, the fix

```
grep -A1 -ni ENABLE_RELOAD release/kubernetes-manifests.yaml
```

Prints the line numbers, `373` and `374` in our run.

```
sed -i -e '373,374d' release/kubernetes-manifests.yaml
kubectl apply -f release/kubernetes-manifests.yaml
```

**Run the grep first and use its line numbers.** The `sed` deletes by absolute line number, so if the manifest ever shifts, that command silently deletes the wrong two lines and the apply would push a broken manifest.

Only `productcatalogservice` should report as configured; everything else `unchanged`. Give the pod two to three minutes to stop crashing, then Reset Stats in Locust and watch the failure rate stay at zero.
