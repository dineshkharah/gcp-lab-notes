# GSP694, gcloud for Network Configuration

Eight tasks, **two** checkpoints, ten minutes stated and **no cost**. About **two minutes**. Entirely the lab's own terminal.

Second of the five required labs in the **Essential Google Cloud CLI Tools** course. See `gsp693-gcloud-cli-beginners-guide.md` for the first.

Manual last updated and last tested June 2026.

## Six of the eight tasks are read only

Only **task 4** and **task 6** are scored, and each is a single `firewall-rules create`. Tasks 1, 2, 3, 5 and 7 are `list` and `describe`; task 8 is a connectivity test with no checkpoint.

Everything is pre created: networks `labnet` and `privatenet`, subnets `labnet-sub` and `private-sub`, and VMs `lnet-vm` and `pnet-vm`. Nothing is randomised except the project.

## The two scored commands

```
gcloud compute firewall-rules create labnet-allow-internal --network=labnet --action=ALLOW --rules=icmp,tcp:22 --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create privatenet-deny --network=privatenet --action=DENY --rules=icmp,tcp:22 --source-ranges=0.0.0.0/0
```

Click a checkpoint after each. The rest of the lab:

```
gcloud compute networks list
gcloud compute networks describe labnet
gcloud compute networks describe privatenet
gcloud compute networks subnets list
gcloud compute networks subnets describe labnet-sub --region=us-east1
gcloud compute networks subnets describe private-sub --region=us-east1
gcloud compute firewall-rules describe labnet-allow-internal
gcloud compute firewall-rules list --sort-by=NETWORK
gcloud compute instances list
```

## The lab's own instruction for --sort-by is wrong

Task 6 ends with:

> `gcloud compute firewall-rules list --sort-by=NETWORK_NAME` — *"Replace NETWORK_NAME with the name of a pre-created custom network."*

**`--sort-by` takes a field name, not a value.** Substituting `labnet` there gives an invalid sort field. The field is `NETWORK`. Follow the command as printed and ignore the instruction to substitute, which is the reverse of the usual advice about placeholders.

## The deny rule is redundant in effect, and that is the lesson

Worth understanding rather than just running, because it looks like the rule is what blocks the traffic.

The lab states it in task 4 and then does not connect the dots: *"Auto networks include default rules, custom networks do not include any firewall rules."* Both `labnet` and `privatenet` are **CUSTOM** mode, so `privatenet` already denies all ingress before you touch it. The `privatenet-deny` rule changes nothing observable; the ping to `pnet-vm` would fail identically without it.

So the lab is teaching you to write an **explicit** deny, not to close a hole. That distinction matters on a real network, because an explicit deny at priority 1000 will also override any lower priority allow added later, which an absent rule would not. GSP399's `containment-deny-http` is the case where that priority interaction decides whether containment actually works.

## Task 8, if you run it

Not required, and unscored.

```
export LNET_IP=$(gcloud compute instances list --filter=name:lnet-vm --format='value(EXTERNAL_IP)')
export PNET_IP=$(gcloud compute instances list --filter=name:pnet-vm --format='value(EXTERNAL_IP)')
ping -c 3 -W 2 $LNET_IP
ping -c 3 -W 2 $PNET_IP
```

**`-W 2`** bounds the wait per packet. The lab's own version has you Ctrl-C out of the second ping, which is the same hang GSP693 produces with `curl`: 100% packet loss and no response at all, because a GCP firewall **drops** rather than rejects. See the refused versus timeout section in `gotchas.md`.

## Related files

- `gsp693-gcloud-cli-beginners-guide.md`, the previous lab in the course, and the reference case for drop versus refuse.
- `gsp322-build-a-secure-network-challenge.md` for the same commands as a challenge, at larger scale, plus the filter expression the Compute list api rejects.
- `gsp399-design-implement-network-security-challenge.md` for where explicit deny rules and priorities start to matter, and for migrating all of this to Cloud NGFW.
