# GSP092, Monitoring and Logging for Cloud Run Functions

Five tasks, **two** checkpoints, twenty minutes. Mostly console. Cloud Shell appears only to run a load generator.

Checkpoints: **Creating a Cloud Run function** after task 1, **Create logs-based metric** after task 2. Tasks 3 to 5, Metrics Explorer, dashboards and the quiz, are **unscored**.

## Task 1, the function

Cloud Run, Services, **Write a Function**:

| Field | Value |
| --- | --- |
| Service name | `helloworld` |
| Region | the lab's region |
| Runtime | Node.js 22 |
| Authentication | Allow public access |
| Execution environment | **second generation** |
| Maximum number of instances | 5 |

Those last two are under the collapsed **Containers, Networking, Security** section, which is easy to click past. Second generation matters here for the same reason it does across the rest of these notes; see `gotchas.md`.

Accept the popup offering to enable required APIs, then **Save and Redeploy**. Green tick means done, a few minutes.

Claim the first checkpoint.

## Generating traffic, which the metric needs

The logs-based metric has nothing to measure until the function is actually hit. In Cloud Shell:

```
curl -LO 'https://github.com/tsenart/vegeta/releases/download/v12.12.0/vegeta_12.12.0_linux_386.tar.gz'
tar -xvzf vegeta_12.12.0_linux_386.tar.gz

CLOUD_RUN_URL=$(gcloud run services describe helloworld --region=YOUR_REGION --format='value(status.url)')
echo $CLOUD_RUN_URL

echo "GET $CLOUD_RUN_URL" | ./vegeta attack -duration=300s -rate=200 > results.bin
```

Vegeta is an http load testing tool that fires at a constant rate; here 200 requests a second for five minutes.

**Start this before task 2 and leave it running.** It is the reason the metric has data by the time you reach Metrics Explorer. The url is derived rather than copied out of the console, so there is nothing to mistype.

## Task 2, the logs-based metric

Observability, Logging, **Logs Explorer**. In the **All resources** dropdown pick **Cloud Run Revision**, then `helloworld`, Apply, then **Run query**.

Actions dropdown, **Create metric**:

| Field | Value |
| --- | --- |
| Metric Type | **Distribution**, not Counter |
| Log-based metric name | `CloudRunFunctionLatency-Logs` |
| Field name | `httpRequest.latency` |

Distribution is the part to get right. A Counter metric only counts matching log entries; Distribution extracts a **numeric value** out of each one, which is what makes `httpRequest.latency` meaningful and what lets task 4 render it as a heatmap.

Claim the second checkpoint. That is the lab scored.

## Tasks 3 and 4, unscored but where the delay lives

A new logs-based metric shows as **Not Active** in Metrics Explorer until it has seen enough data. That is not a mistake:

- untick the **Active** filter in the metric picker, or the metric will not appear at all
- give vegeta time to run, and log processing its own lag on top

Path in the picker: **Cloud Run Revision, Logs-based metric, Logging/user/CloudRunFunctionLatency-Logs**.

Task 4 builds a custom dashboard with four widgets: Request Count as a stacked bar, the latency metric as a **heatmap** (which only makes sense because the metric is a Distribution), Request_latency as a line chart with Mean aggregation, and Container CPU Allocation as a stacked bar. Rename the dashboard to `Cloud Run Function Custom Dashboard`.

## Quiz answers

- Two types of log-based metrics: **user-defined logs-based metrics** and **system logs-based metrics**
- Vegeta is an http load testing tool with a constant request rate: **True**
- Logs-based metrics are Cloud Monitoring metrics based on the content of log entries: **True**
