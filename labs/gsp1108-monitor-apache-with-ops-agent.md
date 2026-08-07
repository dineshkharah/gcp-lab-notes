# GSP1108, Monitor an Apache Web Server using Ops Agent

Six tasks, four checkpoints, fifteen minutes. Console for the vm and the alerting policy, ssh on the vm for everything in between. No Cloud Shell.

Checkpoints: create the vm, install Apache, install the Ops Agent, create the alerting policy. Tasks 4 and 6, generating traffic and watching the dashboard, are **unscored**.

## Task 1, the vm

| Field | Value |
| --- | --- |
| Name | `quickstart-vm` |
| Zone | the lab's zone |
| Series / machine type | E2 / `e2-small` |
| Boot disk | Debian GNU/Linux 12 (bookworm) |
| Firewall | tick **both** Allow HTTP traffic and Allow HTTPS traffic |

Those two firewall boxes are under **Networking** and are the only part of this screen that is easy to miss. Without HTTP the browser check in task 2 fails, and the natural instinct is to go hunting in VPC firewall rules rather than rebuild the instance.

## Task 2, Apache

```
sudo apt-get update
sudo apt-get install apache2 php7.0
```

`php7.0` **does not exist on Debian 12**. The lab notes this in passing; expect the command to fail and use:

```
sudo apt-get install apache2 php
```

Answer `Y` if prompted. Then open `http://EXTERNAL_IP` in a browser and accept the "does not support a secure connection" warning. `It works!` means done.

## Task 3, the Ops Agent

```
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
```

Then replace the agent config. This is the substance of the lab:

```
set -e
sudo cp /etc/google-cloud-ops-agent/config.yaml /etc/google-cloud-ops-agent/config.yaml.bak

sudo tee /etc/google-cloud-ops-agent/config.yaml > /dev/null << EOF
metrics:
  receivers:
    apache:
      type: apache
  service:
    pipelines:
      apache:
        receivers:
          - apache
logging:
  receivers:
    apache_access:
      type: apache_access
    apache_error:
      type: apache_error
  service:
    pipelines:
      apache:
        receivers:
          - apache_access
          - apache_error
EOF

sudo service google-cloud-ops-agent restart
sleep 60
```

The shape worth remembering: **receivers** declare what to collect, **service.pipelines** wire those receivers into an active pipeline. A receiver that is declared but not listed under a pipeline collects nothing, which is the usual reason an Ops Agent config looks right and produces no data.

Three receiver types here: `apache` for metrics, `apache_access` and `apache_error` for logs.

The `sleep 60` is not decoration. The checkpoint wants the agent up and reporting.

## Task 4, traffic

```
timeout 120 bash -c -- 'while true; do curl localhost; sleep $((RANDOM % 4)) ; done'
```

Two minutes of requests at a random zero to three second gap. Monitoring, Dashboards, **Apache Overview** — the dashboard is created automatically by the integration, you do not build it. Turn on Auto Refresh.

## Task 5, the alerting policy

Notification channel first: Monitoring, Alerting, **Edit notification channels**, Email, Add new, an address you can actually read, plus a display name. The policy form will not offer a channel that does not exist yet.

Then Alerting, **Create policy**:

| Setting | Value |
| --- | --- |
| Metric | filter on `VM instance`, category **Apache**, metric **workload/apache.traffic** |
| Rolling window | 1 min |
| Rolling window function | **rate** |
| Alert trigger | Any time series violates |
| Threshold position | Above threshold |
| Threshold value | **4000** |
| Incident autoclose duration | 30 min |
| Name | `Apache traffic above threshold` |

Two things to get right. **Rolling window function must be `rate`** — the raw metric is a cumulative byte count, so without the rate transform the threshold is meaningless and would trip permanently. And the threshold is **4000**, not 4, because the metric is in bytes per second while the lab describes it as 4 KiB/s.

The metric only appears in the picker once the Ops Agent has reported, so do task 3 and let it settle before this.

## Task 6

Re-run the same traffic command. The email takes several minutes to arrive and nothing is scored on it.
