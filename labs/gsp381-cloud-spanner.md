# gsp381, Cloud Spanner Create and Manage Instances challenge lab

Six tasks: instance, database, four tables, small inserts, a five hundred row load, and one added column. Ten minutes total if you skip Dataflow, twenty five to thirty if you follow the handout's suggestion.

Run twice now, once with region `us-east4` and once with `europe-west4`, so the region does change per run. Everything else stayed identical: instance `banking-ops-instance`, database `banking-ops-db`, the same four tables, and the same data.

## Do task 5 before clicking the tables checkpoint

The single "Create and Load Tables" checkpoint covers tasks 3 and 4, and on the second run it would not go green after the four tables plus the three small inserts, even though all of it had clearly worked. It cleared once the five hundred row `Customer` load had also run. So load `Customer` first and then click both checkpoints together, rather than stopping after task 4.

## Instance and database

```
export REGION=YOUR_REGION

gcloud spanner instances create banking-ops-instance \
  --config=regional-$REGION --description="Banking Operations Instance" --nodes=1

gcloud spanner databases create banking-ops-db --instance=banking-ops-instance
```

## Tables, one ddl call

`gcloud spanner databases ddl update` accepts `--ddl` more than once, so all four tables go in a single call. Portfolio, Category, Product, Customer, with the primary keys from the lab table.

## Inserts

The `Category` sample data has only three values while the table has four columns, so the column list has to be explicit or Spanner rejects it for arity mismatch:

```
gcloud spanner databases execute-sql banking-ops-db --instance=banking-ops-instance \
  --sql='INSERT INTO Category (CategoryId, PortfolioId, CategoryName) VALUES
    (1, 1, "Cash"), (2, 2, "Investments - Short Return"),
    (3, 2, "Annuities"), (4, 3, "Life Insurance")'
```

## The five hundred row load, without Dataflow

The lab suggests Dataflow or a client library in batch mode. One multi row dml statement is faster and simpler.

```
gcloud storage cp gs://spls/gsp381/Customer_List_500.csv .

python3 - <<'PYEOF' > customers.sql
import csv
rows = []
with open('Customer_List_500.csv', newline='') as f:
    for r in csv.reader(f):
        if len(r) < 3:
            continue
        v = [c.strip().replace('\', '\\').replace('"', '\\"') for c in r[:3]]
        rows.append('("%s","%s","%s")' % tuple(v))
print('INSERT INTO Customer (CustomerId, Name, Location) VALUES')
print(',\n'.join(rows) + ';')
PYEOF

gcloud spanner databases execute-sql banking-ops-db \
  --instance=banking-ops-instance --sql="$(cat customers.sql)"
```

Reported `Statement modified 500 rows` and took seconds. Fifteen hundred mutations, well under the per commit limit, and the statement is about thirty five kilobytes against a one megabyte limit.

## Added column

```
gcloud spanner databases ddl update banking-ops-db --instance=banking-ops-instance \
  --ddl='ALTER TABLE Category ADD COLUMN MarketingBudget INT64'
```

Verify with `gcloud spanner databases ddl describe`.
