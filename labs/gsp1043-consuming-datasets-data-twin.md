# GSP1043, Consuming Customer Specific Datasets from Data Sharing Partners using BigQuery

Four tasks, **three** checkpoints, twenty minutes stated. Console only, no Cloud Shell. **Task 4 is unscored.**

Third of three labs in this family. `gsp1041-bigquery-authorized-views-data-sharing.md` has the shared material, and this file only covers what is different. Manual last updated January 2026, the oldest of the three.

## The shape is a chain, not a fan

The real difference from the other two. GSP1041 and GSP1042 have one partner sharing with two parallel customers. Here the three projects sit in a line:

- **Data Sharing Partner** owns `demo_dataset` and the source data.
- **Data Publisher** owns `data_publisher_dataset` and sits in the middle, reading from the partner and republishing a narrowed slice.
- **Customer**, called the Data Twin, owns `customer_dataset` and reads only from the publisher.

Two hops, so two separate authorize plus grant pairs, one per hop. The word "Twin" is just the lab's term for a customer project holding views over someone else's data with nothing copied.

## Five sign ins, not three

The practical cost of the lab. Each task opens a different console and the lab has you close the previous one first, and **task 4 goes back to the partner and then back to the customer**, so the sequence is partner, publisher, customer, partner, customer. Five logins with terms acceptance on the first visit to each.

Keeping three sessions alive at once would fix it, but not with incognito windows, since Chrome shares one incognito session across every incognito window. It takes three distinct browser profiles, which is more setup than the lab is worth. Budget for the logins instead.

## Task 1 makes a table, not a view, and that is the point

Everything in the other two labs is `Save > Save view`. Here the first artefact is a real table, written by setting a query destination:

```
SELECT * FROM (
SELECT *, ROW_NUMBER() OVER (PARTITION BY state_code ORDER BY area_land_meters DESC) AS cities_by_area
FROM `bigquery-public-data.geo_us_boundaries.zip_codes`) cities
WHERE cities_by_area <= 10 ORDER BY cities.state_code
LIMIT 1000;
```

**More > Query Settings > Set a destination table for query results**, dataset `demo_dataset`, table `authorized_table`, Save, then **Run the query again**. The settings change alone writes nothing; the second run is what materialises the table. If `More` is not on the toolbar it is behind the three dots.

The reason it has to be a table is task 4, which inserts a row into it. A view cannot be written to, and without a writable source there is nothing to demonstrate at the end.

Then **Share > Authorize datasets**, adding `demo_dataset` to itself. Note this is a different feature from Authorize views, which is what the other two labs use and what task 2 here uses. An authorized dataset authorizes every view in that dataset at once instead of naming them one by one. Adding a dataset as an authorized dataset of itself is circular and does nothing, same ceremony as GSP1041's task 2, and the checkpoint wants it.

## The grant in task 1 quietly defeats the isolation

Task 1 grants **BigQuery Data Viewer on `authorized_table` to both the Data Publisher user and the Customer user.**

Give that a second look, because it undoes what the lab is teaching. The whole argument for an authorized view is that the consumer gets access to the view and **never** to the source. The correct wiring for this chain is to authorize the publisher's `authorized_view` against the partner's `demo_dataset`, so the view reads the partner table on the customer's behalf while the customer holds nothing on `demo_dataset` at all.

Instead the customer is handed direct read on the partner's table. Consequences:

- The `WHERE state_code="NY"` filter in the publisher's view is **cosmetic**. The customer can query `PARTNER_PROJECT.demo_dataset.authorized_table` directly and see all fifty states.
- The publisher's view works from the customer's session because of that direct grant, not because of any authorization. Both Authorize steps in this lab could be removed and the queries would still run.
- It is why task 4 works with no cross project authorization anywhere.

Follow the lab, the checkpoints want it. Just do not carry this wiring into anything real. GSP1041's two `Access Denied` queries are the check that would have caught it, and this lab has no equivalent.

## Task 2, the middle hop

```
SELECT *
FROM `PARTNER_PROJECT.demo_dataset.authorized_table`
WHERE state_code="NY"
LIMIT 1000
```

Saved as a view into `data_publisher_dataset` named `authorized_view`. Then **Authorize Views** on `data_publisher_dataset` adding that view, and Data Viewer on the view for the Customer user.

Same self referential authorization as before, and the labels differ from GSP1041 again: `Add Authorization` here, `Add authorization` there, `Authorize views` versus `Authorize Views`. The dialogs are the same dialogs.

## Task 3, and the join key changed

```
SELECT cities.zip_code, cities.city, cities.state_code, customers.last_name, customers.first_name
FROM `CUSTOMER_PROJECT.customer_dataset.customer_info` as customers
JOIN `PUBLISHER_PROJECT.data_publisher_dataset.authorized_view` as cities
ON cities.state_code = customers.state;
```

`customer_info` in this lab carries a `state` column and the join is on **state code**, not on `postal_code` as in the other two. Saved as a view called `customer_table` in `customer_dataset`, which is the scored artefact.

Because the join is on state and the view is filtered to NY, the result is every NY customer crossed with every NY city in the top ten by land area. Many rows per customer. That is expected, not a broken join.

## Task 4, unscored, and worth doing anyway

Partner inserts a row, customer re-runs the identical query and sees it. Nothing is copied anywhere, so the twin picks the row up immediately.

```
INSERT INTO `PARTNER_PROJECT.demo_dataset.authorized_table` (zip_code, city, county,
  state_fips_code, state_code, state_name, fips_class_code, functional_status,
  area_land_meters, area_water_meters, cities_by_area)
VALUES ("11012", "New City", "New County", "02", "NY", "New York", "B5", "S",
  123632007174.0, 544474039.0, 10)
```

Two things in that row are inconsistent and neither matters. `state_fips_code` is `02`, which is Alaska, against `state_code` of NY. And `cities_by_area` is hardcoded to `10` rather than derived, because the `ROW_NUMBER()` window from task 1 ran once into a static table and is not re-evaluated. The land area given is larger than any real value in the source, so a genuine ranking would have made this row 1.

That is the table versus view tradeoff made concrete. A view would have kept the ranking honest and been unwritable; the table is writable and its computed column is now a lie. Worth remembering when a lab or a design reaches for a destination table to make something appear live.

## Related files

- `gsp1041-bigquery-authorized-views-data-sharing.md`, the shared tasks and the properly tested version of the separation.
- `gsp1042-analytics-as-a-service-data-sharing.md`, the same first three tasks ending in Looker Studio dashboards.
- `gsp375-share-data-google-data-cloud-challenge.md`, the challenge lab, where the same authorize step is required rather than decorative.
