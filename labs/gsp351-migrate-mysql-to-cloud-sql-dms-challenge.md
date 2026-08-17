# GSP351, Migrate MySQL Data to Cloud SQL Using Database Migration Service: Challenge Lab

Five tasks, five checkpoints, fifty five minutes stated and **1 credit**. About **50 minutes**, and for once the stated time is close to honest, with very little slack.

**Tasks 2 and 3 are console. Tasks 1, 4 and 5 are Cloud Shell**, along with the row count verification for task 2. Four of five tasks are touched from the terminal but the two longest are clicked.

Manual last updated and last tested August 2026.

## Why tasks 2 and 3 stay in the console

`gcloud database-migration` exists and covers connection profiles, migration jobs and the promote. It is still the wrong tool for the two migration jobs here.

The lab requires **Existing instance** as the destination type for both jobs. The CLI wants a **destination connection profile**, and `connection-profiles create cloudsql` **provisions a new Cloud SQL instance** rather than pointing at one that already exists. So from the CLI you would be reverse engineering what profile shape the console produces for an existing instance, and then getting the VPC peering handshake right blind, against a clock with no slack.

The console also gives you a **Test connection** button before you commit, which is worth more here than anywhere else in these notes: a bad source profile shows up as a failed test rather than as a migration that starts and stalls.

GSP355 is the PostgreSQL twin of this lab and uses the same DMS wizard, so the same reasoning applies there.

## Both destination instances already exist

The lab says **Existing instance** twice, and names them: `mysql-pln-jux` for the one time job and `mysql-pln-jux-cont` for the continuous one. That is what makes 55 minutes feasible at all, since you skip two Cloud SQL instance creations at roughly ten minutes each.

The dataset is also tiny, 5030 rows, so neither migration spends long moving data. The clock goes on job configuration and on the VPC peering setup.

## Task 1, the connection profile, with the IP derived rather than typed

```
export REGION=us-east4
export ZONE=us-east4-a
export PROJECT=$(gcloud config get-value project)
export SOURCE_VM=dev-pln-jux
export PROFILE_ID=mysql-source-profile
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
gcloud services enable datamigration.googleapis.com servicenetworking.googleapis.com

export SOURCE_IP=$(gcloud compute instances describe $SOURCE_VM --zone=$ZONE --format='value(networkInterfaces[0].accessConfigs[0].natIP)')
echo "source external ip=[$SOURCE_IP]"

gcloud database-migration connection-profiles create mysql $PROFILE_ID \
  --region=$REGION --display-name=$PROFILE_ID \
  --host=$SOURCE_IP --port=3306 --username=admin --password=changeme

gcloud database-migration connection-profiles list --region=$REGION
```

**`servicenetworking.googleapis.com` matters**, not just `datamigration`. Task 3's VPC peering needs it, and enabling both now avoids a stall halfway through the wizard.

The IP is derived. The circulating script prompts you to type it, which is an avoidable transcription error in the single field that decides whether the profile connects at all.

## Tasks 2 and 3 in the console

Database Migration > Migration jobs, twice, **reusing the same source profile** from task 1.

- **Task 2**: job type **One-time**, destination **Existing instance** `mysql-pln-jux`, connectivity via the source's **external IP**.
- **Task 3**: job type **Continuous**, destination **Existing instance** `mysql-pln-jux-cont`, connectivity **VPC peering**.

**Task 3's checkpoint wants the job in `Running`**, not merely created. The lab states it: *"Wait until the job is in the Running state before checking your progress."* A continuous job passes through a startup phase first, so clicking early fails.

Verify task 2 with the row count the lab gives you:

```
gcloud sql connect mysql-pln-jux --user=admin --quiet <<'EOF'
use customers_data;
select count(*) from customers;
EOF
```

Password `changeme`. DMS migrates the source's users, so `admin` works on the destination. Expect **5030**.

## Task 4, the one task that can half work silently

The update runs **on the source Compute Engine instance**, which needs an SSH the circulating guide omits entirely:

```
gcloud compute ssh $SOURCE_VM --zone=$ZONE --quiet --command="mysql -u admin -pchangeme -e \"use customers_data; update customers set gender='FEMALE' where addressKey=934; select addressKey, gender from customers where addressKey=934;\""
```

Then, after about a minute, the same read against the **continuous** destination:

```
gcloud sql connect mysql-pln-jux-cont --user=admin --quiet <<'EOF'
use customers_data;
select addressKey, gender from customers where addressKey = 934;
EOF
```

`FEMALE` there is the proof replication is live. **If it still shows the old value, that is a task 3 problem surfacing late**, not a task 4 problem: the job was created but is not actually replicating.

Note the quoting: double quotes outside the `--command` so `$ZONE` expands, escaped double quotes around the SQL, and no shell metacharacters in the password.

## Task 5, the promote

```
gcloud database-migration migration-jobs promote mysql-pln-jux-cont --region=$REGION
gcloud database-migration migration-jobs describe mysql-pln-jux-cont --region=$REGION --format="value(state,phase)"
```

One command rather than hunting the Promote button, and the `describe` confirms it rather than assuming. The lab's hint to name the migration job the same as its destination instance is worth taking, because it makes this command predictable.

## One oddity in the scoring panel

Tasks 1, 2, 3 and 5 rendered `Assessment Completed!` around their checkpoint labels. **Task 4 rendered its hint text instead**, *"Please perform continuous migration from the stand alone MySQL database to Cloud SQL"*, while the lab still totalled **100 of 100**. So task 4 appears to carry no points of its own, or to share task 3's. Worth knowing so that seeing a hint where the others say completed does not send you rechecking work that scored.

## The community script

It only claims task 1, and does that part competently: it checks whether the profile already exists, so it is idempotent, and it actually tests `$?` after the create. Three problems.

- **It prompts you to type the source external IP.** Derive it.
- **`REGION=${REGION:-us-east4}`.** A hardcoded default that happened to match this run's region exactly, which is the most dangerous kind: press Enter at that prompt and it silently works today and silently builds the profile in the wrong region next time, where no migration job in the lab's region can select it.
- **Its task 4 instruction is `mysql -u admin -p` presented as a Cloud Shell command.** There is no MySQL server in Cloud Shell. That has to run on the source instance, and the SSH step is missing, so following it literally gives a connection refused that reads like a broken database.

It also enables `servicenetworking`, which is more than most of these guides get right.

## Related files

- `gsp306-migrate-mysql-to-cloud-sql-challenge.md`, the same migration done by hand with `mysqldump` and no DMS. Shorter, and its checkpoint 3 scores the authorized network rather than the data.
- `gsp1048-spanner-database-fundamentals.md` and `gsp1049-spanner-loading-and-backups.md` for the other managed database labs.
