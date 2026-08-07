# GSP1026, Collect Metrics from Exporters using the Managed Service for Prometheus

Seven tasks, **two** checkpoints, twenty minutes stated and about thirty in practice. All Cloud Shell, but it needs **two tabs** because two of the commands never exit.

Checkpoints: **Check if prometheus has been deployed** after task 2, and **Check if config.yaml is configured correctly** after the bucket upload in task 7. Everything else, including the PromQL queries the lab builds towards, is unscored.

Manual last tested October 2023, which is much older than its May 2026 update date.

## The shape: two blocking processes, so three blocks not one

`./prometheus` and `./node_exporter` both run in the foreground and stay there. Everything else is scriptable, and **both checkpoints fall inside the scriptable part**, so the lab can be reduced to one paste plus two long running commands.

- **Tab 1**: everything scriptable, then `./prometheus`, in `~/prometheus`
- **Tab 2**: `./node_exporter` only, in `~/node_exporter-1.3.1.linux-amd64`

Start **node_exporter before prometheus**. Prometheus scrapes `localhost:9100` every 15 seconds, so with nothing listening yet its output fills with connection refused errors. Harmless, but it reads like a real failure.

## Block A, tab 1

```
until [ "$(gcloud container clusters list --format='value(status)' | sort -u)" = "RUNNING" ]; do gcloud container clusters list --format="csv(name,status)"; sleep 20; done; echo "CLUSTER RUNNING"

export PROJECT_ID=$(gcloud config get-value project)
export PROJECT=$PROJECT_ID
export ZONE=YOUR_ZONE

gcloud container clusters get-credentials gmp-cluster --zone=$ZONE

kubectl create ns gmp-test
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/example-app.yaml
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/pod-monitoring.yaml
kubectl get podmonitoring -A

cd ~
git clone https://github.com/GoogleCloudPlatform/prometheus && cd prometheus
git checkout v2.28.1-gmp.4
wget https://storage.googleapis.com/kochasoft/gsp1026/prometheus
chmod a+x prometheus

cat > config.yaml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
EOF
cat config.yaml

gsutil mb -p $PROJECT gs://$PROJECT
gsutil cp config.yaml gs://$PROJECT
gsutil -m acl set -R -a public-read gs://$PROJECT \
  || gcloud storage buckets add-iam-policy-binding gs://$PROJECT --member=allUsers --role=roles/storage.objectViewer
```

Three deviations from the lab, all safe:

- **Heredoc instead of `vi`** for `config.yaml`. Same file, nothing to mis-save, and it keeps the whole thing pasteable.
- **`||` fallback on the acl.** New buckets default to uniform bucket level access, where `gsutil acl set` fails outright. The iam binding achieves the same thing. Either path is fine.
- The cluster wait is **one physical line**, so it survives the paste. See `gotchas.md`.

The bucket is named after the project id, which is what the checkpoint looks for.

## Block B, tab 2

```
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
cd node_exporter-1.3.1.linux-amd64
./node_exporter
```

Wait for `Listening on address=:9100`. Leave it running.

## Block C, back in tab 1

```
cd ~/prometheus
export PROJECT=$(gcloud config get-value project)
export ZONE=YOUR_ZONE
./prometheus --config.file=config.yaml --export.label.project-id=$PROJECT --export.label.location=$ZONE
```

Re-exporting the two variables because a Cloud Shell tab does not inherit them; same reason `deploy/region` had to be repeated on GSP1079.

Then Web Preview, **Change Preview Port**, `9090`, Change and Preview. Query anything prefixed `node_`; `node_cpu_seconds_total` gives the clearest graph.

## Skip the lab's first prometheus run

Task 6 has you run prometheus against `documentation/examples/prometheus.yml`, look at a `up` query, then stop it and switch to `config.yaml`. Nothing is scored on it and you only stop it again a moment later. Going straight to `config.yaml` saves a few minutes and one process restart.

## What the pieces are

Worth keeping straight, because the lab mixes two unrelated collection paths:

- **In cluster**: `--enable-managed-prometheus` on the GKE cluster, plus a **PodMonitoring** custom resource. That scrapes the example app's pods automatically. A PodMonitoring CR only sees its own namespace; `ClusterPodMonitoring` is the cross namespace version.
- **On Cloud Shell**: the GMP prometheus binary scraping `node_exporter` on `localhost:9100`, with `--export.label.project-id` and `--export.label.location` telling the managed service where to file the metrics.

The GKE half and the Cloud Shell half never touch each other. Both write into the same managed Prometheus backend, which is the actual point of the lab.
