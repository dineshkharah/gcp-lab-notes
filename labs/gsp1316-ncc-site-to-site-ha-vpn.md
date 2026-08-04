# GSP1316, Establish Site to Site Connectivity with HA-VPN using NCC

Four tasks: create a Network Connectivity Center hub, define two vpn tunnel spokes, ping across, delete. Twenty minutes, one scored checkpoint. Everything is already built for you: two on prem vpcs, a routing vpc, two vms, firewall rules, ha vpn gateways, routers, and eight tunnels.

Carries the "Do not deviate from instructions" warning, and the whole thing is a single console wizard, so following the lab is the path of least friction here.

## Two regions, not one

The two offices sit in different regions. In our run office 1 was `us-east1` and office 2 was `europe-west1`. The spoke region has to match the office, so read both off the lab text rather than assuming a single region like most labs.

## Console path, as the lab writes it

Network Connectivity Center, Create hub, leave the hub policy mode default, name the hub, then Next step and Add a spoke twice.

For each spoke: type **VPN tunnel**, name `office-1-spoke` or `office-2-spoke`, the matching region, **Site-to-site data transfer On**, vpc network `routing-vpc`, then Add tunnel and pick the two tunnels for that region.

Site to site data transfer being On is what the checkpoint wants, and it is easy to skip past in the form.

## Pick the routing side tunnels

`gcloud compute vpn-tunnels list` shows eight tunnels, four pairs, both directions:

```
onprem-office1-to-routing-tunnel-0 / -1     us-east1
routing-to-onprem-office1-tunnel-0 / -1     us-east1
onprem-office2-to-routing-tunnel-0 / -1     europe-west1
routing-to-onprem-office2-tunnel-0 / -1     europe-west1
```

The spokes use the **`routing-to-onprem-*`** pairs, because the spoke's vpc network is `routing-vpc` and every tunnel in a spoke has to terminate in that same vpc. The `onprem-*-to-routing` tunnels terminate in the on prem vpcs and will not attach. In the console dropdown only the valid ones are offered, which hides the distinction; from the cli you have to know it.

## The same thing from the cli

Not needed, but useful for the challenge lab later:

```
gcloud services enable networkconnectivity.googleapis.com

gcloud network-connectivity hubs create ncc-hub

gcloud network-connectivity spokes linked-vpn-tunnels create office-1-spoke \
  --hub=ncc-hub \
  --region=REGION_OFFICE_1 \
  --vpn-tunnels=routing-to-onprem-office1-tunnel-0,routing-to-onprem-office1-tunnel-1 \
  --site-to-site-data-transfer

gcloud network-connectivity spokes linked-vpn-tunnels create office-2-spoke \
  --hub=ncc-hub \
  --region=REGION_OFFICE_2 \
  --vpn-tunnels=routing-to-onprem-office2-tunnel-0,routing-to-onprem-office2-tunnel-1 \
  --site-to-site-data-transfer

gcloud network-connectivity hubs list-spokes ncc-hub
```

`--site-to-site-data-transfer` is the flag form of the wizard toggle.

## Task 3, the ping

```
gcloud compute instances list --format="table(name,zone,networkInterfaces[0].networkIP)"
```

Ssh to `onprem-office1-vm`, ping office 2's internal ip, then the reverse. Not scored, but it is the proof the mesh is live.
