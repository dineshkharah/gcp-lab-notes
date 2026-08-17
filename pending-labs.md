# Pending labs

Labs that were read, analysed or started and then set aside. **Remove an entry from this list when its file lands in `labs/`.**

Anything not on this list either has a file in `labs/` or has never been looked at.

## The list

| Lab id | Name | State |
|---|---|---|
| GSP355 | Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab | analysed, never started |

## What is already known about each

### GSP355

The PostgreSQL twin of `labs/gsp351-migrate-mysql-to-cloud-sql-dms-challenge.md`, using Database Migration Service with **VPC peering** and a **continuous** job.

- **Split:** the pglogical prep on `postgres-vm` and tasks 2 to 4 are Cloud Shell; the DMS connection profile and migration job are console, for the same reason as GSP351, that the CLI's destination expects a profile that provisions a *new* Cloud SQL instance while the lab requires **Existing instance**.
- **The first value to read off the started page is the PostgreSQL major version.** The unstarted text leaves it as an unrendered template, and the package name `postgresql-VERSION-pglogical` plus both config paths under `/etc/postgresql/VERSION/main/` depend on it. One earlier run showed 14; do not assume it.
- **Order matters for pglogical.** Install the package, edit `postgresql.conf`, restart, *then* `CREATE EXTENSION`. The extension needs `shared_preload_libraries = 'pglogical'` already loaded, and doing it in the intuitive order gives an error that reads like a bad install.
- **The migration user name has spaces**, `Postgres Migration User`, so every SQL reference needs double quotes and the DMS username field takes it literally.
- **The primary key requirement is the silent failure.** Five tables need keys and the lab warns a badly prepared source "might fail to migrate some individual tables", so a migration that copies four of five looks green.
- **Two restarts** that cannot overlap the migration: enabling the IAM auth database flag, and enabling point in time recovery.
- Task 4 needs a UTC RFC 3339 timestamp taken *before* the row is added, and the clone must be named `postgres-orders-pitr` and left in place.

## Not pending

- **GSP364**, Managed Service for Prometheus: Challenge Lab, already has a file at `labs/gsp364-managed-prometheus-challenge.md` and will not be repeated. Check the README index before starting anything that feels familiar; it caught that one before a lab attempt was spent.
