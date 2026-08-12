# GSP510, Manage Kubernetes in Google Cloud: Challenge Lab

Six tasks, **six** checkpoints, twenty five minutes stated and about thirty five in practice, most of it waiting. **Entirely Cloud Shell**, including task 4, which the lab describes purely as console work.

Scoring is 20/20/20/10/10/20. The badge closer for GSP1026, GSP736, GSP053 and GSP1131.

Manual last updated and last tested March 2026.

## The one thing that actually cost time

**Enabling Managed Prometheus at cluster creation is not enough for the task 2 checkpoint.**

Passing `--enable-managed-prometheus` on `clusters create` is tempting, because it folds task 2's first requirement into task 1 and saves a five minute control plane operation. The cluster comes up correctly by every measure:

```
monitoringConfig:
  managedPrometheusConfig:
    enabled: true
```

and `kubectl get pods -n gmp-system` shows the operator plus one collector per node, all `Running`. The checkpoint still returned **15 of 20** with `Please enable Managed Prometheus on the cluster.`

Re-clicking after every pod was ready did not help. The other three sub-checks were provably fine: `prometheus-test` at `3/3` available, and the PodMonitoring carrying `interval: 45s`, `port: metrics`, the right labels and `ConfigurationCreateSuccess`. Running the command the lab actually asked for took it straight to 20 of 20:

```
gcloud container clusters update $CLUSTER_NAME --zone=$ZONE --enable-managed-prometheus
```

The state was already true before that ran, so **the scorer wanted the update operation, not the resulting configuration.** Do it as its own step and take the extra few minutes.

## Order that the checkpoints require

Two hard constraints, both easy to violate by scripting straight through.

**Task 4 before task 5.** A logs-based metric only counts entries that arrive *after* it exists, and the `InvalidImageName` failure is what produces them. Fix the image first and the metric has nothing to count.

**Checkpoint 5 before task 6.** Task 5 is scored on `helloweb` running `hello-app:1.0`, and task 6 replaces that image with your `:v2` build. Claim it first; checkpoints latch.

## Read four things out of the environment before writing any command

All four differ from what the lab text implies.

```
grep -n '<todo>' prometheus-app.yaml pod-monitoring.yaml hello-app/manifests/helloweb-deployment.yaml
sed -n '40,55p' hello-app/main.go
cat hello-app/Dockerfile
```

- **The lab's line numbers are wrong.** It says `pod-monitoring.yaml` lines 18-24, and `interval` is on **line 27**. A `sed '18,24s/<todo>/.../'` leaves the interval unset, and the four markers want four different values so a global replace is not an option either. Line addressed `sed` against the real numbers is the only safe form.
- **The Dockerfile has no `EXPOSE`.** Task 6 says to use "the target port of the container to the one specified in the Dockerfile", and what is actually there is `ENV PORT 8080`. Target port 8080.
- **The container name is `hello-app`**, from the deployment manifest, while the deployment is named `helloweb`. Needed for `set image`. `kubectl set image deployment/helloweb *=IMAGE` also works but bash tries to glob `*=...`, so naming the container is cleaner.
- **`main.go` line 49 is correct**, the one number in the lab that is.

## Do not diagnose a missing resource in the first minute

`gcloud artifacts docker images list` on `demo-repo` returned:

```
ERROR: NOT_FOUND: Requested entity was not found.
```

which reads as the repository not existing, contradicting the challenge's claim that the developers created it. It did exist. `repositories list` a few minutes later showed `CREATE_TIME: 07:37:50`, which was **after** the first command ran. Lab provisioning was still finishing.

The repo is genuinely empty at 0 MB, so the challenge's claim that it "contains a containerized version of the hello-app sample app" is not true, but that costs nothing since the point is to push `:v2` into it.

## Task 4 entirely from the CLI

The lab writes this task as Logs Explorer plus the Monitoring console. Both halves are commands.

**Validate the filter against real log entries before building anything on it.** The lab prints the two lines the query should return, which makes it verifiable rather than a guess:

```
gcloud logging read 'resource.type="k8s_pod" AND severity=WARNING' --limit=5 --freshness=10m --format="value(jsonPayload.message)"
```

That returned exactly the lab's expected output:

```
Error: InvalidImageName
Failed to apply default image tag "<todo>": couldn't parse image name "<todo>": invalid reference format
```

One resource type and one severity, which is what the lab's hint means. Then the metric:

```
gcloud logging metrics create pod-image-errors \
  --description="Count pod image errors on the cluster" \
  --log-filter='resource.type="k8s_pod"
severity=WARNING'
```

Then the alerting policy as a file. Every field in the lab's table maps to one JSON key:

| Lab field | JSON |
|---|---|
| Rolling Window: 10 min | `alignmentPeriod: 600s` |
| Rolling window function: Count | `perSeriesAligner: ALIGN_COUNT` |
| Time series aggregation: Sum | `crossSeriesReducer: REDUCE_SUM` |
| Condition type: Threshold | `conditionThreshold` |
| Threshold position: Above threshold | `comparison: COMPARISON_GT` |
| Threshold value: 0 | `thresholdValue: 0` |
| Alert trigger: Any time series violates | `trigger: {count: 1}` |
| Use notification channel: Disable | the key is simply absent |
| Alert policy name: Pod Error Alert | `displayName` |

