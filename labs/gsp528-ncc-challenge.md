# GSP528, Connecting Cloud Networks with NCC challenge lab

Three tasks: connect two on prem vpcs, connect two workload vpcs, then connect one on prem vpc to one workload vpc. Twenty five minutes, three checkpoints. Everything is prebuilt: five vpcs, five vms, and eight vpn tunnels.

Nothing in the task text gives you names or a region, so discover the environment first.

## Discovery block

```
gcloud services enable networkconnectivity.googleapis.com

gcloud compute networks list
gcloud compute networks subnets list --format="table(name,region,network,ipCidrRange)"
gcloud compute vpn-tunnels list --format="table(name,region,status)"
gcloud compute instances list --format="table(name,zone,networkInterfaces[0].network.basename(),networkInterfaces[0].networkIP)"
```

In our run everything was in `us-east1` with:

| vpc | subnet | vm | ip |
| --- | --- | --- | --- |
| on-prem-office-1-vpc | 10.1.0.0/24 | onprem-office1-vm | 10.1.0.2 |
| on-prem-office-2-vpc | 10.2.0.0/24 | onprem-office2-vm | 10.2.0.2 |
| routing-vpc | 10.11.0.0/24 | | |
| workload-vpc-1 | 10.10.0.0/24 | workload1-vm | 10.10.0.2 |
| workload-vpc-2 | 10.20.0.0/24 | workload2-vm | 10.20.0.2 |

## The naming overlap that saves a rebuild

This is the thing worth knowing before starting.

Task 2 says Workload VPC 1's spoke must **include** `workload-1`. Task 3 says Workload VPC 1's spoke must **include** `hybrid`. A vpc can only be a vpc spoke on one hub, and cannot be attached twice to the same hub, so on the face of it task 3 forces you to detach and recreate what task 2 built.

It does not, because the requirements are substring matches. One spoke named **`hybrid-workload-1-spoke`** contains both `workload-1` and `hybrid`, so it satisfies task 2 and task 3 at once. No detach, no walk back, and no second hub.

Five spokes on one hub covers all three tasks:

| spoke | type | serves |
| --- | --- | --- |
| office-1-spoke | vpn tunnels, routing vpc side, site to site on | task 1 |
| office-2-spoke | vpn tunnels, routing vpc side, site to site on | task 1 |
| hybrid-workload-1-spoke | vpc network, workload-vpc-1 | tasks 2 and 3 |
| workload-2-spoke | vpc network, workload-vpc-2 | task 2 |
| hybrid-office-1-spoke | vpc network, on-prem-office-1-vpc | task 3 |

## The commands

```
export REGION=us-east1

gcloud network-connectivity hubs create ncc-hub

gcloud network-connectivity spokes linked-vpn-tunnels create office-1-spoke \
  --hub=ncc-hub --region=$REGION \
  --vpn-tunnels=routing-to-onprem-office1-tunnel-0,routing-to-onprem-office1-tunnel-1 \
  --site-to-site-data-transfer

gcloud network-connectivity spokes linked-vpn-tunnels create office-2-spoke \
  --hub=ncc-hub --region=$REGION \
  --vpn-tunnels=routing-to-onprem-office2-tunnel-0,routing-to-onprem-office2-tunnel-1 \
  --site-to-site-data-transfer

gcloud network-connectivity spokes linked-vpc-network create hybrid-workload-1-spoke \
  --hub=ncc-hub --vpc-network=workload-vpc-1 --global

gcloud network-connectivity spokes linked-vpc-network create workload-2-spoke \
  --hub=ncc-hub --vpc-network=workload-vpc-2 --global

gcloud network-connectivity spokes linked-vpc-network create hybrid-office-1-spoke \
  --hub=ncc-hub --vpc-network=on-prem-office-1-vpc --global

gcloud network-connectivity hubs list-spokes ncc-hub
```

Use the `routing-to-onprem-*` tunnels, not the `onprem-*-to-routing` ones. Every tunnel in a spoke has to terminate in the same vpc, and the spoke sits in `routing-vpc`.

`--site-to-site-data-transfer` is required on the vpn spokes. Task 3's two spokes are both **vpc network** type, including the on prem one, which is easy to misread as another pair of tunnel spokes.

Attaching `on-prem-office-1-vpc` as a vpc spoke to the same hub that already carries its tunnels as a vpn spoke worked without complaint, so one hub really does cover everything.

## Connectivity tests

One per task, each from an ssh session:

- task 1: from `onprem-office1-vm`, `ping -c 5 10.2.0.2`
- task 2: from `workload1-vm`, `ping -c 5 10.20.0.2`
- task 3: from `onprem-office1-vm`, `ping -c 5 10.10.0.2`
