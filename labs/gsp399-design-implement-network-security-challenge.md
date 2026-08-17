# GSP399, Design and Implement Network Security in Google Cloud: Challenge Lab

Three tasks, **three** checkpoints, ninety minutes stated and **1 credit**. About **forty five minutes**. **Entirely Cloud Shell.**

The first Cloud NGFW lab in these notes: global network firewall policies, secure IAM tags, and the `firewall-rules migrate` tool.

Manual last updated and last tested May 2026. Brand new, and it shows: the Setup section still contains the literal placeholder **`[add in boilerplate here]`**. Expect rough task wording rather than stale commands.

## Read the migrate tool's help before writing anything

The single most useful thing in this lab. Task 1's bullets read like descriptions of an approach:

> *Phased Migration (Exclusion Filters)* ... *Secure IAM Tags Mapping* ...

They are the tool's flag names.

```
gcloud beta compute firewall-rules migrate --help

  --source-network=SOURCE_NETWORK
  --target-firewall-policy=TARGET_FIREWALL_POLICY
  --export-exclusion-patterns / --exclusion-patterns-file=FILE
  --export-tag-mapping / --tag-mapping-file=FILE
  --bind-tags-to-instances
```

Every bullet in task 1 maps onto one of those. That turns what looks like a hand written migration into four invocations of a purpose built tool, and it preserves the actual legacy rules rather than approximating them. Now a section in `gotchas.md`.

## Discover before touching anything

```
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=europe-west1
export ZONE=europe-west1-c
export BASTION_IP=YOUR_BASTION_EXTERNAL_IP
gcloud config set project $PROJECT_ID

gcloud compute firewall-rules list --filter="network:unified-vpc" --format="table(name,direction,priority,sourceRanges.list(),allowed[].map().firewall_rule().list(),targetTags.list(),disabled)"
gcloud compute networks subnets list --filter="network:unified-vpc" --format="table(name,region.basename(),ipCidrRange,logConfig.enable)"
gcloud compute instances list --format="table(name,zone.basename(),networkInterfaces[0].subnetwork.basename(),tags.items.list())"
gcloud compute routers describe unified-router --region=$REGION --format="yaml(nats)"
gcloud beta compute firewall-rules migrate --help 2>&1 | sed -n '1,60p'
```

What that returned, which every subsequent command depends on:

| Legacy rule | Source | Allow | Target tags |
|---|---|---|---|
| `allow-iap-access` | 35.235.240.0/20 | tcp:22 | none |
| `allow-internal-vpc` | 10.0.0.0/8 | icmp, tcp:22, tcp:80 | none |
| `allow-ssh` | 0.0.0.0/0 | tcp:22 | `ssh` |
| `allow-web` | 0.0.0.0/0 | tcp:80 | `web` |

Subnets: `application-subnet` 10.10.10.0/24 and `secondary-subnet` 10.20.20.0/24, flow logs off on both.

Instances, **all three in `europe-west1-c` and all in `application-subnet`**: `bastion-host` tagged `bastion,ssh`; `compromised-vm` tagged `compromised-vm,ssh,web`; `private-instance` untagged.

**The NAT is on `secondary-subnet`.** That is task 2's whole bug, visible in one `describe`:

```
sourceSubnetworkIpRangesToNat: LIST_OF_SUBNETWORKS
subnetworks:
- name: .../subnetworks/secondary-subnet
```

## Two traps the discovery output exposes

**`compromised-vm` shares tags with the bastion.** It carries `compromised-vm`, `ssh` **and** `web`, and `bastion-host` also carries `ssh`. So task 3's rules must target **`compromised-vm` only**. Targeting all of a VM's tags, which is the obvious defensive move, would deny HTTP to the bastion and open forensics SSH to it, the opposite of containment. The circulating guide takes `tags.items[0]`, which is a coin flip among three.

**Task 1's cleanup deletes every classic rule on the VPC**, so it must run **before** task 3 creates `containment-deny-http` and `allow-forensics-ssh`. The circulating guide's cleanup loop is an unfiltered `for RULE in $(firewall-rules list --filter="network:unified-vpc")`, which would sweep them up. Order: **task 1, then 2, then 3.**

## Task 1

