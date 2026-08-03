# GSP511, Build Google Cloud Infrastructure for AWS Professionals challenge lab

The Griffin team lab. Nine tasks: two vpcs, a dual nic bastion, Cloud SQL, a gke cluster, kubernetes secrets and volume, a WordPress deployment, an uptime check, and an editor grant. About twenty five to thirty minutes if the long waits overlap.

## Everything is doable from Cloud Shell

Including the two that usually push people to the console. `gcloud monitoring uptime create` handles the uptime check, so no Terraform is needed, and the sql statements go through `gcloud sql` subcommands so there is no interactive mysql client.

## Overlap plan

Cloud SQL is about ten minutes and gke about six, and they do not depend on each other. Launch both with `--async` in the first block, build the bastion and firewall rules while they provision.

## Block one, tasks 1, 2, 3 plus the async launches

```
export REGION=YOUR_REGION
export ZONE=YOUR_ZONE
export PROJECT_ID=$DEVSHELL_PROJECT_ID

gcloud compute networks create griffin-dev-vpc --subnet-mode=custom
gcloud compute networks subnets create griffin-dev-wp \
  --network=griffin-dev-vpc --region=$REGION --range=192.168.16.0/20
gcloud compute networks subnets create griffin-dev-mgmt \
  --network=griffin-dev-vpc --region=$REGION --range=192.168.32.0/20

gcloud compute networks create griffin-prod-vpc --subnet-mode=custom
gcloud compute networks subnets create griffin-prod-wp \
  --network=griffin-prod-vpc --region=$REGION --range=192.168.48.0/20
gcloud compute networks subnets create griffin-prod-mgmt \
  --network=griffin-prod-vpc --region=$REGION --range=192.168.64.0/20

gcloud sql instances create griffin-dev-db \
  --database-version=MYSQL_8_0 --tier=db-n1-standard-1 \
  --region=$REGION --root-password=stormwind_rules --async

gcloud container clusters create griffin-dev \
  --zone=$ZONE --machine-type=e2-standard-4 --num-nodes=2 \
  --network=griffin-dev-vpc --subnetwork=griffin-dev-wp --async

gcloud compute instances create griffin-bastion \
  --zone=$ZONE --machine-type=e2-medium \
  --network-interface=network=griffin-dev-vpc,subnet=griffin-dev-mgmt \
  --network-interface=network=griffin-prod-vpc,subnet=griffin-prod-mgmt \
  --tags=ssh

gcloud compute firewall-rules create fw-ssh-dev \
  --network=griffin-dev-vpc --allow=tcp:22 --source-ranges=0.0.0.0/0 --target-tags=ssh
gcloud compute firewall-rules create fw-ssh-prod \
  --network=griffin-prod-vpc --allow=tcp:22 --source-ranges=0.0.0.0/0 --target-tags=ssh
```

The prod vpc is built by hand here. Community scripts build it with Deployment Manager using a config from an older lab id, and the ranges do not match what this lab asks for.

The two ssh firewall rules are what makes "make sure you can ssh to the host" pass.

## Filling the wait, tasks 9 and part of 6

The editor grant and the file download have no dependencies:

```
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:SECOND_USER_EMAIL --role=roles/editor

gcloud storage cp -r gs://spls/gsp511/wp-k8s .
cd wp-k8s
sed -i 's/username_goes_here/wp_user/; s/password_goes_here/stormwind_rules/' wp-env.yaml
cat wp-env.yaml
```

Note the bucket is `gs://spls/gsp511/wp-k8s`. Community scripts point at `gs://cloud-training/gsp321/wp-k8s`, which is a different lab.

## Block two, tasks 4 and 6, once both show ready

```
gcloud sql databases create wordpress --instance=griffin-dev-db
gcloud sql users create wp_user --instance=griffin-dev-db --host=% --password=stormwind_rules

gcloud container clusters get-credentials griffin-dev --zone=$ZONE
kubectl apply -f wp-env.yaml

gcloud iam service-accounts keys create key.json \
  --iam-account=cloud-sql-proxy@$PROJECT_ID.iam.gserviceaccount.com
kubectl create secret generic cloudsql-instance-credentials --from-file key.json
```

The `GRANT ALL PRIVILEGES` statement in the lab text is not needed. Cloud SQL gives users created through the api a broad privilege set already.

Wait for both with:

```
until [[ $(gcloud sql instances describe griffin-dev-db --format='value(state)') == "RUNNABLE" && \
         $(gcloud container clusters describe griffin-dev --zone=$ZONE --format='value(status)') == "RUNNING" ]]; do
  sleep 20
done
echo "BOTH READY"
```

## Block three, tasks 7 and 8

```
export INSTANCE_CN=$(gcloud sql instances describe griffin-dev-db --format='value(connectionName)')
sed -i "s/YOUR_SQL_INSTANCE/$INSTANCE_CN/g" wp-deployment.yaml
grep instances= wp-deployment.yaml

kubectl apply -f wp-deployment.yaml
kubectl apply -f wp-service.yaml

until [[ -n $(kubectl get svc wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}') ]]; do sleep 15; done
export WP_IP=$(kubectl get svc wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://$WP_IP"

gcloud monitoring uptime create griffin-dev-uptime \
  --resource-type=uptime-url \
  --resource-labels=host=$WP_IP,project_id=$PROJECT_ID \
  --protocol=http --port=80
```

Pod should reach `2/2 Running`. The load balancer takes one to three minutes to get an ip.
