# GSP322, Build a Secure Google Cloud Network challenge lab

Six checkpoints. Twenty minutes stated and it is about right, all gcloud. Two vms (`bastion`, `juice-shop`) in an `acme-vpc` with two subnets are prebuilt, and the job is to lock down the firewall.

The three network tags are handed to you on the lab page and are per run, ending in a number like `-ql-118`. Everything else can be discovered.

## What the lab is actually asking

Four requirements, and the phrase that matters is "overly permissive permissions will not be marked correct":

- bastion has **no public ip**
- ssh to bastion **only via IAP**, so source `35.235.240.0/20` and nothing wider
- ssh to juice-shop **only from the management subnet**
- **only http** open to the world for juice-shop

## Discover first, four things

```
export ZONE=YOUR_ZONE
export REGION=${ZONE%-*}

export NETWORK=$(gcloud compute instances describe juice-shop --zone=$ZONE \
  --format="value(networkInterfaces[0].network.basename())")
export MGMT_CIDR=$(gcloud compute networks subnets describe acme-mgmt-subnet \
  --region=$REGION --format="value(ipCidrRange)")
echo "NETWORK=$NETWORK  MGMT_CIDR=$MGMT_CIDR"

gcloud compute firewall-rules list \
  --format="table(name,network,sourceRanges.list(),allowed[].map().firewall_rule().list(),targetTags.list())"

gcloud compute instances list \
  --format="table(name,status,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

Our run: network `acme-vpc`, mgmt subnet `192.168.10.0/24`, bastion `TERMINATED` at `192.168.10.2` with **no** nat ip, juice-shop `RUNNING` at `192.168.11.2`.

The single overly permissive rule was **`open-access`** on `acme-vpc`, `0.0.0.0/0` on all of `tcp`. The four `default-*` rules sit on the unused `default` network and can be left alone; the checkpoint does not want them.

Bastion already had no external ip in our run, so nothing to remove. Check anyway, because the requirement is explicit and no script handles it:

```
export NAT=$(gcloud compute instances describe bastion --zone=$ZONE \
  --format="value(networkInterfaces[0].accessConfigs[0].name)")
if [ -n "$NAT" ]; then
  gcloud compute instances delete-access-config bastion --zone=$ZONE --access-config-name="$NAT"
fi
```

Do that while the instance is still stopped, before starting it.

## The commands

```
gcloud compute firewall-rules delete open-access --quiet

gcloud compute instances start bastion --zone=$ZONE

gcloud compute firewall-rules create allow-ssh-iap \
  --network=$NETWORK --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 --target-tags=$IAP_TAG

gcloud compute firewall-rules create allow-http \
  --network=$NETWORK --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 --target-tags=$HTTP_TAG

gcloud compute firewall-rules create allow-ssh-internal \
  --network=$NETWORK --allow=tcp:22 \
  --source-ranges=$MGMT_CIDR --target-tags=$INT_TAG

gcloud compute instances add-tags bastion --zone=$ZONE --tags=$IAP_TAG
gcloud compute instances add-tags juice-shop --zone=$ZONE --tags=$HTTP_TAG,$INT_TAG
```

`35.235.240.0/20` is the IAP tcp forwarding range and is the only correct source for the bastion rule. `0.0.0.0/0` there scores nothing.

juice-shop needs **both** tags. Rule names are free; only the tags are dictated.

Verify the tags rather than trusting the two `add-tags` calls:

```
gcloud compute instances describe bastion --zone=$ZONE --format="value(tags.items)"
gcloud compute instances describe juice-shop --zone=$ZONE --format="value(tags.items)"
```

Both instances also carry leftover tags from the original build, `allow-ssh` on bastion and `lab-vm` on juice-shop. Harmless once `open-access` is gone, since no remaining rule targets them, and the scorer does not mind.

## The ssh chain

Run these one at a time. Both generate an ssh key on first use and prompt for a passphrase, which is a reason not to bundle them.

```
gcloud compute ssh bastion --zone=$ZONE --tunnel-through-iap
```

then from inside bastion:

```
gcloud compute ssh juice-shop --zone=YOUR_ZONE --internal-ip
```

`--tunnel-through-iap` is explicit because bastion has no external ip. `--internal-ip` on the second hop is what proves the mgmt subnet rule works.

The lab mentions `--troubleshoot` on `gcloud compute ssh` if the tunnel misbehaves. Did not need it.

## A filter that gcloud pushes server side and the api rejects

Worth recording as a gcloud lesson rather than a lab one. This looks reasonable and fails:

```
gcloud compute firewall-rules list --filter="network:$NETWORK AND sourceRanges:0.0.0.0/0"
```

```
ERROR: Invalid value for field 'filter': '(network eq ".*\bacme\-vpc\b.*")(sourceRanges eq ".*\b0\.0\.0\.0/0\b.*")'.
Invalid list filter expression.
```

Compute's list api only accepts a restricted filter grammar, and gcloud translated the compound expression into something it refuses. The failure cost the "Remove the overly permissive rules" checkpoint on the first pass: the `for` loop that was supposed to delete matching rules iterated over nothing, printed two errors amid a wall of successful output, and `open-access` survived. Everything else in the same paste worked, so the score came back 90 of 100 with no obvious cause.

Two takeaways. For a handful of rules, **list them, read them, delete by name** rather than filtering. And a loop over a failed command substitution is silent, so if a delete loop matters, echo the names first and check the list is not empty.

## The community script for this lab

Do not paste it. Four separate problems:

- **Four `read` prompts at the top.** Pasting the block means each `read` swallows the following line of the script. Same failure as ARC114.
- It deletes a rule named `open-access` without checking. Right in our run, but the name and count are not guaranteed.
- It hardcodes `192.168.10.0/24` for the mgmt subnet. Also right in our run. Derive it.
- It never checks whether bastion has an external ip, which is one of the four stated requirements.

It also assumes the network is `acme-vpc`, and its final `gcloud compute ssh bastion` omits `--tunnel-through-iap`.
