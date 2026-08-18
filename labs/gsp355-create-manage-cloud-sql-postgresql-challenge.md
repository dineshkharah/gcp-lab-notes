# GSP355, Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab

Four tasks, **five** checkpoints, one hour stated and **1 credit**. Took **two lab instances**: the first was abandoned unrecoverable, the second finished at 100/100 with about fifty minutes to spare. **Read this whole file before starting it again**; the lab as written cannot succeed on the current PostgreSQL build.

Manual and last tested both August 2026.

Split: the source prep is SSH on the VM, the migration is console, tasks 3 and 4 are Cloud Shell.

## The blocker: PostgreSQL now refuses pglogical's output plugin

Every migration job fails with this, three attempts in a row, on two lab instances:

```
finished replication setup with errors: failed to set up logical replication:
speckle::ERROR_RDBMS: failed to sync all databases.
please check postgres logs for errors and retry migration
```

The cause is not in that message, and not in the `ERROR` lines of the postgres log either. It is in the **`DETAIL` and `HINT`** lines underneath:

```
ERROR:   library "pglogical_output" may not be used as an output plugin
DETAIL:  The configuration parameter "output_plugin_libraries"
         (currently 'pgoutput, test_decoding') does not name this library
         as a trusted output plugin.
HINT:    If it is safe for all REPLICATION users to use this library as an
         output plugin, add it to "output_plugin_libraries" and reload.
STATEMENT: CREATE_REPLICATION_SLOT "pgl_orders_..." LOGICAL pglogical_output
```

**PostgreSQL 14.23 ships an allowlist of trusted logical decoding plugins**, defaulting to `pgoutput` and `test_decoding`. DMS asks for `pglogical_output`, which is not on it, so the slot is never created. This is recent hardening and the lab has not caught up.

The whole fix:

```
sudo bash -c "cat >> /etc/postgresql/14/main/postgresql.conf <<'EOF'
output_plugin_libraries = 'pgoutput, test_decoding, pglogical_output, pglogical'
EOF"
sudo systemctl restart postgresql
sudo -u postgres psql -Atc "SHOW output_plugin_libraries"
```

**Add this during source prep**, before the first job attempt. Every minute spent elsewhere is wasted until it is in place.

**A dead end worth not repeating.** `pglogical_output.so` is a 14K stub next to a 260K `pglogical.so`, which makes it look like the wrong file is being loaded. Copying `pglogical.so` over it changes nothing, because the refusal is the allowlist and not a missing symbol. Restore the original if you try it.

## Read DETAIL and HINT, not just ERROR

The reason this took three attempts is that the diagnosis used:

```
sudo grep -iE "error|fatal|panic|denied" /var/log/postgresql/postgresql-14-main.log
```

`DETAIL` and `HINT` are **separate log lines** and contain neither word, so the grep printed the symptom and hid the answer twice. A plain `tail -30` showed it immediately. Now a section in `gotchas.md`.

Also worth knowing which errors in that log are noise. **These are DMS capability probes and always appear:**

```
ERROR: function aurora_version() does not exist
ERROR: role "cloudsqladmin" does not exist
FATAL: unsupported frontend protocol 0.0
FATAL: no PostgreSQL user name specified in startup packet
```

I wrote off the `pglogical_output` line as another probe because it sat between two of these. It is not; correlate timestamps against the DMS log instead of judging by neighbours. The DMS side names the real failure plainly:

```
textPayload: "Failed to add [databaseName=orders] to migration."
```

Find it with Logs Explorer on `resource.type="cloudsql_database"` and `logName` ending `replication-setup.log`.

## Ownership, not just grants, for checkpoint 1

Checkpoint 1 stayed at 0/20 with `pglogical` installed in both databases, all five primary keys present, the user created with `REPLICATION`, and every documented grant applied. Its message is unhelpfully generic:

> Please ensure you have created a migration user, prepared the source database for migration & granted all permissions.

**Two things were missing.**

**`listen_addresses`.** Debian defaults to `localhost`, so nothing outside the VM can connect at all:

```
sudo bash -c "cat >> /etc/postgresql/14/main/postgresql.conf <<'EOF'
listen_addresses = '*'
EOF"
```

**Ownership transfer.** This is what finally turned it green, and no amount of `GRANT` substitutes for it:

```
sudo -u postgres psql -c "ALTER DATABASE orders OWNER TO migration_admin;"
for t in distribution_centers inventory_items order_items products users; do sudo -u postgres psql -d orders -c "ALTER TABLE public.$t OWNER TO migration_admin;"; done
```

