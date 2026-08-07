# ARC115, Monitoring in Google Cloud challenge lab

Five tasks, five checkpoints, twenty minutes. The vm `apache-vm` is prebuilt with Apache already running, so there is no vm to create and no Apache to install.

This is the three guided labs of the badge recombined. Nothing new is asked.

| ARC115 task | Comes from | Difference |
| --- | --- | --- |
| 1, install Ops Agent and configure it | `gsp1108-monitor-apache-with-ops-agent.md` | identical, including the `config.yaml` block |
| 2, uptime check | `gsp089-cloud-monitoring-qwik-start.md` | identical, Resource Type **URL** with the external ip |
| 3, alert policy | `gsp1108` | threshold is **3 KiB/s**, not 4 |
| 4, dashboard and two charts | `gsp089` and `gsp092` | CPU load (1m) for the vm, Requests for Apache |
| 5, log based metric | `gsp092` | a **filter based counter** here, not a Distribution on a latency field |

Read those three first. The work is not repeating a mistake between them.

## Task 1

Same Ops Agent install and the same `config.yaml` as GSP1108: `apache` receiver under metrics, `apache_access` and `apache_error` under logging, each wired into a `service.pipelines` entry. A receiver declared but not listed under a pipeline collects nothing, which is the failure mode to watch for.

```
sudo service google-cloud-ops-agent restart
sudo systemctl status google-cloud-ops-agent"*"
```

The checkpoint wants **both** `Logging Agent` and `Metrics Agent` running. `q` exits the pager.

Note there is no Apache install step and therefore no `php7.0` problem, unlike the two guided labs.

## Task 2

Uptime check, Resource Type **URL**, hostname is the vm's **external ip**. Click Test before Create.

## Task 3, threshold is 3000

Metric `workload/apache.traffic`, rolling window **rate**, above threshold, value **3000**. The lab says 3 KiB/s; the metric is bytes per second. GSP1108 uses 4000 for the same metric, so this is the one number not to copy across.

Traffic generator differs too, and is heavier:

```
timeout 120 bash -c -- 'while true; do curl localhost | grep -oP "<title>.*</title>"; sleep .1s;done '
```

A tenth of a second between requests rather than a random zero to three seconds. That is deliberate, since the threshold has to actually be crossed.

Create the email notification channel before opening the policy form, or use the Manage Notification Channels link and then the **refresh icon** next to the dropdown.

## Task 4

One dashboard, two line charts:

- **CPU load (1m)** filtered to VM Instance, Cpu
- **Requests** for Apache, which is `workload/apache.requests` under VM Instance, Apache

Untick **Active** in both metric pickers. Agent metrics do not appear otherwise.

## Task 5, the log based metric from the cli

The console path works, but this is cleaner and leaves nothing to mistype:

```
PROJECT_ID=$(gcloud config get-value project)

gcloud logging metrics create METRIC_NAME \
  --description="Count Apache 200 OK responses" \
  --log-filter='resource.type="gce_instance"
logName="projects/'"$PROJECT_ID"'/logs/apache-access"
textPayload:"200"'
```

Scored and accepted in our run. Three things about it:

- The filter is **multi line inside single quotes**, which is valid; Logging filters treat newlines as implicit `AND`.
- The project id is spliced in by closing the single quote, expanding `"$PROJECT_ID"`, and reopening. Necessary because the surrounding quotes are single and would not expand it.
- The metric name is free. Any name scores.

This is a **counter** metric with a filter, unlike GSP092 which needs a **Distribution** metric extracting `httpRequest.latency`. Do not carry the Distribution setting over.

The lab's closing line about exploring `VM Instance > Apache > Workload/apache.requests` is exploration of an existing agent metric, not part of creating the log based metric. Easy to read as a required step.

## Note on scoring

Our run showed **80 of 100** in the header while all five tasks displayed `Assessment Completed`. If that happens, refresh the lab page and re-click any checkpoint that has not registered; see the refresh entry in `gotchas.md`.
