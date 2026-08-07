# GSP089, Cloud Monitoring: Qwik Start

Seven tasks, four checkpoints, twenty minutes. Console throughout, plus an ssh session on the vm. Cloud Shell only sets the default region and zone.

Checkpoints: create the vm, install Apache, get a response over the external ip, create an uptime check **and** alerting policy. Tasks 5 to 7, dashboards, logs and watching the alerts fire, are **unscored**.

The overlap with `gsp1108-monitor-apache-with-ops-agent.md` is large. Same badge, same Apache on Debian, same Ops Agent install. The differences are the uptime check and which metric the alert watches.

## Tasks 1 and 2

| Field | Value |
| --- | --- |
| Name | `lamp-1-vm` |
| Region / zone | the lab's, `us-west1` / `us-west1-a` in our run |
| Series / machine | E2 / `e2-medium` |
| Boot disk | Debian GNU/Linux 12 (bookworm) |
| Firewall | **Allow HTTP traffic** |

Only HTTP here, unlike GSP1108 which wants both HTTP and HTTPS. Missing it fails the third checkpoint, which is specifically about reaching the external ip.

```
sudo apt-get update
sudo apt-get install apache2 php7.0
sudo service apache2 restart
```

**`php7.0` does not exist on Debian 12.** The lab suggests falling back to `php5`, which does not exist either. Use:

```
sudo apt-get install apache2 php
```

Same trap as GSP1108, with a worse hint attached.

Then click the **External IP** on the VM instances page to get the Apache default page. If the column is missing, the Column Display Options icon at the right of the table adds it back. Two checkpoints here: Apache installed, and the external ip responding.

## The agents

Despite the prose describing separate "Monitoring agent" and "Logging agent" installs, both are the **same Ops Agent** and the same script:

```
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

sudo systemctl status google-cloud-ops-agent"*"
```

`q` to exit the status pager. The second block in the lab is a status check, not a second install, and no `config.yaml` editing is needed in this lab because the built in host metrics are enough.

## Task 3, uptime check

Monitoring, Uptime checks, Create Uptime Check.

| Field | Value |
| --- | --- |
| Protocol | HTTP |
| Resource Type | **URL** |
| Hostname | the vm's **external ip** |
| Check Frequency | 1 minute |
| Title | `Lamp Uptime Check` |

Everything else default. **Click Test before Create.** A green tick there means the probe can actually reach the vm; if it fails, the firewall box from task 1 is the cause and it is much cheaper to find out now.

Resource Type is `URL` with a bare ip, not `Instance`. Both work in principle but the checkpoint follows the lab.

## Task 4, alerting policy

Notification channel is created **mid form** here: Notification Channels dropdown, Manage Notification Channels, which opens a new tab, Add new for Email, then back to the original tab and hit the **refresh icon** next to the dropdown. Without that refresh the new channel does not appear and it looks like the save failed.

| Setting | Value |
| --- | --- |
| Metric | untick **Active**, filter `Network traffic`, VM instance, Interface, **Network traffic** (`agent.googleapis.com/interface/traffic`) |
| Threshold position | Above threshold |
| Threshold value | 500 |
| Retest window | 1 min, under Advanced Options |
| Name | `Inbound Traffic Alert` |

**Untick Active in the metric picker** or agent metrics will not be listed. `agent.googleapis.com/...` only exists because the Ops Agent is installed, and it takes a few minutes after install to show up.

Note this is a different metric from GSP1108, which alerts on `workload/apache.traffic` with a rate transform. This one is a plain threshold with no transform.

The single checkpoint covers **both** the uptime check and the alerting policy, so both must exist before clicking.

## Tasks 5 to 7, unscored

Dashboard `Cloud Monitoring LAMP Qwik Start Dashboard` with two line charts, CPU load (1m) and Received packets. Untick **Active** in both metric pickers, same as above.

Then Logs Explorer filtered to VM Instance, `lamp-1-vm`, and stopping and restarting the instance to watch the log entries and the uptime check go red then green. Regions can take up to five minutes to recover, so a failing uptime check straight after a restart is expected.

The lab's closing advice is worth following: **remove the email notification from the alerting policy** when done, or it keeps emailing you until the project is torn down.
