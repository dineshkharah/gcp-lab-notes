# GSP1042, Analytics as a Service for Data Sharing Partners

Five tasks, **five** checkpoints, twenty five minutes stated. No Cloud Shell, console plus Looker Studio.

**Tasks 1, 2 and 3 are GSP1041 verbatim.** Same two views over `bigquery-public-data.geo_us_boundaries.zip_codes`, same authorization step on `demo_dataset`, same per view BigQuery Data Viewer grants to the two customer accounts. Everything in `gsp1041-bigquery-authorized-views-data-sharing.md` applies unchanged, including the **`Save view` then `Save view as`** trap on the second view. Read that file first, then only this page's differences matter.

Manual last updated May 2026, **last tested February 2026**, both older than GSP1041's July 2026 despite the two labs sharing three tasks. The product names in the text are stale accordingly.

## What this lab does instead of GSP1041's ending

GSP1041 finishes in BigQuery, proving the separation with two queries that return `Access Denied`. GSP1042 replaces that with a dashboard per customer and proves the separation by handing one customer the other's report link.

Two consequences worth knowing before starting.

**The order of the save changed.** GSP1041 saves the plain `SELECT *` off the authorized view as `customer_a_table`, then runs the join separately. GSP1042 runs the **join** first and saves *that* as `customer_a_table`:

```
SELECT geos.zip_code, geos.city, cust.last_name, cust.first_name
FROM `CUSTOMER_A_PROJECT.customer_a_dataset.customer_info` as cust
JOIN `PARTNER_PROJECT.demo_dataset.authorized_view_a` as geos
ON geos.zip_code = cust.postal_code;
```

That is deliberate. The pie chart later needs `city` as a dimension and it has to come from somewhere.

**Everything after the save is almost certainly unscored.** The checkpoints for tasks 4 and 5 are Qwiklabs checks, and Qwiklabs can see BigQuery but not a Looker Studio report living in the account's own Drive. So the scored artefact in each of the last two tasks is `customer_a_table` and `customer_b_table` existing in the right dataset. Worth clicking the checkpoint right after the Save dialog, before building any chart, and only continuing for the content.

## Data Studio is Looker Studio

The lab still says Data Studio throughout, including the product name in the connector dialog and the screenshots. It has been Looker Studio since 2022. `datastudio.google.com` redirects, so the link works and only the naming is behind.

Consequences of the rename that the text does not mention: the **Account setup** step the lab flags as "if prompted" is a per account, once ever dialog asking for country and terms, so it appears for **both** customer accounts. Same for the BigQuery connector's **Authorize** and the **Allow** consent dialog. Three extra clicks each way, twice.

## The chart

Blank Report, BigQuery connector, **Recent Projects**, then customer project > `customer_a_dataset` > `customer_a_table`, Add, Add to Report.

Rename the report to `Customer A Visualization`, `Insert > Pie chart`, then **drag `city` from Available Fields onto the `zip_code` dimension** to replace it. Metric stays Record Count. Dropping `city` next to `zip_code` rather than on top of it gives a two dimension chart that looks nothing like the screenshot.

## The identity dance in tasks 4 and 5

Heavier than GSP1041, and the lab's flow is better than it looks if followed exactly.

Task 4 ends with: copy the report link, **sign out of Looker Studio**, `Use another account`, sign in as **Customer B**, open the copied link, confirm no access.

That leaves the browser signed in as Customer B, which is exactly who task 5 needs. Do not sign out again at that point. Task 5's opening instruction to close the Customer A console and open the Customer B console still applies to the **Cloud console** tab, which is a separate session from the Looker Studio one.

Task 5 then mirrors it, ending signed in as Customer A looking at Customer B's report link and being refused.

Save both report links somewhere outside the browser. Signing out loses the tab and the link is not reconstructible.

## What the negative test actually proves

Not what the lab's narrative implies. The refusal at the end of tasks 4 and 5 is **Looker Studio's own sharing model**, not the authorized views. A new report is private to its creator, so any second account opening the link is refused regardless of what BigQuery permissions exist underneath.

The authorized views are not being tested there at all, and the distinction matters because of how Looker Studio data sources work. A data source defaults to **Owner's credentials**, meaning the report queries BigQuery as the person who built it. Had Customer A shared the report with Customer B, B would have seen Customer A's Texas rows, because the query would still run as A. The per view Data Viewer grant from task 3 would not have stopped it.

So the two mechanisms in this lab guard different things and neither substitutes for the other: authorized views control who can query what in BigQuery, Looker Studio sharing controls who can open the report. GSP1041's `Access Denied` queries are the real test of the first one.

## Related files

- `gsp1041-bigquery-authorized-views-data-sharing.md`, tasks 1 to 3 identical, and the BigQuery side of the separation tested properly.
- `gsp1043-consuming-datasets-data-twin.md`, the third lab in the family, where the three projects form a chain instead of a fan.
- `gsp375-share-data-google-data-cloud-challenge.md`, the challenge lab, which reuses this Looker Studio flow with a bar chart over the customer's view.
- `connected-sheets-bigquery.md` for the other route out of BigQuery into a reporting surface.
