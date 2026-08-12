# GSP364, Monitor Environments with Google Cloud Managed Service for Prometheus: Challenge Lab

Four tasks, **two** checkpoints, fifteen minutes stated, **5 credits**. About **eight minutes of runtime**, nearly all of it the cluster create. Entirely Cloud Shell.

Checkpoints: **Check if prometheus has been deployed** after task 3, covering tasks 1 to 3, and **Check if metrics filter has been applied** after task 4.

The challenge lab for `gsp1026-managed-service-for-prometheus.md`, which shares the bucket upload verification pattern and the acl trap.

Manual last updated May 2026, **lab last tested October 2023**. Nearly three years, and two of the commands it hands you no longer work.

## Two commands in the lab text are broken

**The bucket create does not parse:**

```
gcloud storage buckets create -p $PROJECT gs://$PROJECT
ERROR: (gcloud.storage.buckets.create) unrecognized arguments: -p
```

`-p` is a **gsutil `mb`** flag that did not survive the move to `gcloud storage`. Left as written, every command after it fails against a bucket that never existed, and the checkpoint has nothing to read. Correct form:

```
gcloud storage buckets create gs://$PROJECT --location=US
```

**The public read grant does not work either.** The lab gives `gsutil -m acl set -R -a public-read gs://$PROJECT`, which uniform bucket level access refuses outright. Skip straight to the iam binding, which works whether or not uniform access is on:

```
gcloud storage buckets add-iam-policy-binding gs://$PROJECT \
  --member=allUsers --role=roles/storage.objectViewer
```

## Checkpoint 2 reads a public URL, not your cluster

Despite being titled "Check if metrics filter has been applied", the only thing it can see is the `op-config.yaml` you uploaded and made world readable. So verify that directly before clicking:

```
curl -s -o /dev/null -w "%{http_code}\n" https://storage.googleapis.com/$PROJECT/op-config.yaml
```

**200 or the checkpoint cannot pass**, however correct the cluster is. In our run this printed `404` first, which is how the broken `-p` was caught: `403` would have meant a permissions problem, `404` meant no bucket at all.

The reverse also holds. The file scores on its own, so a correct upload would pass even if the cluster patch had failed. Do both anyway; the task asks for both.

## The flag alone does the whole of tasks 1 and 2

Task 1's note says to use a flag on `clusters create`, and task 2's note points at `setup.yaml` and `operator.yaml` from `GoogleCloudPlatform/prometheus-engine`. Those are **two alternative paths**, not two steps: the manifests are the self deployed route for clusters *without* the flag, and running both puts two operators on one cluster.

`--enable-managed-prometheus` on create installed everything by itself, within about forty seconds of the cluster reporting ready:

```
gcloud container clusters create gmp-cluster \
  --num-nodes=3 --zone=$ZONE --enable-managed-prometheus
```

```
===== MANAGED PROMETHEUS ENABLED =====
True
===== GMP-SYSTEM =====
collector-l976k                 0/2     Init:0/2
collector-rmkdk                 0/2     Init:0/2
collector-vmk4t                 0/2     Init:0/2
gmp-operator-54bb4fd85f-4vvb2   1/1     Running
===== OPERATORCONFIG =====
config   37s
```

One collector per node plus the operator, and the `config` OperatorConfig in `gmp-public` that task 4 edits. **Checkpoint 1 passed on that plus the example app**, with `setup.yaml` and `operator.yaml` never applied.

Worth checking all three of those before clicking, since the collectors are still in `Init` at that point and may need another minute:

```
gcloud container clusters describe gmp-cluster --zone=$ZONE --format="value(monitoringConfig.managedPrometheusConfig.enabled)"
kubectl get pods -n gmp-system
kubectl -n gmp-public get operatorconfig
```

If `gmp-public` comes back empty, then and only then apply the two manifests, **without `-n`** since they carry their own namespaces and cluster scoped CRDs:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/manifests/setup.yaml
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/manifests/operator.yaml
```

Only the example app belongs in `gmp-test`:

```
kubectl create ns gmp-test
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/example-app.yaml
```

Its deployment is named `prom-example`, which is what the task 4 filter's `{job="prom-example"}` matches.

## The opposite of GSP510, on the same flag

Worth putting side by side, because generalising from either one alone is wrong.

- **GSP510** left its checkpoint at 15 of 20 with `Please enable Managed Prometheus on the cluster` after `--enable-managed-prometheus` on **create**, and only a separate `gcloud container clusters update --enable-managed-prometheus` cleared it, with the config already reading enabled beforehand.
- **GSP364** asks for the flag on create and its checkpoint accepted exactly that.

The difference is what each lab instructs. GSP510 presents enabling as its own task, GSP364 presents it as a flag on the create command. Follow the lab's framing rather than a habit carried from the other one, and if a checkpoint sticks, the `clusters update` is the first thing to try.

## Task 4, patch the live object rather than pasting a file

The lab says `vi op-config.yaml` and "copy the contents of operatorconfig inside" it. Patching and then exporting gets the same result with no editor and no guessed field values:

```
kubectl -n gmp-public patch operatorconfig config --type=merge \
  -p '{"collection":{"filter":{"matchOneOf":["{job=\"prom-example\"}","{__name__=~\"job:.+\"}"]}}}'

kubectl -n gmp-public get operatorconfig config -o yaml > op-config.yaml
grep -A5 'filter' op-config.yaml
```

The JSON patch sits inside single quotes so the escaped double quotes reach `kubectl` intact.

**The exported object is `apiVersion: monitoring.googleapis.com/v1`.** That matters: the circulating script hardcodes a file with `v1alpha1`, captured from a cluster in March 2022, which a current cluster would reject. Reading the live object means the apiVersion, the external labels and the managed alertmanager block all come from your own cluster.

## The community script, and what is wrong with it

Five things, beyond the `v1alpha1` above:

- **It omits `--enable-managed-prometheus`**, the one flag task 1 explicitly demands.
- **It applies `setup.yaml` and `operator.yaml` with `-n gmp-test`**, which is wrong for manifests that declare their own namespaces and cluster scoped resources.
- **It never applies the filter to the cluster**, only writes the file and uploads it. That scores, but it does not do what task 4 asks.
- **Its `op-config.yaml` is a dump of a stranger's live object**, carrying `creationTimestamp: "2022-03-14T22:34:23Z"`, `resourceVersion: "2882"` and someone else's `uid`.
- **It derives the zone from project metadata**, which returns empty when that metadata is unset. Same failure recorded in `gsp1079-continuous-delivery-cloud-deploy.md`. Use the zone off the lab page.

## Related files

- `gsp1026-managed-service-for-prometheus.md`, the guided lab, for PodMonitoring and the same bucket upload checkpoint.
- `gsp510-manage-kubernetes-challenge.md`, for the contrary behaviour of the same flag and for Managed Prometheus inside a wider GKE challenge.
