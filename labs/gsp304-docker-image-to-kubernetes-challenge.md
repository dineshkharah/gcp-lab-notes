# GSP304, Build and Deploy a Docker Image to a Kubernetes Cluster: Challenge Lab

Four tasks, **four** checkpoints, twenty minutes stated and **5 credits**. About **ten minutes** of runtime, entirely Cloud Shell.

No console, no SSH, no uploads. Two blocks, and the split exists only because `kubectl` cannot run until the cluster is `RUNNING`.

Checkpoints: the cluster, the pushed image, the deployment, the service.

Manual last updated and last tested July 2026, so this one is current.

## Every name is fixed, and only the zone is a variable

The note under the scenario names all four: image repository `echo-app`, cluster `echo-cluster`, deployment `echo-web`, and the zone. Nothing is randomised except the zone and the project id, which makes this the rare challenge lab where the values in a set of notes are almost the whole answer.

The zone appears twice in the lab text, once in the note and once in the troubleshooting section, and the second mention is the useful one: *"Not receiving assessment score for the last three objectives: this might just indicate that you have created your Kubernetes cluster in the different zone."* Three of the four checkpoints look the cluster up by zone.

## Two node count defaults have to be overridden

```
gcloud container clusters create echo-cluster --zone=$ZONE \
  --num-nodes=2 --machine-type=e2-standard-2 --async
```

`clusters create` gives you **three** nodes and an `e2-medium` by default. The task phrases the requirement as a capacity limit, *"limited in capacity, so you should limit the test cluster to just two e2-standard-2 instances"*, which reads like advice rather than a specification. It is a specification, and both flags are needed.

## Overlap the build with the cluster, using --async

The cluster takes five to seven minutes and **nothing in tasks 2 and 3 depends on it**. Running them in sequence wastes that whole window.

`--async` returns immediately, then a bounded wait sits between the push and `get-credentials`:

```
until [ "$(gcloud container clusters describe echo-cluster --zone=$ZONE --format='value(status)' 2>/dev/null)" = "RUNNING" ]; do echo "cluster still provisioning"; sleep 20; done
gcloud container clusters get-credentials echo-cluster --zone=$ZONE
```

The `2>/dev/null` matters: for the first few seconds after `--async`, `describe` errors rather than returning a status, and per the failed substitution note in `gotchas.md` an unguarded loop condition on an erroring command behaves unpredictably. Fifteen minutes collapses to ten this way. See also the async warning at the top of `gotchas.md`, which is the same idea from the direction where it went wrong.

## The bucket is named after the project and the object is never named

*"The archive has been copied to a Cloud Storage bucket belonging to your lab project called gs://[PROJECT_ID]."* The bucket is the bare project id, and the object is the `echo-web.tar.gz` mentioned a sentence earlier:

```
gcloud storage cp gs://$PROJECT/echo-web.tar.gz .
tar -xzf echo-web.tar.gz
```

Same pattern as GSP364's public bucket named for the project. `$DEVSHELL_PROJECT_ID` is already set in Cloud Shell, so nothing needs pasting.

## Find the build context rather than assuming it

The archive's layout is not documented, and `docker build .` against the wrong directory fails with a confusing `Dockerfile not found` after the download appeared to succeed. One line removes the guess:

```
export CTX=$(dirname $(find . -name Dockerfile | head -1))
docker build -t gcr.io/$PROJECT/echo-app:v1 $CTX
```

Reading the Dockerfile first is also worth the second it costs, since it is where the port 8000 claim in task 3 can be confirmed rather than trusted.

## The tag is the whole of tasks 2 and 3

```
gcloud auth configure-docker gcr.io --quiet
docker push gcr.io/$PROJECT/echo-app:v1
gcloud container images list-tags gcr.io/$PROJECT/echo-app
```

The image name is not a label, it is the destination. `gcr.io/PROJECT/echo-app:v1` encodes the registry host the lab insists on, the project, the repository name the note fixes, and the `v1` tag, and a `docker push` of anything else goes somewhere the scorer does not look. Build it with the final tag rather than tagging afterwards.

`--quiet` on `configure-docker` because it prompts to overwrite the Docker config, which in a pasted block would eat the next line.

Checkpoint 2 does not touch the cluster, so claim it as soon as `list-tags` shows the tag rather than waiting for the end of the lab.

## Port 80 outside, 8000 inside

```
kubectl create deployment echo-web --image=gcr.io/$PROJECT/echo-app:v1
kubectl expose deployment echo-web --type=LoadBalancer --port=80 --target-port=8000
```

`--port` is what the load balancer listens on, `--target-port` is what the container listens on. Reversing them is the 504 the troubleshooting section describes, and it is a plausible mistake because the task sentence leads with 8000: *"even though the application is configured to respond on port 8000, you must configure the service to respond to normal web requests on port 80."*

Two separate objects get created here, a Deployment and a Service, and there is a checkpoint for each.

## The service checkpoint needs an external IP, so wait for it

`expose` returns instantly; the load balancer takes a minute or two, and until then the service has a pending external IP and answers nothing.

```
kubectl rollout status deployment/echo-web --timeout=180s
until [ -n "$(kubectl get svc echo-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for load balancer ip"; sleep 15; done
export LB=$(kubectl get svc echo-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s -w "\nHTTP %{http_code}\n" http://$LB
```

`-n` on the substitution rather than a comparison, because the field is absent rather than empty while provisioning. `--timeout` on `rollout status` per the standing note in `gotchas.md`. And `http://`, not `https://`, for the same reason as GSP101.

`HTTP 200` and a page reporting the hostname and environment means the last checkpoint is ready.

## Related files

- `gsp053-managing-deployments-kubernetes-engine.md` for `kubectl` deployments and services at more depth, including the rolling update commands this lab does not reach.
- `gsp1131-artifact-registry-qwik-start.md` for the same build and push cycle against Artifact Registry rather than the `gcr.io` hostname this lab mandates.
- `gsp1077-gke-pipeline-using-cloud-build.md` for the same image into the same kind of cluster, built by Cloud Build instead of by hand.
- `gsp510-manage-kubernetes-challenge.md` and `gsp736-debug-apps-on-gke.md` for what goes wrong once something is running on GKE.
