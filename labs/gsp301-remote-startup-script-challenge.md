# GSP301, Deploy a Compute Instance with a Remote Startup Script: Challenge Lab

Four tasks, **four** checkpoints, fifteen minutes stated and **5 credits**. About **four minutes of runtime**, entirely Cloud Shell.

**No SSH session at all**, which makes it shorter than `gsp101-deploy-troubleshoot-website-challenge.md` despite sounding harder: the startup script does the Apache install, so there is nothing to run on the instance by hand.

Manual last updated and last tested July 2026.

## What it is

GSP101 with one idea added. The Apache install script lives in a **Cloud Storage bucket** and the instance references it by URL, rather than the script being pasted into instance metadata. Everything else, the tag pairing and the firewall rule and the `http://` test, is identical to GSP101 and covered there.

Only the zone is randomised. The bucket and instance names are not specified anywhere, so anything works; the project id for the bucket and `web-server` for the instance are fine.

The script itself is provided at `gs://spls/gsp301/install-web.sh` and you never write it. Task 1 is copying it into a bucket you own. Read it before using it:

```
gcloud storage cat gs://$BUCKET/install-web.sh
```

## The metadata key is startup-script-url, not startup-script

The single most important detail, and the failure is silent.

```
gcloud compute instances create $VM --zone=$ZONE --machine-type=e2-medium \
  --tags=http-server \
  --metadata=startup-script-url=gs://$BUCKET/install-web.sh
```

`startup-script` expects the **body** of the script inline. `startup-script-url` expects a **location** to fetch. Hand a `gs://` URL to the first one and the guest environment dutifully executes that URL as if it were a shell script, which does nothing useful. The VM boots cleanly, reports `RUNNING`, and Apache never appears, with no error visible from outside.

Confirm before moving on:

```
gcloud compute instances describe $VM --zone=$ZONE --format="yaml(metadata)"
```

## The permissions tip is about the bucket, not the scopes

The lab spends more words on this than anything else: *"Your Compute Engine instance might not have the correct permissions required to read the startup script from the storage bucket."*

Two halves, and only one of them is a default. A new instance **does** get `devstorage.read_only` in its default scope set, so no `--scopes` flag is needed. What is not guaranteed is that the service account has read on the object. Granting it explicitly costs one command:

```
gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/storage.objectViewer
```

Scopes and iam are two separate gates and both have to be open. Scopes bound what the instance's token is allowed to request; iam decides whether that identity may read the object. A default instance passes the first and can still fail the second.

## The firewall half is GSP101 verbatim

```
gcloud compute firewall-rules create default-allow-http --network=default \
  --allow=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```

`--target-tags=http-server` matching `--tags=http-server` on the instance, and the default network still has no rule for port 80 of its own. See `gsp101-deploy-troubleshoot-website-challenge.md` for why.

## Task 4 needs a wait, and HTTP 000 is not a firewall problem

The instance reports `RUNNING` well before the startup script has finished `apt install apache2`, so an immediate `curl` returns `HTTP 000`. Per the refused versus timeout note in `gotchas.md`, that means nothing is listening yet rather than the rules being wrong.

Bounded wait on one physical line, twelve tries at fifteen seconds:

```
export IP=$(gcloud compute instances describe $VM --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
for i in $(seq 1 12); do [ "$(curl -s -m 5 -o /dev/null -w '%{http_code}' http://$IP)" = "200" ] && echo "SERVING" && break; echo "waiting for apache ($i)"; sleep 15; done
curl -s -w "\nHTTP %{http_code}\n" http://$IP
```

## The serial console separates the three failure modes

The lab points you at the console for this; it reads fine from Cloud Shell:

```
gcloud compute instances get-serial-port-output $VM --zone=$ZONE | grep -i -E "startup|apache|denied"
```

Three outcomes that look identical from outside the instance:

- **no mention of the script at all** → wrong metadata key
- **fetched and refused** → the service account cannot read the object
- **fetched and still running** → just slow, wait longer

Worth reaching for before changing anything, since two of those three would be made worse by recreating the instance.

## One constraint

**Do not touch the pre-created `lab-monitor` instance.** The lab states that activity tracking depends on it, which puts it in the same category as GSP306's twenty points for a running blog: a resource whose job is to be left alone.

## Related files

- `gsp101-deploy-troubleshoot-website-challenge.md`, the same lab without the bucket, and the source of the tag pairing and refused versus timeout notes.
- `gsp089-cloud-monitoring-qwik-start.md` and `gsp1108-monitor-apache-with-ops-agent.md` for Apache on Compute Engine with monitoring on top.