It matters twice over: the checkpoint reads it, and pglogical needs it to add tables to a replication set. Once ownership was set the log started showing `adding table public.order_items to replication set ...` where before there was nothing.

## The password breaks a pasted command

The migration password is **`DMS_1s_cool!`** and the `!` triggers bash history expansion in an interactive shell:

```
-bash: !': event not found
ERROR:  role "migration_admin" does not exist
```

The `CREATE USER` never reached psql, and the three following grants failed on the missing role. **Run `set +H` first**, or the same block silently prepares nothing.

## Never delete a failed migration job

**This is what cost the first lab instance.** The job failed, so it was deleted and recreated, which is the intuitive recovery and is wrong here twice over.

**Checkpoint 2 reads the job, not the data.** Its message is *"Please perform continuous migration from the stand alone PostgreSQL database to Cloud SQL"*, and it stayed at 0/20 even though the destination already held a fully replicated `orders`. Deleting the job deleted the only thing being scored.

**And the destination cannot be reused.** A failed job leaves the instance as a `READ_REPLICA_INSTANCE` of a source representation instance named `INSTANCE-master`, and DMS will not adopt an existing replica, so it vanishes from the Existing instance picker. Recovering it took four steps:

```
gcloud sql instances promote-replica INSTANCE --quiet
```
then dropping the migrated database, which `gcloud` refuses because the migration left it owned by the migration user:

```
ERROR: failed to delete database "orders". Detail: pq: must be owner of database orders.
       (Please use psql client to delete database that is not owned by "cloudsqlsuperuser")
```

so via `gcloud sql connect ... --user=postgres` and `DROP DATABASE orders;`, then clearing the destination's `postgres` database, because **DMS's emptiness test looks inside it** and the previous run left the extension there:

```
The destination instance contains existing data or user defined entities.
You can only migrate to empty instances.
```
```
DROP EXTENSION IF EXISTS pglogical CASCADE;
DROP SCHEMA IF EXISTS pglogical CASCADE;
```

Deleting the instance instead is not an option: the lab hands you a fixed instance id and Cloud SQL will not let you reuse a name for days.

**Use Restart on the job page instead**, which wipes the destination and redoes the full dump. Leave the validation checkbox unchecked so config problems are reported.

## Choose Specific databases

At *Configure migration databases*, pick **Specific databases** and tick **`orders`** only. The task says to migrate the `orders` database, and the failure message is literally *"failed to sync all databases"*, so there is no reason to include `postgres`, which has no user tables and no ownership prepared.

## The working order

**SSH**, source prep, all of it before any job attempt:

```
export PGV=$(ls /etc/postgresql | head -1)
sudo apt-get update -qq && sudo apt-get install -y postgresql-$PGV-pglogical
sudo bash -c "cat >> /etc/postgresql/$PGV/main/postgresql.conf <<'EOF'
listen_addresses = '*'
wal_level = logical
max_worker_processes = 10
max_replication_slots = 10
max_wal_senders = 10
shared_preload_libraries = 'pglogical'
output_plugin_libraries = 'pgoutput, test_decoding, pglogical_output, pglogical'
EOF"
sudo bash -c "echo 'host all all 0.0.0.0/0 md5' >> /etc/postgresql/$PGV/main/pg_hba.conf"
sudo systemctl restart postgresql
sleep 5
set +H
for db in postgres orders; do sudo -u postgres psql -d $db -c "CREATE EXTENSION IF NOT EXISTS pglogical;"; done
sudo -u postgres psql -c "CREATE USER MIGRATION_USER PASSWORD 'DMS_1s_cool!';"
sudo -u postgres psql -c "ALTER USER MIGRATION_USER WITH REPLICATION SUPERUSER;"
sudo -u postgres psql -c "ALTER DATABASE orders OWNER TO MIGRATION_USER;"
for db in postgres orders; do sudo -u postgres psql -d $db -c "GRANT USAGE ON SCHEMA pglogical TO MIGRATION_USER; GRANT ALL ON SCHEMA pglogical TO MIGRATION_USER; GRANT SELECT ON ALL TABLES IN SCHEMA pglogical TO MIGRATION_USER; GRANT SELECT ON ALL SEQUENCES IN SCHEMA pglogical TO MIGRATION_USER; GRANT USAGE ON SCHEMA public TO MIGRATION_USER; GRANT ALL ON SCHEMA public TO MIGRATION_USER; GRANT SELECT ON ALL TABLES IN SCHEMA public TO MIGRATION_USER; GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO MIGRATION_USER;"; done
for t in distribution_centers inventory_items order_items products users; do sudo -u postgres psql -d orders -c "ALTER TABLE public.$t ADD PRIMARY KEY (id);" || echo "PK already on $t"; sudo -u postgres psql -d orders -c "ALTER TABLE public.$t OWNER TO MIGRATION_USER;"; done
sudo -u postgres psql -Atc "SHOW listen_addresses; SHOW wal_level; SHOW output_plugin_libraries;"
sudo -u postgres psql -Atc "SELECT rolname, rolsuper, rolreplication FROM pg_roles WHERE rolname='MIGRATION_USER'"
sudo -u postgres psql -d orders -Atc "SELECT tablename, tableowner FROM pg_tables WHERE schemaname='public'"
```

