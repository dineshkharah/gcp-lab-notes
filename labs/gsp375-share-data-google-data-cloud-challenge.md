# GSP375, Share Data using Google Data Cloud: Challenge Lab

Four tasks, **five** checkpoints, twenty five minutes stated and comfortably under that. Console plus Looker Studio, no Cloud Shell.

The badge closer for GSP1041, GSP1042 and GSP1043, and it is those three recombined. Nothing new to learn, but it is the one lab in the family where the authorization step actually does something.

Checkpoints: create the partner view, authorize plus grant on the partner side, create the customer view, authorize plus grant on the customer side, connect BigQuery to Looker Studio. **Task 2 is unscored and the chart itself is unscored.**

Manual last updated May 2026, **lab last tested May 2025**. A year apart, the widest gap in the family.

## What is actually new: the exchange goes both ways

The guided labs all push data one direction, partner to customer. This one adds the return leg:

- Partner publishes `demo_dataset.authorized_view_RANDOM` over the public zip codes table, granted to the customer.
- Customer uses it to fill in a missing `county` column on their own `customer_info`.
- Customer publishes `customer_dataset.customer_authorized_view_RANDOM`, a county rollup, granted back to the **partner**.
- Partner charts the customer's aggregate without ever seeing customer rows.

Two authorized views, one in each direction, each with its own authorize step and its own Data Viewer grant. Three console switches: partner, customer, partner.

## The view names are randomised per run

`authorized_view_i8xl` and `customer_authorized_view_ic1w` in our run. The four character suffixes change every time and the scorer wants them exactly. Read them off your own lab page, do not copy them from here or from a script.

## The one authorization in this family that is load bearing

Worth stopping on, because the three guided labs all make this step look pointless and here it is not.

**Task 1 is ceremonial**, same as GSP1041. The partner's view reads `bigquery-public-data.geo_us_boundaries.zip_codes`, which everyone can already read, and the authorization is added against `demo_dataset`, the dataset the view lives in rather than one holding source tables. Remove it and the customer's query still works.

**Task 3 is real.** The customer's view selects from `customer_dataset.customer_info`, which lives in the **same dataset as the view**, and the partner is granted Data Viewer on the view alone and nothing on `customer_info`. Authorizing the view against `customer_dataset` is the only thing that lets the view read that table on the partner's behalf. Skip it and task 4 fails with an `Access Denied` naming `customer_info`, a table the lab never told you to grant anything on.

That is the whole feature working as designed, and it is the difference between this lab and GSP1043, where the customer was simply handed direct read on the partner's source table and the authorizations were decoration.

## Task 2 is unscored and task 4 depends on it

```
UPDATE `CUSTOMER_PROJECT.customer_dataset.customer_info` cust
SET cust.county = vw.county
FROM `PARTNER_PROJECT.demo_dataset.authorized_view_RANDOM` vw
WHERE vw.zip_code = cust.postal_code;
```

`This statement modified 14 rows in customer_info.`

No checkpoint on it, which makes it exactly the kind of step that gets skipped in a challenge lab. Do not skip it. The customer view in task 3 is:

```
SELECT county, COUNT(1) AS Count
FROM `CUSTOMER_PROJECT.customer_dataset.customer_info` cust
GROUP BY county
HAVING county is not null
```

`county` starts out null on every row, so before the update that view returns **zero rows**. The view still gets created and its checkpoint still passes, and the chart in task 4 then has nothing to draw. An empty chart at the end of a challenge lab with a green checkpoint behind it is a slow thing to diagnose.

Order is partner view, then update, then customer view. The update is also the "enrich datasets based on curated data" objective, so it is content rather than plumbing.

## Task 4, and finding the customer project in the connector

The partner builds the report, over the customer's view, cross project.

The partner holds **one view level grant** in the customer project and nothing else, so it does not necessarily show up under **My Projects** in the BigQuery connector, which needs permission to list the project. If it is not there, the connector's **Custom Query** option takes the fully qualified name directly:

```
SELECT * FROM `CUSTOMER_PROJECT.customer_dataset.customer_authorized_view_RANDOM`
```

Billing stays on the partner project either way, which is presumably how the checkpoint sees this at all. It is named "Connect BigQuery to Data Studio" and cannot read a report living in the account's Drive, so it is almost certainly reading a BigQuery job in the partner project that referenced the customer's view.

Chart requirements, verbatim from the lab:

- Report name: `Data Sharing Partner Vizualization`
- **Vertical Bar Chart**
- `county` as **Dimension**, `Count` as **both Breakdown Dimension and Metric**

Two oddities, both harmless. The report name is misspelled in the lab text and copying it exactly costs nothing. And `Count` as a Breakdown Dimension is strange, since a breakdown is normally a second categorical field and `Count` is the numeric aggregate; the lab asks for it in both slots and the screenshot matches. Follow the text rather than fixing it.

Data Studio has been Looker Studio since 2022, see `gsp1042-analytics-as-a-service-data-sharing.md`.

## Related files

- `gsp1041-bigquery-authorized-views-data-sharing.md`, the mechanics of authorized views and the per view grant.
- `gsp1042-analytics-as-a-service-data-sharing.md`, the Looker Studio half, including the per account setup dialogs.
- `gsp1043-consuming-datasets-data-twin.md`, the chained three project version, and the counterexample where the authorization does nothing.
