# GSP693, gcloud CLI: A Beginner's Guide

Three tasks, **one** checkpoint, ten minutes stated and **no cost**. About **two minutes** of actual work. Entirely the lab's own terminal.

First of the five required labs in the **Essential Google Cloud CLI Tools** course, which is 75 minutes total across GSP693 and four siblings covering networking, gsutil, bq and GKE.

Manual last updated and last tested June 2026.

## Only task 2 is scored

Task 1 connects and installs nginx, task 3 reads logs, and neither has a checkpoint. Everything that scores is the firewall rule plus the network tag.

The VM `gcelab2` is pre created in the `default` network, and only the zone is randomised.

## nginx is already installed

```
sa_...@gcelab2:~$ sudo apt install -y nginx
nginx is already the newest version (1.22.1-9+deb12u9).
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

So task 1 is a no op in practice, which matters for the next section: the service is **already listening on port 80** before you touch anything.

## This lab is the clean demonstration of drop versus refuse

The most useful thing in it, and the reason to remember it.

The lab has you `curl` the VM before creating the firewall rule and warns *"the nginx server will not respond and you will see a frozen remote shell. Press Ctrl-c."* That hang is the point:

- nginx **is** running and **is** listening on 80
- the packet is **dropped** by the absence of a rule, so nothing ever answers
- after the rule and tag, the same `curl` returns `200` instantly

Same host, same port, same listener. The only variable is whether the packet is dropped. A **refusal** would have come back immediately and would have meant nothing was listening, which is a different problem with a different fix. Getting those two the wrong way round is what sends people auditing rules that were already correct, per `gotchas.md`.

Bound it rather than hanging, if you are pasting a block:

```
curl -s -m 5 -o /dev/null -w "before firewall: HTTP %{http_code}\n" http://$VM_IP
```

`HTTP 000` there is the same information as the hang, without stalling the paste.

## The scored commands

```
export ZONE=us-west1-a
gcloud compute ssh gcelab2 --zone=$ZONE --quiet --command="sudo apt-get update -qq && sudo apt-get install -y nginx && systemctl is-active nginx"
export VM_IP=$(gcloud compute instances list --filter=name:gcelab2 --format='value(EXTERNAL_IP)')
curl -s -m 5 -o /dev/null -w "before firewall: HTTP %{http_code}\n" http://$VM_IP
gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
gcloud compute instances add-tags gcelab2 --tags=http-server --zone=$ZONE
gcloud compute firewall-rules list --filter=ALLOW:'80'
gcloud compute instances list --filter='tags:http-server'
curl -s -m 10 -w "\nafter firewall: HTTP %{http_code}\n" http://$VM_IP
```

**Both halves of the tag pairing are required and neither is a default.** `--target-tags=http-server` on the rule and `add-tags http-server` on the instance. Creating the rule alone changes nothing, which is precisely what GSP101 tests as a challenge with no guidance at all.

**`--quiet` on the `ssh`.** The lab shows the passphrase prompt and says press Enter twice. In a pasted block that prompt consumes the next line, the ARC114 trap in `gotchas.md`. `--quiet` takes the empty default, and `--command` means no interactive session and no `exit`.

**`apt-get update` before the install**, since a stale package index occasionally breaks `apt install` on a fresh image. Harmless here given nginx is already present.

Also worth noting `gcloud compute instances list --filter='tags:http-server'`, which is how you confirm the second half landed. A rule that lists correctly and an instance that is not tagged look identical from the rule's side.

## Task 3, unscored but reusable

```
gcloud logging logs list
gcloud logging logs list --filter="compute"
gcloud logging read "resource.type=gce_instance" --limit 5
gcloud logging read "resource.type=gce_instance AND labels.instance_name=gcelab2" --limit 5
```

The last form is the one to keep: `resource.type` plus `labels.instance_name` reads a single VM's logs with no console. That is the query shape GSP303 needed when `get-serial-port-output` returned nothing on three consecutive attempts.

## Related files

- `gsp101-deploy-troubleshoot-website-challenge.md`, the same tag pairing and the same `curl` test as a challenge lab, and the source of the refused versus timeout note.
- `gsp301-remote-startup-script-challenge.md` for the same setup with the Apache install moved into a startup script.
- `gsp399-design-implement-network-security-challenge.md` for where network tags go once Cloud NGFW and secure tags replace them.
