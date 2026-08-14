# GSP306, Migrate a MySQL Database to Google Cloud SQL: Challenge Lab

Five tasks, **five** checkpoints, twenty minutes stated and **7 credits**. About **fifteen minutes**, and roughly ten of that is one Cloud SQL instance creating. Entirely Cloud Shell, no interactive SSH session.

Marked Advanced, and the difficulty is entirely in what the checkpoints actually measure rather than in the commands.

Manual last updated and last tested August 2026.

## You start at 20 points and can lose them

The lab says so: *"Your lab activity tracking score will initially report a score of 20 points because your blog is running... If the database has been incorrectly migrated, the blog is running test will fail, reducing your score by 20 points."*

`wp-config.php` is the only file whose edit can break the blog, so it goes **last**, after a `SHOW TABLES` has confirmed the data is actually in Cloud SQL. Back it up in the same command that edits it:

```
sudo cp /var/www/html/wordpress/wp-config.php /var/www/html/wordpress/wp-config.php.bak
```

That backup plus an `apache2 restart` is a complete path back to a working local database, which is worth having before touching a file that is holding a fifth of the score.

## Checkpoint 3 scores the authorized network, not the migration

Task 3 is titled *"Perform a database dump and import the data"* and its checkpoint reads **"Check that the blog instance is authorized to access Cloud SQL."**

So the thing that earns those points is:

```
gcloud sql instances patch $SQL_NAME --authorized-networks=$BLOG_IP --quiet
```

The dump and import are still needed, but for checkpoints 4 and 5. Reading the checkpoint text rather than the task title tells you which command to run first, and this is one of the clearest examples in these notes of the two disagreeing.

**It also lags.** Clicking immediately after the patch returned `No matching authorization found between instance IP and Cloud SQL instance IP` while `describe` showed the correct value already stored. Running the import first, then re-clicking, cleared it. Since a successful import **is** proof the authorization works, doing the import before retrying the checkpoint turns a guess into a test.

If it were still red after a working import, the remaining difference from the circulating guide is the CIDR form, `--authorized-networks=$BLOG_IP/32`, on the theory that the scorer does a string comparison. It was not needed here, and `patch` **replaces** the list rather than appending, so there is no reason to swap a demonstrably working value speculatively.

## Reuse the existing credentials, and the config edit becomes one line

Read the config before deciding any names:

```
gcloud compute ssh blog --zone=$ZONE --quiet --command='grep -n "DB_NAME\|DB_USER\|DB_PASSWORD\|DB_HOST" /var/www/html/wordpress/wp-config.php'
```

```
23:define('DB_NAME', 'wordpress');
26:define('DB_USER', 'blogadmin');
29:define('DB_PASSWORD', 'Password1*');
32:define('DB_HOST', 'localhost');
```

Create the Cloud SQL database as `wordpress` and the user as `blogadmin` with `Password1*` and **three of those four lines are already correct**, leaving `DB_HOST` as the only edit. Choosing different names means four edits and four chances to break the blog.

```
sudo sed -i "s/'DB_HOST', *'[^']*'/'DB_HOST', '$SQL_IP'/" /var/www/html/wordpress/wp-config.php
```

Substituting only the quoted value, rather than the circulating guide's `sed -i "/DB_HOST/c\define(...)"`, which rewrites **every** line containing `DB_HOST` including any comment that mentions it.

## --region and --zone cannot both be passed

The lab hands you both, *"Use the following values for the zone and region where applicable Zone: us-east1-d Region: us-east1"*, and the command accepts one:

```
ERROR: (gcloud.sql.instances.create) argument --region: At most one of --region | --gce-zone | --secondary-zone --zone can be specified.
```

**`--zone` alone**, since it implies the region and the lab names a zone:

```
gcloud sql instances create $SQL_NAME --database-version=MYSQL_8_0 --tier=db-n1-standard-1 --zone=$ZONE --root-password='Password1*'
```

The circulating guide passes only `--region`, which parses but lets Cloud SQL choose the zone. `describe` settles it:

```
gcloud sql instances describe $SQL_NAME --format="value(state,region,gceZone,settings.ipConfiguration.requireSsl,settings.ipConfiguration.sslMode)"
RUNNABLE   us-east1   us-east1-d   False   ALLOW_UNENCRYPTED_AND_ENCRYPTED
```

One command confirming state, zone **and** the SSL posture, which is the other thing that can silently block the import.

**`MYSQL_5_7` is the wrong version to reach for.** The community guide specifies it and it is past end of life on Cloud SQL, increasingly refused for new instances. `MYSQL_8_0` worked against a MariaDB source without incident.

## Two failure modes that cost eight minutes between them

**The unbounded wait.** An `until ... = "RUNNABLE"` loop with `2>/dev/null` on the `describe` printed `still creating` for eight minutes because the create above it had been rejected for the flag clash. The redirection that makes the loop robust also hides the reason it can never succeed. Bound the wait, or drop `--async` and let the create block. Now a section in `gotchas.md`.

**The 403 that was a 404.** With no instance present, `sql databases create` and `sql users create` both returned `HTTPError 403: The client is not authorized to make this request`, naming the account, which reads like a permissions problem. The `describe` on the next line said `HTTPError 404: The Cloud SQL instance does not exist`. Also now in `gotchas.md`.

## The source is MariaDB, not MySQL

```
-- MariaDB dump 10.19  Distrib 10.11.18-MariaDB, for debian-linux-gnu (x86_64)
```

Worth knowing because Cloud SQL has no MariaDB, so `MYSQL_8_0` is the only sensible target. A WordPress schema ports cleanly and the import produced no errors. The dump's first line is a MariaDB specific `/*M!999999\- enable the sandbox mode */` comment, which the MySQL client skips as a comment.

The dump and import both run **on the blog VM**, which already has the client, via `--command` rather than an interactive session:

```
gcloud compute ssh blog --zone=$ZONE --quiet --command='mysqldump -u blogadmin -p"Password1*" wordpress > /tmp/wordpress.sql; wc -l /tmp/wordpress.sql'
gcloud compute ssh blog --zone=$ZONE --quiet --command="mysql -h $SQL_IP -u blogadmin -p'Password1*' wordpress < /tmp/wordpress.sql"
gcloud compute ssh blog --zone=$ZONE --quiet --command="mysql -h $SQL_IP -u blogadmin -p'Password1*' -e 'SHOW TABLES;' wordpress"
```

`--quiet` on each, because the first `gcloud compute ssh` of a session generates a key pair and prompts for a passphrase. Note the quoting flips between the dump and the import: single quotes outside for the dump so nothing expands, double quotes outside for the import so `$SQL_IP` does expand, with the password in single quotes either way to protect the `*`.

569 lines for this blog, so the transfer is instant.

## Related files

- `gsp101-deploy-troubleshoot-website-challenge.md` and `gsp301-remote-startup-script-challenge.md` for the Apache on Compute Engine side, including why `http://` matters when testing.
- `gsp322-build-a-secure-network-challenge.md` for authorized ranges and tags on the Compute side of the same idea.
