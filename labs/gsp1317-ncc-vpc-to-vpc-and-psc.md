# GSP1317, Establish VPC to VPC Connectivity using NCC

Six tasks: create an NCC hub, attach two vpcs as spokes, verify the data path, set up Private Service Connect to a Cloud SQL instance, connect through it with psql, then clean up. Thirty five minutes, four scored checkpoints. Two vpcs and a Postgres instance with psc enabled are built for you.

Carries the "Do not deviate from instructions" warning, but the whole lab is already gcloud based so there is nothing to substitute.

## The real difficulty is value threading, not the commands

Four values have to flow from one command into the next: the subnet cidr, the internal ip you pick inside it, the service attachment uri, and the suggested dns record. The lab has you copy each by hand into a bracketed placeholder. Capturing them in shell variables instead removes the transcription risk entirely and is the same set of actions.

Also worth noting: the lab tells you to replace `[ADDRESS/RANGE]` in the dns record command "with the output of the previous command", but the previous command shown is the `dnsName` lookup, not the address. The A record points at the ip you reserved.

## Tasks 1 and 2

```
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
export SQL_INSTANCE=YOUR_SQL_INSTANCE

gcloud services enable networkconnectivity.googleapis.com

gcloud network-connectivity hubs create ncc-hub

gcloud config set accessibility/screen_reader false
gcloud compute networks subnets list --network=vpc1-ncc

gcloud network-connectivity spokes linked-vpc-network create vpc1-spoke1 \
  --hub=ncc-hub --vpc-network=vpc1-ncc \
  --exclude-export-ranges=10.1.2.0/24 --global

gcloud network-connectivity spokes linked-vpc-network create vpc2-spoke2 \
  --hub=ncc-hub --vpc-network=vpc2-ncc \
  --exclude-export-ranges=10.3.3.0/24 --global

gcloud network-connectivity hubs route-tables routes list --hub=ncc-hub --route_table=default
```

The point of `--exclude-export-ranges` is visible in the final route table: vpc1 has subnets `10.1.1.0/24`, `10.1.2.0/25` and `10.1.2.128/25`, and only `10.1.1.0/24` shows up in the hub table. The `/24` summary excludes both `/25` halves.

## Task 3, data path

Two ssh sessions. On `vm1-vpc1-ncc` run `sudo tcpdump -i any icmp -v -e -n`, then from `vm2-vpc2-ncc` run `ping 10.1.1.2` and watch the echoes arrive. Not scored.

## Task 4, Private Service Connect

```
export CIDR=$(gcloud compute networks subnets describe vpc2-ncc-subnet1 \
  --region=$REGION --project=$PROJECT_ID --format="value(ipCidrRange)")

gcloud compute instances list --format="table(name,networkInterfaces[0].networkIP)"

export PSC_IP=$(echo $CIDR | cut -d/ -f1 | awk -F. '{print $1"."$2"."$3".100"}')
echo $PSC_IP
```

Taking the `.100` host of the subnet avoids the collision the lab warns about. In our run the subnet was `10.2.2.0/24` with vms at `.2` and `.3`, so `10.2.2.100` was free. Check it against the instance list before reserving.

```
gcloud compute addresses create cloudsql-psc \
  --project=$PROJECT_ID --region=$REGION \
  --subnet=vpc2-ncc-subnet1 --addresses=$PSC_IP

export SA_URI=$(gcloud sql instances describe $SQL_INSTANCE \
  --project=$PROJECT_ID --format="value(pscServiceAttachmentLink)")

gcloud compute forwarding-rules create cloudsql-psc-ep \
  --address=cloudsql-psc --project=$PROJECT_ID --region=$REGION \
  --network=vpc2-ncc --target-service-attachment=$SA_URI \
  --allow-psc-global-access

gcloud compute forwarding-rules describe cloudsql-psc-ep \
  --project=$PROJECT_ID --region=$REGION \
  --format="value(pscConnectionStatus)"
```

That must print `ACCEPTED`. `PENDING` means wait and re-check.

```
gcloud dns managed-zones create cloudsql-dns \
  --project=$PROJECT_ID \
  --description="DNS zone for the Cloud SQL instances" \
  --dns-name=$REGION.sql.goog. \
  --networks=vpc2-ncc --visibility=private

export DNS_RECORD=$(gcloud sql instances describe $SQL_INSTANCE \
  --project=$PROJECT_ID --format="value(dnsName)")
echo $DNS_RECORD

gcloud dns record-sets create $DNS_RECORD \
  --project=$PROJECT_ID --type=A --ttl=300 \
  --rrdatas=$PSC_IP --zone=cloudsql-dns
```

The record looks like `ef8aabf64d6c.33ndfhng678bq.europe-west4.sql.goog.` and sits inside the `REGION.sql.goog.` zone.

## Task 5, psql through the endpoint

```
echo $DNS_RECORD

gcloud compute ssh --zone $ZONE "cloudsql-client" \
  --tunnel-through-iap --project $PROJECT_ID
```

The variables do not exist inside the vm, so echo the record first and paste the literal value in. Pasting the placeholder gets you `could not translate host name`.

```
psql "sslmode=disable dbname=postgres user=postgres host=THE_ECHOED_RECORD"
```

Password `changeme`. Then the lab's sql: create the `company` database, `\c company`, create the `employees` table, insert three rows, select them, `\q`, `exit`.

## Cleanup

Spokes, hub, dns record, dns zone. The forwarding rule and reserved address are left in the lab's own cleanup list, which is fine since nothing is scored after this.
