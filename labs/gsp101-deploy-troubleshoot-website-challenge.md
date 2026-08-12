# GSP101, Deploy and Troubleshoot a Website: Challenge Lab

Four tasks, **three** checkpoints, ten minutes stated and **5 credits**. About **four minutes of runtime**, entirely Cloud Shell. The smallest challenge lab in these notes.

Checkpoints: tasks 1 and 2 together, then task 3, then task 4. Two randomised values, the instance name and the zone.

Manual last updated March 2024, lab last tested September 2023. Stale, but nothing in it has rotted.

## The whole lab is one sentence: tag the VM and the rule identically

Task 2 says only "apply the appropriate firewall rules". Two things have to line up and neither is a default:

```
gcloud compute instances create $INSTANCE --zone=$ZONE --machine-type=e2-medium \
  --tags=http-server

gcloud compute firewall-rules create default-allow-http --network=default \
  --allow=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```

**A new project's default network has no rule for port 80.** It ships `default-allow-icmp`, `default-allow-internal`, `default-allow-rdp` and `default-allow-ssh`. The `default-allow-http` rule people half remember exists only because the console's *Allow HTTP traffic* checkbox creates it, so from the CLI you make it yourself.

**`--tags` on the instance and `--target-tags` on the rule must match.** A rule with no matching tag on the VM is precisely the cause the lab's own troubleshooting section names, and it is easy to create the rule, see it in the listing, and believe task 2 is done.

List what exists before creating, so an already present rule shows up rather than colliding:

```
gcloud compute firewall-rules list --format="table(name,allowed[].map().firewall_rule().list(),sourceRanges.list(),targetTags.list())"
```

No `--image` flag anywhere. gcloud's current default Debian is what you want for `apt install apache2`, and pinning an image family is a hostage to it being renamed.

## Connection refused is not a firewall problem

The most useful thing in this lab, and it generalises.

```
ssh: connect to host 35.196.7.108 port 22: Connection refused
```

That happened running the install about a minute after the instance was created, and it is **not** a rules failure. A GCP firewall denial **drops** packets, so a blocked port gives a **timeout**. A **refusal** means the packet arrived and nothing was listening, which here meant sshd had not started yet.

So refused means wait, timeout means check the rules. Getting that the wrong way round sends you auditing firewall rules that were correct all along.

Bounded wait, one physical line so it survives a paste:

```
for i in 1 2 3 4 5 6; do gcloud compute ssh $INSTANCE --zone=$ZONE --quiet --command='echo SSH_READY' && break; echo "sshd not up yet, waiting 15s"; sleep 15; done
```

`curl` has its own version of the same distinction: `HTTP 000` means no response at all, either nothing listening or no route, as against a `403` or `404` which means something answered.

## Task 3, and two quoting details

```
gcloud compute ssh $INSTANCE --zone=$ZONE --quiet --command='sudo apt-get update -qq && sudo apt-get install -y apache2 && echo "<html><body><h1>Hello World!</h1></body></html>" | sudo tee /var/www/html/index.html && sudo systemctl restart apache2 && systemctl is-active apache2'
```

**`--quiet` matters.** On first use `gcloud compute ssh` generates a key pair and prompts for a passphrase. That is the ARC114 trap: a prompt inside a pasted block consumes the next line as its answer. `--quiet` accepts the empty default.

**The `--command` is in single quotes with double quotes inside, not the other way round.** The `!` in `Hello World!` sits inside single quotes and so never reaches interactive bash's history expansion. Double quotes happen to survive `World!`, but a `<!doctype html>` prefix would not: `!doctype` is `!` followed by a letter, which bash treats as a history event and rejects with `event not found`. Dropping the doctype is the other way out.

**The default Apache page is not "Hello World!"** A fresh install serves the Apache2 Debian Default Page, so `index.html` has to be written. The lab states the expected page and never says to create it.

## Task 4

```
export IP=$(gcloud compute instances describe $INSTANCE --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
curl -s -w "\nHTTP %{http_code}\n" http://$IP
```

`curl` rather than a browser, and note **`http://`**. The lab warns about this and it is a real trap: there is no TLS on the instance, so an `https://` address gives a connection error indistinguishable from a firewall problem, and browsers increasingly upgrade bare addresses to https on their own. `curl` also prints the status code, where a browser can quietly show a cached page.

Expect the Hello World HTML and `HTTP 200`.

## Related files

- `gsp089-cloud-monitoring-qwik-start.md` and `gsp1108-monitor-apache-with-ops-agent.md` for the same Apache on Compute Engine setup, with monitoring on top.
- `gsp322-build-a-secure-network-challenge.md` for firewall rules and network tags at a larger scale, including a filter expression the Compute list api rejects.
