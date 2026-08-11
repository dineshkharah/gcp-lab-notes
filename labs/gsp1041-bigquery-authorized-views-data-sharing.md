# GSP1041, Data Publishing on BigQuery using Authorized Views for Data Sharing Partners

Four tasks, **four** checkpoints, twenty minutes stated and it holds. **No Cloud Shell at all**, everything is BigQuery Studio in the console.

Checkpoints: one per task, all four scored on BigQuery objects existing and being readable by the right principal.

Manual last updated and last tested July 2026.

## Three identities, three consoles

The distinguishing feature of this lab. The credentials panel hands out **three** logins, not one:

- **Data Sharing Partner**, owns `demo_dataset` and both views. Tasks 1, 2 and 3 happen here.
- **Customer A**, owns `customer_a_dataset` with a `customer_info` table already in it.
- **Customer B**, same shape with `customer_b_dataset`.

The lab pane has a separate console button for each. Follow its instruction to **close one console before opening the next**. Signing into a second student account in the same browser profile while the first is still live is the usual way this goes wrong, and the symptom is an `Access Denied` on the view you just granted yourself, which reads like a broken grant.

Three sign ins with terms acceptance each is most of the twenty minutes. The actual work is six queries and four Save View dialogs.

Everything here has a `bq` equivalent, but three identities means three separately authenticated sessions, so the console really is the lighter path for once.

## Task 1, and the Save View trap

Two views over the same public table, differing only in the state filter:

```
SELECT * FROM `bigquery-public-data.geo_us_boundaries.zip_codes`
WHERE state_code="TX"
LIMIT 4000
```

```
SELECT * FROM `bigquery-public-data.geo_us_boundaries.zip_codes`
WHERE state_code="CA"
LIMIT 4000
```

Saved into `demo_dataset` as `authorized_view_a` and `authorized_view_b`.

**The menu item is different for the second one and that is deliberate.** The lab says `Save > Save View` for A and `Save > Save View as` for B. After saving view A the editor tab is bound to that view, so plain `Save` would overwrite `authorized_view_a` with the California query and you would end up with one view instead of two. Read the menu, do not pattern match off the first step.

## Task 2, what the authorization is really for

`demo_dataset` > **Share** > **Authorize Views**, then add both views by full path:

```
PARTNER_PROJECT.demo_dataset.authorized_view_a
PARTNER_PROJECT.demo_dataset.authorized_view_b
```

Worth being clear about what this does, because in this lab it does almost nothing. Authorizing a view on a dataset lets that view read the dataset's tables **on behalf of a caller who has no access to those tables**. That is the whole point of the feature. Here the underlying table is `bigquery-public-data`, which everyone can already read, and the views are being authorized against the dataset they themselves live in rather than one holding source tables. So the step is ceremonial and exists because the checkpoint looks for the authorization entries.

In a real setup the authorization goes on the dataset that holds the **source** tables, and the customer is never granted anything on that dataset at all.

## Task 3, the grant is on the view, not the dataset

`authorized_view_a` > **Share** > **Manage permissions** > **Add Principal**, the Customer A user address, role **BigQuery Data Viewer**. Same for B.

This is the step that actually enforces the separation, together with the `WHERE state_code` filter baked into each view's SQL. Data Viewer on a **single view** rather than on `demo_dataset` is what makes Customer A's query against `authorized_view_b` fail. Granting at dataset level instead would leave both customers able to read both views and task 4's two denial steps would return rows.

## Task 4, where two of the steps are supposed to fail

In each customer console, three queries and one save.

```
SELECT * FROM `PARTNER_PROJECT.demo_dataset.authorized_view_a`
```

Save that as a view into `customer_a_dataset` named `customer_a_table`. Cross project reference: the object in the customer's project is a view pointing at the partner's view, no data is copied. The name says table and it is a view, which is just the lab being loose.

Then the join that is the actual payoff, customer owned data enriched by partner data:

```
SELECT geos.zip_code, geos.city, cust.last_name, cust.first_name
FROM `CUSTOMER_A_PROJECT.customer_a_dataset.customer_info` as cust
JOIN `PARTNER_PROJECT.demo_dataset.authorized_view_a` as geos
ON geos.zip_code = cust.postal_code;
```

`zip_code` in `geo_us_boundaries` is a STRING, and `postal_code` in `customer_info` matches, so the join needs no cast. Texas rows only, because that filter is inside the view and the customer cannot see past it.

Last step in each console, the negative test:

```
SELECT * FROM `PARTNER_PROJECT.demo_dataset.authorized_view_b`
```

**`Access Denied` is the pass condition.** Half the verification in this lab is a query that must fail, which is easy to misread as something being broken right before clicking the final checkpoint.

## The project ids in the expected error are from a different run

The manual quotes the denial as:

```
Access Denied: Table qwiklabs-gcp-04-b39db6c444b1:demo_dataset.authorized_view_b: User does not have permission to query table ...
```

That `qwiklabs-gcp-04-...` id belongs to whatever run the manual was written from, while the same page's query blocks carry a `qwiklabs-gcp-03-...` partner project. They will not match each other or yours. Only the shape of the message matters. The usual rule applies: read your own ids off the lab panel and out of the query blocks the lab actually gives you, since each of the three projects has a different id and they are easy to swap by accident.

## Related files

Two sibling labs reuse tasks 1 to 3 of this one almost verbatim and then diverge:

- `gsp1042-analytics-as-a-service-data-sharing.md` ends in a Looker Studio dashboard per customer instead of the two `Access Denied` queries.
- `gsp1043-consuming-datasets-data-twin.md` rearranges the three projects into a chain, partner to publisher to customer, and starts from a destination table rather than a view.

Also:

- `gsp647-iam-permissions-with-gcloud.md` for the same class of problem in Cloud Shell, where the risk is the wrong `gcloud` configuration being active rather than the wrong console being open.
- `connected-sheets-bigquery.md` and `gsp374-bigquery-soccer-bqml.md` for the other BigQuery labs in these notes.