**Four tables already have primary keys** and only `inventory_items` needs one, so the `|| echo` is expected to fire four times rather than being a problem.

Then console: connection profile with the VM's **internal** IP, port 5432, region `europe-west1`. Job: Continuous, Existing instance, VPC peering on `default`, Specific databases with `orders`. Test, Create & start. Wait for `Running / CDC`, checkpoint 2. Then **Promote on the job page**, not `gcloud sql instances promote-replica`, so the job status becomes `Completed`, which is what checkpoint 3 reads.

**Cloud Shell**, task 3:

```
gcloud sql instances patch INSTANCE --authorized-networks=VM_EXTERNAL_IP --quiet
gcloud sql instances patch INSTANCE --database-flags=cloudsql.iam_authentication=on --quiet
for i in $(seq 1 20); do S=$(gcloud sql instances describe INSTANCE --format='value(state)'); echo "state=$S"; [ "$S" = "RUNNABLE" ] && break; sleep 15; done
gcloud sql users create STUDENT_EMAIL --instance=INSTANCE --type=cloud_iam_user
gcloud sql connect INSTANCE --user=postgres --database=orders --quiet
```

The IAM flag **restarts the instance**, so the user creation has to wait for `RUNNABLE`. Then, password `supersecret!`:

```
GRANT SELECT ON TABLE_NAME TO "STUDENT_EMAIL";
```

Quotes are required around the email. `\dp TABLE_NAME` confirms `=r/cloudsqlexternalsync` against the student address.

**Cloud Shell**, task 4:

```
gcloud sql instances patch INSTANCE --backup-start-time=00:00 --enable-point-in-time-recovery --retained-transaction-log-days=DAYS --quiet
gcloud sql backups create --instance=INSTANCE
export TS=$(date -u --rfc-3339=ns | sed -r 's/ /T/; s/\.([0-9]{3}).*/\.\1Z/')
echo "TS=[$TS]"
```

**Take the on demand backup.** A point in time clone needs a base backup plus WAL, and the scheduled one will not run until the configured hour, so without it the clone can fail on a lab that is otherwise finished.

Then insert a row after that timestamp and clone:

```
INSERT INTO distribution_centers (id, name, latitude, longitude) VALUES (999, 'Test DC', '0', '0');
```
```
gcloud sql instances clone INSTANCE postgres-orders-pitr --point-in-time $TS
```

`latitude` and `longitude` are **`character varying(50)`** here, not numeric, so quote them. The clone must be named exactly `postgres-orders-pitr` and left in place.

## Small things that cost minutes

**The VM is `postgresql-vm`, not `postgres-vm`.** The lab text says `postgres-vm` throughout, on both instances. A `--filter="name=postgres-vm"` returns nothing, and task 3 needs that VM's external IP.

**The migration user name is randomised.** `import_user` on one run, `migration_admin` on the next. Read it off the panel.

**A psql backslash command in a pasted block eats the following lines.** `\d distribution_centers` followed by an `INSERT` consumed the whole rest of the paste as arguments and printed a dozen `extra argument ... ignored` lines without running anything. Paste backslash commands alone.

**Save the timestamp outside the shell.** `$TS` is a shell variable and this lab loses its shell readily; the clone is the last scored step and needs it.

## Related files

- `gsp351-migrate-mysql-to-cloud-sql-dms-challenge.md`, the MySQL sibling, much less hostile.
- `gsp306-migrate-mysql-to-cloud-sql-challenge.md` for Cloud SQL instance flags and authorized networks.
- `gotchas.md`, the log reading section and the scorer reads the job note, both of which came from this lab.
