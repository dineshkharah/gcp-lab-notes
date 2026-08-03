# AlloyDB, Create and Manage Instances challenge lab

Five tasks: cluster and primary instance, tables, data, read pool, backup. About twenty five minutes if the slow parts overlap, closer to forty five in strict order.

gcloud beats the console here even though the console is allowed, because cluster and instance creation is a ten minute wait either way.

## Timing plan

Cluster is two to three minutes, primary instance five to eight, read pool five to eight, backup two to four. Launch the read pool with `--async` right after task 1 passes and do the psql work while it builds.

## Tasks 1 and 4

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID

gcloud beta alloydb clusters create lab-cluster \
  --project=$PROJECT_ID --region=$REGION \
  --password=Change3Me --network=peering-network

gcloud beta alloydb instances create lab-instance \
  --project=$PROJECT_ID --region=$REGION --cluster=lab-cluster \
  --instance-type=PRIMARY --cpu-count=2

gcloud beta alloydb instances create lab-instance-rp1 \
  --project=$PROJECT_ID --region=$REGION --cluster=lab-cluster \
  --instance-type=READ_POOL --cpu-count=2 --read-pool-node-count=2 --async
```

Private ip for the next task:

```
gcloud beta alloydb instances describe lab-instance \
  --cluster=lab-cluster --region=$REGION --format="value(ipAddress)"
```

## Tasks 2 and 3, tables and data

Ssh to the `alloydb-client` vm, then one block. No interactive psql needed.

```
export ALLOYDB=PRIVATE_IP
echo $ALLOYDB > alloydbip.txt
export PGPASSWORD=Change3Me

psql -h $ALLOYDB -U postgres <<'EOF'
CREATE TABLE regions (region_id bigint NOT NULL, region_name varchar(25));
ALTER TABLE regions ADD PRIMARY KEY (region_id);
CREATE TABLE countries (country_id char(2) NOT NULL, country_name varchar(40), region_id bigint);
ALTER TABLE countries ADD PRIMARY KEY (country_id);
CREATE TABLE departments (department_id smallint NOT NULL, department_name varchar(30), manager_id integer, location_id smallint);
ALTER TABLE departments ADD PRIMARY KEY (department_id);
EOF
```

Then the inserts from the lab text in the same style. Expect counts of four regions, nine countries, six departments.

## Task 5, backup

```
gcloud beta alloydb backups create lab-backup \
  --project=$PROJECT_ID --region=$REGION --cluster=lab-cluster
```

## What went wrong

The whole run stalled on a wrong project id typed from a truncated screenshot. Every call returned `PERMISSION_DENIED` or `CONSUMER_INVALID` and it looked like a disabled api. `gcloud services enable` then failed with `RESOURCES_NOT_FOUND`. The fix was restarting Cloud Shell and using `$DEVSHELL_PROJECT_ID`. Also `gcloud auth login` had been run, which was unnecessary and added noise.

The `export REGION=${ZONE%-*}` line in the community script depends on `$ZONE`, which Cloud Shell does not set. Use the region printed on the lab page.
