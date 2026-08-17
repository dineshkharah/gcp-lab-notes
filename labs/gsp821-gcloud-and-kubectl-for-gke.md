# GSP821, gcloud and kubectl for Google Kubernetes Engine

Three tasks, **two** checkpoints, fifteen minutes stated and **no cost**. About **three minutes**. Entirely the lab's own terminal.

Fifth and last of the required labs in the **Essential Google Cloud CLI Tools** course, and the shortest of the five in practice because **the cluster is pre created**.

Manual last updated January 2025, lab last tested November 2024.

## Both checkpoints are single kubectl commands

Checkpoint 1 is the Deployment, checkpoint 2 is the Service. Tasks 1 and 2 are config and credentials with nothing scored.

The cluster is `lab-cluster`, already running, so none of the seven minute create that GSP304 spends.

## The commands

```
gcloud config set compute/region us-east1
gcloud config set compute/zone us-east1-b
gcloud container clusters get-credentials lab-cluster
kubectl get nodes
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl rollout status deployment/hello-server --timeout=120s
```
Checkpoint 1.

```
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
until [ -n "$(kubectl get svc hello-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for load balancer ip"; sleep 10; done
export LB=$(kubectl get svc hello-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
kubectl get service
curl -s -w "\nHTTP %{http_code}\n" http://$LB:8080
```
Checkpoint 2.

**Setting the zone matters even though nothing is created in it.** `get-credentials lab-cluster` with no `--zone` relies on `compute/zone` being set, which is what task 1 is for. Skip task 1 and the credentials fetch fails with a cluster not found, which reads like the cluster is missing.

## The port here is 8080 on both sides, which is the opposite of its siblings

Worth reading against `gsp304-docker-image-to-kubernetes-challenge.md` and `gsp305-scale-out-update-containerized-app-challenge.md`, which both use `--port 80 --target-port 8000`.

```
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```

**`--target-port` defaults to `--port` when omitted.** So this Service listens on 8080 and forwards to 8080, because `hello-app` serves on 8080. That is why the test url is `http://IP:8080` and not a bare address.

The two shapes exist for different reasons: here the app's port is the port you want exposed, and in GSP304 the app serves 8000 while the requirement is to answer on 80, so both flags are needed. Omitting `--target-port` when the two differ is the 504 that GSP304's troubleshooting section describes.

## Two small improvements over the lab's own text

**The lab says to rerun `kubectl get service` if `EXTERNAL-IP` shows pending.** The `until` loop above does that for you. `-n` on the substitution rather than a comparison, because the field is **absent** while provisioning rather than empty.

**`rollout status --timeout=120s`** rather than assuming the Deployment came up, per the standing note in `gotchas.md`. Without a timeout it can hang indefinitely on a Deployment that will never become ready.

Expect `Hello, world!` with a version and hostname, and `HTTP 200`.

## One thing to know for later

`kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0` produces a Deployment named `hello-server` containing a container named **`hello-app`**, after the image rather than the deployment. Irrelevant here, but it is exactly what GSP305 turns into a trap when `kubectl set image` has to address the container by name.

## Related files

- `gsp693-gcloud-cli-beginners-guide.md`, `gsp694-gcloud-for-network-configuration.md`, `gsp695-manage-storage-configuration-gsutil.md` and `gsp685-bq-for-google-bigquery.md`, the rest of this course.
- `gsp304-docker-image-to-kubernetes-challenge.md` and `gsp305-scale-out-update-containerized-app-challenge.md` for the same commands as challenge labs, with the port pairing as a requirement rather than a given.
- `gsp053-managing-deployments-kubernetes-engine.md` for rolling updates and scaling on top of this.