```
cat > pod-error-alert.json <<'EOF'
{
  "displayName": "Pod Error Alert",
  "combiner": "OR",
  "enabled": true,
  "conditions": [
    {
      "displayName": "pod-image-errors above threshold",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/pod-image-errors\" AND resource.type=\"k8s_pod\"",
        "aggregations": [
          {
            "alignmentPeriod": "600s",
            "perSeriesAligner": "ALIGN_COUNT",
            "crossSeriesReducer": "REDUCE_SUM"
          }
        ],
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0,
        "duration": "0s",
        "trigger": { "count": 1 }
      }
    }
  ]
}
EOF

gcloud alpha monitoring policies create --policy-from-file=pod-error-alert.json
```

Scored 10 of 10 first attempt. **The lab's warning about unticking the Active filter tag is a console only problem**, since newly created log metrics are hidden behind that filter in the metric picker. From the CLI it does not exist.

Note `thresholdValue: 0` comes back **empty** in `policies list` output rather than as `0`, because gcloud omits zero values in table formatting. Not a sign it failed to set.

## The rest, in the order that worked

```
gcloud container clusters create $CLUSTER_NAME --zone=$ZONE \
  --release-channel=regular --num-nodes=3 \
  --enable-autoscaling --min-nodes=2 --max-nodes=6
```

`--zone` for a **zonal** cluster, which is what the lab's table specifies. Three nodes total; a regional cluster would give nine.

Then task 2's update, the namespace, and both manifests with `-n $NAMESPACE_NAME`. Task 3 is one `apply` of the deliberately broken manifest, and the pod going `InvalidImageName` is the goal rather than a problem.

Task 5:

```
sed -i '37s|<todo>|us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0|' hello-app/manifests/helloweb-deployment.yaml
kubectl -n $NAMESPACE_NAME delete deployment helloweb
kubectl -n $NAMESPACE_NAME apply -f hello-app/manifests/helloweb-deployment.yaml
kubectl -n $NAMESPACE_NAME rollout status deployment/helloweb --timeout=180s
```

`|` as the sed delimiter because the image path contains `/` and `:`. `--timeout` on `rollout status` for the GSP053 reason: it can block forever, and Ctrl+C out of a pasted block lets the remaining lines keep running.

Task 6:

```
export IMAGE=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/hello-app:v2
cd ~/hello-app
sed -i '49s|Version: 1.0.0|Version: 2.0.0|' main.go
grep -n 'Version:' main.go

gcloud auth configure-docker $REGION-docker.pkg.dev --quiet
docker build -t $IMAGE .
docker push $IMAGE

kubectl -n $NAMESPACE_NAME set image deployment/helloweb hello-app=$IMAGE
kubectl -n $NAMESPACE_NAME rollout status deployment/helloweb --timeout=180s
kubectl -n $NAMESPACE_NAME expose deployment helloweb --name=$SERVICE_NAME --type=LoadBalancer --port=8080 --target-port=8080

until [ -n "$(kubectl -n $NAMESPACE_NAME get svc $SERVICE_NAME -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for external ip"; sleep 10; done; echo "IP READY"
curl -s http://$(kubectl -n $NAMESPACE_NAME get svc $SERVICE_NAME -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'):8080
```

`grep` the version line immediately after the `sed`, before spending build time. A missed `sed` builds the old source and the only symptom is `Version: 1.0.0` in the response several minutes later.

The whole of task 6 ran in about two minutes, faster than the four to six expected, since the `golang:1.19.2` pull is quick from Cloud Shell and the pushed distroless image is tiny.

The `until` loop is one physical line. Split across lines the command substitution breaks and the loop exits immediately printing success.

## Capacity is not a problem here, unlike GSP053

The manifest requests `200m` cpu for a **single** replica on three `e2-medium` nodes. GSP053's trouble came from three plus a canary plus a green deployment at 200m each on `e2-small`. Nothing here comes close, so no scaling down is needed.

## Related files

- `gsp1026-managed-service-for-prometheus.md`, PodMonitoring and the in cluster collection path.
- `gsp364-managed-prometheus-challenge.md`, where the same `--enable-managed-prometheus` flag on `clusters create` **was** accepted by the checkpoint, which is the counterweight to this lab's finding.
- `gsp736-debug-apps-on-gke.md`, logs based metrics and alerting policies through the console, and the reason to grep for line numbers before a line addressed `sed`.
- `gsp053-managing-deployments-kubernetes-engine.md`, `rollout status` timeouts and deployment capacity.
- `gsp1131-artifact-registry-qwik-start.md`, the `REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG` path anatomy.
- `arc115-monitoring-challenge.md`, the other challenge lab built on logs based metrics.
