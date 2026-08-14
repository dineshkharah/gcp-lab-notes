# GSP305, Scale Out and Update a Containerized Application on a Kubernetes Cluster: Challenge Lab

Five tasks, **four** checkpoints, fifteen minutes stated and about **five**. Entirely Cloud Shell, two pastes.

The direct sequel to `gsp304-docker-image-to-kubernetes-challenge.md` and much shorter, because **the cluster is already created for you**. Tasks 1 and 2 share a checkpoint.

Manual last updated and last tested July 2026.

## The three commands the lab gives you are mandatory, not illustration

```
gcloud container clusters get-credentials echo-cluster --zone=$ZONE
kubectl create deployment echo-web --image=gcr.io/qwiklabs-resources/echo-app:v1
kubectl expose deployment echo-web --type=LoadBalancer --port 80 --target-port 8000
```

They sit above "Your challenge" and read like a demonstration. They are the starting state: the challenge is an **update**, so v1 has to be running first, and this is also what creates the Service that checkpoint 5 tests.

Note the v1 image is `gcr.io/qwiklabs-resources/echo-app:v1`, in **someone else's project**. Only v2 goes into yours.

## Task 3 is an update, and the wrong instinct creates a second Service

*"Deploy the updated application to the Kubernetes cluster. The deployment should be named echo-web and the application should be exposed on port 80."*

That phrasing invites `kubectl create deployment` and `kubectl expose` again. Both already exist. Exposing again produces a **second** Service on a **different** external IP, and the checkpoint tests the first one, so a perfectly working v2 reports as broken.

```
export CONTAINER=$(kubectl get deployment echo-web -o jsonpath='{.spec.template.spec.containers[0].name}')
kubectl set image deployment/echo-web $CONTAINER=gcr.io/$PROJECT/echo-app:v2
kubectl scale deployment echo-web --replicas=2
kubectl rollout status deployment/echo-web --timeout=180s
```

**The container is named `echo-app`, not `echo-web`.** `kubectl create deployment` names the container after the **image**, and `set image` addresses containers by name, so `echo-web=` fails with an unresolved reference that looks like the deployment is wrong. Deriving it costs one line and cannot be wrong. Both points are now in `gotchas.md`.

## Overlap the build with the load balancer

`expose` returns instantly and the load balancer takes a minute or two. Putting the download, build and push **after** the `expose` in the same block means that provisioning happens during the build rather than after it. Same trick as GSP304's `--async`, different mechanism: here the wait is simply moved rather than declared.

```
export ZONE=us-east1-c
export PROJECT=$DEVSHELL_PROJECT_ID
export IMAGE=gcr.io/$PROJECT/echo-app:v2
gcloud container clusters get-credentials echo-cluster --zone=$ZONE
kubectl create deployment echo-web --image=gcr.io/qwiklabs-resources/echo-app:v1
kubectl expose deployment echo-web --type=LoadBalancer --port 80 --target-port 8000
gcloud storage cp gs://$PROJECT/echo-web-v2.tar.gz .
tar -xzf echo-web-v2.tar.gz
export CTX=$(dirname $(find . -name Dockerfile | head -1))
docker build -t $IMAGE $CTX
gcloud auth configure-docker gcr.io --quiet
docker push $IMAGE
gcloud container images list-tags gcr.io/$PROJECT/echo-app
```

The bucket is the **bare project id**, `gs://qwiklabs-gcp-04-081c6905cf35`, so `$DEVSHELL_PROJECT_ID` covers it with nothing to paste. GSP304's draw appended `-bucket`; this one does not.

`find` for the Dockerfile rather than assuming the archive's layout, same reason as GSP304.

## Task 5 reads the response body, so wait for the pods to swap

```
until [ -n "$(kubectl get svc echo-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for load balancer ip"; sleep 15; done
export LB=$(kubectl get svc echo-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s -w "\nHTTP %{http_code}\n" http://$LB
```

Task 1 says *"V2 of the application adds a version number to the output"*, which is how the last checkpoint distinguishes the two builds. So a `200` is not enough on its own; the body has to mention version 2. Immediately after `set image` the old pods can still be serving, and a second `curl` a few seconds later shows the change.

`-n` on the substitution rather than a comparison, because the load balancer field is **absent** rather than empty while provisioning.

## Three checks that make the whole thing verifiable

```
kubectl get deployment echo-web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl get pods -l app=echo-web
```

The image must end `:v2`, and there must be **two** pods `Running`. Together with the `curl` body those cover checkpoints 3, 4 and 5 before any of them is clicked.

## Port 80 outside, 8000 inside

Inherited from the setup commands rather than something you choose, but the troubleshooting section's 504 is still this and only this. See `gsp304-docker-image-to-kubernetes-challenge.md`, where you do pick the numbers.

## Related files

- `gsp304-docker-image-to-kubernetes-challenge.md`, the prequel: same image, same registry, but you build the cluster and choose the ports.
- `gsp053-managing-deployments-kubernetes-engine.md` for rolling updates and scaling in more depth.
- `gsp510-manage-kubernetes-challenge.md` and `gsp736-debug-apps-on-gke.md` for diagnosing a deployment that will not come up.
