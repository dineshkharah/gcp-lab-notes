# gcp lab notes

Notes from working through Google Cloud Skills Boost labs, mostly the challenge labs that sit at the end of a skill badge. The point of this repo is to not lose time to the same problem twice. Every lab that went wrong has the cause and the fix written down.

## How this is organised

- `workflow.md` is the process. How to run a lab from start to finish, what to line up before touching a command, what to check after each task.
- `gotchas.md` is the list of traps that turn up across many labs. Read this one first. Almost all the time lost in these labs came from a handful of repeating causes and they are all in there.
- `labs/` has one file per lab, named by its lab id. Each file lists the tasks, the commands that actually worked, and what went wrong.
- `labs/assets/` holds files a lab needs uploaded or imported, referenced from the lab file that uses them.

## How to use it before a lab

1. Read `gotchas.md`.
2. Open the matching file in `labs/` if there is one.
3. Copy the values off the started lab page into the commands.

## Starting a session from nothing

The whole method lives in `workflow.md`, so a cold start needs one line of setup:

> Read this repo, follow `workflow.md` and `gotchas.md`, then here is the lab.

`workflow.md` opens with the reading order and the house rules gathered in one block, so nothing else has to be explained first.

## Labs written up so far

Challenge labs:

- `labs/alloydb-create-and-manage-instances.md`
- `labs/arc101-monitor-and-manage-resources.md`
- `labs/arc109-api-gateway-challenge.md`
- `labs/arc114-speech-and-language-challenge.md`
- `labs/arc122-vision-api-challenge.md`
- `labs/arc126-apps-script-and-appsheet-challenge.md`
- `labs/arc130-natural-language-challenge.md`
- `labs/arc131-speech-to-text.md`
- `labs/cloud-storage-three-bucket-lab.md`
- `labs/connected-sheets-bigquery.md`
- `labs/gsp315-store-process-and-manage-data.md`
- `labs/gsp322-build-a-secure-network-challenge.md`
- `labs/gsp374-bigquery-soccer-bqml.md`
- `labs/gsp380-bigtable.md`
- `labs/gsp381-cloud-spanner.md`
- `labs/gsp393-cicd-pipelines-challenge.md`
- `labs/gsp511-infrastructure-for-aws-professionals.md`
- `labs/gsp526-privileged-access-manager-challenge.md`
- `labs/gsp528-ncc-challenge.md`
- `labs/streaming-analytics-into-bigquery.md`

Qwik starts and guided labs:

- `labs/dataflow-qwik-start-python.md`
- `labs/gsp235-apps-script-sheets-maps-gmail.md`
- `labs/gsp250-google-chat-bots-apps-script.md`
- `labs/gsp1048-spanner-database-fundamentals.md`
- `labs/gsp1079-continuous-delivery-cloud-deploy.md`
- `labs/gsp1131-artifact-registry-qwik-start.md`
- `labs/gsp1146-appsheet-no-code-chat-apps.md`
- `labs/gsp1049-spanner-loading-and-backups.md`
- `labs/gsp1050-spanner-schemas-and-query-plans.md`
- `labs/gsp1316-ncc-site-to-site-ha-vpn.md`
- `labs/gsp1317-ncc-vpc-to-vpc-and-psc.md`
- `labs/gsp1318-ncc-hybrid-connectivity.md`
- `labs/gsp190-iam-custom-roles.md`
- `labs/gsp499-iap-user-authentication.md`
- `labs/gsp647-iam-permissions-with-gcloud.md`
- `labs/gsp081-cloud-run-functions-console.md`
- `labs/gsp096-pubsub-qwik-start.md`
- `labs/gsp872-api-gateway-qwik-start.md`
- `labs/stream-processing-pubsub-to-dataflow.md`

Older ones with no commands kept: `labs/earlier-labs.md`

## The values in these notes are not answers

Every lab hands out fresh names when it starts. Region, zone, project id, bucket name, topic name, table name, file names, all of it changes per run. The values written here are from one old run and are only there to show the shape of the command. Read your own off the lab page.
