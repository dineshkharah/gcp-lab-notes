# GSP1318, Establish Hybrid Network Connectivity with NCC

Five tasks: build an ha vpn between an on prem vpc and a routing vpc, create an NCC hub, attach a vpc spoke and a hybrid vpn spoke, verify the data path, clean up. Thirty minutes, four scored checkpoints, all gcloud.

The routing vpc, on prem vpc, workload vpc and vms are prebuilt. Carries the "Do not deviate from instructions" warning, and since the lab is already gcloud there is nothing worth substituting.

## Structure

The lab is long but entirely copy paste. The useful thing is grouping it into four pastes that end on the four checkpoint boundaries:

1. two cloud routers, two vpn gateways, two vpn tunnels
2. bgp interfaces and peers, route advertisements, the med value, status checks
3. the hub
4. the vpc spoke and the hybrid spoke

## Keep block one as a single paste

`secret_key=$(openssl rand -base64 24)` is generated once and both tunnels must use the same value. Split across pastes in different shells and the second tunnel gets a different secret and the tunnel never establishes.

The same applies more broadly: the whole lab is driven by shell variables like `${region}` and `${routing_vpc_tunnel_name}`, so it all has to run in one Cloud Shell session. A restart loses them and later blocks fail with empty arguments rather than an obvious error.

## What the pieces are doing

Two cloud routers with different asns, 64525 in the routing vpc and 64526 on prem. A vpn gateway each side, then a tunnel from each side pointing at the other with `--peer-gcp-gateway`.

Then bgp: an interface on each router with a link local address, `169.254.1.1` on the routing side and `169.254.1.2` on prem, each with `--mask-length=30`, and a bgp peer on each pointing at the other's address with the other's asn.

Two advertisement changes matter:

- The routing vpc router is set to custom advertisement mode with `--set-advertisement-ranges` covering the vpc spoke subnet, because **NCC hub subnets are not announced to hybrid spokes by default**. Without this the on prem side never learns the workload subnet.
- The on prem bgp peer gets `--advertised-route-priority="111"`, which is the bgp med value the lab has you observe later.

## The med value is the lesson

After the spokes exist:

```
gcloud network-connectivity hubs route-tables routes list \
  --hub=mesh-hub --route_table=default \
  --effective-location=REGION --filter=10.0.3.0/24
```

shows priority `111`. The point is that cloud router learned prefixes keep their med values as they propagate across NCC spokes through dynamic route exchange.

## The hybrid spoke has no site to site flag

```
gcloud network-connectivity spokes linked-vpn-tunnels create hybrid-spoke \
  --region="${region}" --hub="${hub_name}" \
  --vpn-tunnels="${routing_vpc_tunnel_name}"
```

No `--site-to-site-data-transfer` here, unlike GSP1316 and the GSP528 challenge lab. This lab relies on dynamic route exchange with a vpc spoke instead of site to site transfer between two hybrid spokes. Worth keeping straight, since the same subcommand is used in both cases with a different flag set.

Also note the hybrid spoke attaches only **one** tunnel, the routing vpc side one. GSP1316 attaches a pair.

## Checkpoint three expects an empty route table

Task 2 has you list the hub's default route table before any spoke exists and the correct result is nothing. Easy to read as a failure.

## Task 4, data path

```
gcloud compute ssh vm3-onprem --zone=ZONE
curl 10.0.1.2 -v
```

Not scored, but it is the proof that on prem reaches the workload vpc through the hub.

## Cleanup order

Spokes, then hub, then tunnels, then routers. Give it its own paste after all four checkpoints are green, since it removes everything they look at.