```
mkdir -p ~/fwmig && cd ~/fwmig
export VPC=unified-vpc
export POLICY=unified-fw-policy

printf 'allow-ssh\nallow-web\n' > exclusions.txt

gcloud beta compute firewall-rules migrate --source-network=$VPC --target-firewall-policy=$POLICY --exclusion-patterns-file=exclusions.txt
gcloud compute network-firewall-policies list --global
gcloud compute network-firewall-policies describe $POLICY --global --format="yaml(name,rules)" | sed -n '1,80p'

gcloud beta compute firewall-rules migrate --source-network=$VPC --export-tag-mapping --tag-mapping-file=tagmap.yaml
cat tagmap.yaml

gcloud resource-manager tags keys create unified-fw-tag \
  --parent=projects/$PROJECT_ID \
  --purpose=GCE_FIREWALL \
  --purpose-data=network=$PROJECT_ID/$VPC \
  --description="Secure IAM tag for unified-vpc firewall policy"
export TAG_KEY=$(gcloud resource-manager tags keys list --parent=projects/$PROJECT_ID --filter="shortName=unified-fw-tag" --format="value(name)")
gcloud resource-manager tags values create ssh --parent=$TAG_KEY --description="Migrated from legacy network tag ssh"
gcloud resource-manager tags values create web --parent=$TAG_KEY --description="Migrated from legacy network tag web"
gcloud resource-manager tags values list --parent=$TAG_KEY --format="table(shortName,name,namespacedName)"
```

**`exclusions.txt` holds the two tagged rule names**, since that file is what gets filtered *out* of the untagged phase. One regex per line, no leading or trailing whitespace, which `printf` gives cleanly.

**`--purpose-data=network=PROJECT_ID/NETWORK_NAME`** is the format. The circulating guide builds a full self link containing the network's **numeric id**, which `GCE_FIREWALL` does not accept, and that alone stops its task 1 before anything else runs.

**Do not pre-create the policy.** The migrate tool creates it; creating it first risks a conflict.

Then fill `tagmap.yaml` with the `ssh` and `web` to `tagValues/...` mappings, using the ids the `values list` printed, and bind them:

```
gcloud beta compute firewall-rules migrate --source-network=$VPC --tag-mapping-file=tagmap.yaml --bind-tags-to-instances
```

**Read `tagmap.yaml`'s schema from the export rather than assuming it.** That is what the `--export-tag-mapping` run is for, and it is the one file format in this lab worth looking at before editing.

Then the tagged rules, the association, the enforcement order and the cleanup:

```
gcloud compute network-firewall-policies rules create 1000 --firewall-policy=$POLICY --global-firewall-policy --direction=INGRESS --action=allow --src-ip-ranges=0.0.0.0/0 --layer4-configs=tcp:80 --target-secure-tags=TAGVALUE_WEB --description="migrated from allow-web"
gcloud compute network-firewall-policies rules create 1010 --firewall-policy=$POLICY --global-firewall-policy --direction=INGRESS --action=allow --src-ip-ranges=0.0.0.0/0 --layer4-configs=tcp:22 --target-secure-tags=TAGVALUE_SSH --description="migrated from allow-ssh"
gcloud compute network-firewall-policies rules create 2000 --firewall-policy=$POLICY --global-firewall-policy --direction=INGRESS --action=allow --src-ip-ranges=35.235.240.0/20 --layer4-configs=tcp:22 --description="allow IAP SSH for verification"

gcloud compute network-firewall-policies associations create --firewall-policy=$POLICY --network=$VPC --global-firewall-policy
gcloud compute networks update $VPC --network-firewall-policy-enforcement-order=BEFORE_CLASSIC_FIREWALL
gcloud compute networks describe $VPC --format="value(networkFirewallPolicyEnforcementOrder)"

for R in allow-iap-access allow-internal-vpc allow-ssh allow-web; do gcloud compute firewall-rules update $R --disabled --quiet; done
gcloud compute firewall-rules list --filter="network:$VPC" --format="table(name,disabled)"
for R in allow-iap-access allow-internal-vpc allow-ssh allow-web; do gcloud compute firewall-rules delete $R --quiet; done
```

Three notes. **The priorities must be unique** within a policy, unlike classic rules where all four legacy ones sat at 1000. **The IAP rule at 2000 is deliberate**: once the classic rules are gone the policy is all that governs, and task 2's verification needs to reach a VM with no external IP. And **disable then delete**, in that order, because the task asks for both and the disable pass gives you a listing to check before anything is destroyed.

**Deleting rules by name, not by filter.** `--filter="network:unified-vpc AND -targetTags:*"` is rejected outright by Compute's list filter grammar, which is a `gotchas.md` entry from GSP322 recurring here:

```
ERROR: Invalid value for field 'filter': ... Invalid list filter expression.
```

## Task 2

```
export PRIV_SUBNET=$(gcloud compute instances describe private-instance --zone=$ZONE --format="value(networkInterfaces[0].subnetwork.basename())")
gcloud compute routers nats describe unified-nat --router=unified-router --region=$REGION --format="yaml(name,sourceSubnetworkIpRangesToNat,subnetworks)"
gcloud compute routers nats update unified-nat --router=unified-router --region=$REGION --nat-custom-subnet-ip-ranges=$PRIV_SUBNET
gcloud compute routers nats describe unified-nat --router=unified-router --region=$REGION --format="yaml(name,sourceSubnetworkIpRangesToNat,subnetworks)"
gcloud compute ssh private-instance --zone=$ZONE --tunnel-through-iap --quiet --command="nslookup www.google.com; curl -s -o /dev/null -w 'egress HTTP %{http_code}\n' https://www.google.com"
```

The `describe` before the update is task 2's "identify the mismatched subnetwork mapping". Read it before overwriting it.

The gateway is already in `LIST_OF_SUBNETWORKS` mode, so `--nat-custom-subnet-ip-ranges` replaces the list rather than switching modes.

**The verification needs `--tunnel-through-iap`**, since `private-instance` has no external IP. If it times out, the policy is missing an IAP allow, which is why the rule at priority 2000 exists above.

## Task 3

```
export CVM_TAGS=compromised-vm
export APP_SUBNET=application-subnet

gcloud compute firewall-rules create containment-deny-http --network=unified-vpc --direction=INGRESS --priority=500 --action=DENY --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=$CVM_TAGS --description="Incident response: block all external HTTP ingress to compromised workload"

gcloud compute firewall-rules create allow-forensics-ssh --network=unified-vpc --direction=INGRESS --priority=400 --action=ALLOW --rules=tcp:22 --source-ranges=$BASTION_IP/32 --target-tags=$CVM_TAGS --description="Incident response: allow SSH from bastion external IP only"

gcloud compute networks subnets update $APP_SUBNET --region=$REGION --enable-flow-logs --logging-aggregation-interval=interval-5-sec --logging-flow-sampling=1.0 --logging-metadata=include-all

gcloud compute firewall-rules list --filter="name:(containment-deny-http OR allow-forensics-ssh)" --format="table(name,priority,denied[].map().firewall_rule().list(),allowed[].map().firewall_rule().list(),sourceRanges.list(),targetTags.list())"
gcloud compute networks subnets describe $APP_SUBNET --region=$REGION --format="yaml(name,logConfig)"
```

**`/32` on the bastion IP.** "Exclusively from your trusted bastion host's external IP address" is a single host, not a range.

**The application subnet is never named by the lab.** `application-subnet` is the one holding all three instances, which settles it, but it is an inference rather than a stated value.

## The containment is nominal, and that is worth understanding

Task 1 sets `BEFORE_CLASSIC_FIREWALL`. Task 3 then creates `containment-deny-http` as a **classic** rule. `compromised-vm` carries the `web` secure tag after the migration, and the policy allows tcp:80 to `web` at priority 1000.

So the policy is evaluated first, allows the traffic, and the classic deny never gets a say. The checkpoint reads the rule's existence rather than its effect, so this scores, but the workload is not actually contained. A real containment would put the deny in the **policy** at a priority below the allow, or remove the `web` tag binding from that instance.

Worth knowing, because the lab is teaching enforcement order in task 1 and then quietly contradicting it in task 3.

## Related files

- `gsp322-build-a-secure-network-challenge.md` for classic firewall rules, network tags, and the source of the Compute list filter limitation.
- `gsp528-ncc-challenge.md` and the three `gsp131x` NCC files for the wider networking family.
- `gsp373-protect-cloud-traffic-chrome-enterprise-premium-challenge.md` for IAP as the access control layer instead of firewall rules.
