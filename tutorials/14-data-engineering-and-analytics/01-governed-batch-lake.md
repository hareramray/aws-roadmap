# Reconcile a small sales lake and govern its query path

[Data tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../14-data-engineering-and-analytics.md)

**Outcome:** Turn seven raw rows into five accepted orders, retain an explicit rejection reason, query validated Parquet, and test an analyst's permissions. The local generator uses only Python. Optional S3/Athena/Glue/Lake Formation steps incur storage, requests, catalog/query, and possibly KMS costs. They require a dedicated sandbox bucket/prefix, approved Athena workgroup, and scoped deployment/analyst identities.

## 1. Define the metric before choosing an engine

The grain is one accepted order. `amount_cents` is a positive integer amount; `sale_day` is a UTC business date. Revenue is the sum of accepted amounts, excluding an identical redelivery and invalid negative values. A changed payload for an existing order ID is a conflict requiring investigation, not an arbitrary last-write-wins update. This is a full-refresh snapshot exercise; CDC requires the source-version rules in the next tutorial.

Save `make_sales.py`:

```python
import csv
import hashlib
import json
from collections import defaultdict
from datetime import date
from pathlib import Path

columns = ["order_id", "region", "amount_cents", "sale_day"]
rows = [
    ["o1", "north", "1000", "2026-01-01"],
    ["o2", "south", "1500", "2026-01-01"],
    ["o3", "north", "500", "2026-01-01"],
    ["o4", "south", "2000", "2026-01-02"],
    ["o5", "north", "750", "2026-01-02"],
    ["o2", "south", "1500", "2026-01-01"],
    ["o6", "north", "-100", "2026-01-02"],
]
def write_csv(path, header, values):
    with Path(path).open("w", newline="", encoding="utf-8") as handle:
        writer = csv.writer(handle)
        writer.writerow(header)
        writer.writerows(values)

write_csv("raw-sales.csv", columns, rows)
accepted, rejected, duplicates = {}, [], 0
for row in rows:
    key, region, amount, day = row
    try:
        if not key or region not in {"north", "south"} or int(amount) <= 0:
            raise ValueError("invalid key, region, or amount")
        date.fromisoformat(day)
        if key in accepted:
            if accepted[key] != row:
                raise ValueError("conflicting duplicate")
            duplicates += 1
            continue
        accepted[key] = row
    except ValueError as error:
        rejected.append(row + [str(error)])
totals = defaultdict(int)
for row in accepted.values():
    totals[row[1]] += int(row[2])
write_csv("accepted-sales.csv", columns, accepted.values())
write_csv("rejected-sales.csv", columns + ["reason"], rejected)
manifest = {"raw_rows": len(rows), "accepted": len(accepted),
            "duplicates": duplicates, "rejected": len(rejected),
            "region_totals_cents": dict(totals),
            "raw_sha256": hashlib.sha256(Path("raw-sales.csv").read_bytes()).hexdigest()}
Path("manifest.json").write_text(json.dumps(manifest, indent=2), encoding="utf-8")
assert (len(accepted), duplicates, len(rejected)) == (5, 1, 1)
assert dict(totals) == {"north": 2250, "south": 3500}
print(json.dumps(manifest, indent=2))
```

Run `python .\make_sales.py`. Expect seven raw rows, five accepted, one duplicate, one rejected, and 5750 total cents. The source hash and counts are publication evidence. In a multi-file batch, publish a completion manifest only after every required object is present; discovering one object does not prove the whole batch is ready.

## 2. Optional: publish isolated raw data

Confirm caller identity/Region, then create or use a dedicated private bucket with public access blocked and encryption. Record prefixes `roadmap-sales/raw/`, `roadmap-sales/curated/run-unique/`, `roadmap-sales/rejected/`, and a separately controlled query-result prefix. Upload only `raw-sales.csv` into `raw/`; placing the manifest or rejected CSV there would make Athena try to read them as raw table data.

```powershell
aws s3 cp .\raw-sales.csv "s3://$env:LAB_DATA_BUCKET/roadmap-sales/raw/raw-sales.csv"
if ($LASTEXITCODE -ne 0) { throw 'Raw upload failed' }
aws s3 cp .\manifest.json "s3://$env:LAB_DATA_BUCKET/roadmap-sales/manifests/run-unique.json"
if ($LASTEXITCODE -ne 0) { throw 'Manifest upload failed' }
```

Choose an Athena workgroup with a reviewed engine, bounded scan controls, and protected encrypted results. The external CTAS location below requires a workgroup configuration that permits it. If your workgroup enforces its own query-result location, follow its documented CTAS behavior and omit `external_location`; record the generated table location instead. Do not weaken an organization's enforced workgroup to fit a lab.

## 3. Create an explicit raw schema and validate

In Athena, run each SQL statement separately. Replace `YOUR_BUCKET` and `RUN_UNIQUE` before execution. Use a unique database if `roadmap_sales` is not exclusively yours.

```sql
CREATE DATABASE roadmap_sales;
```

```sql
CREATE EXTERNAL TABLE roadmap_sales.raw_orders (
  order_id string,
  region string,
  amount_cents string,
  sale_day string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ('separatorChar'=',', 'quoteChar'='"', 'escapeChar'='\\')
STORED AS TEXTFILE
LOCATION 's3://YOUR_BUCKET/roadmap-sales/raw/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

```sql
SELECT count(*) AS raw_rows,
       count_if(try_cast(amount_cents AS bigint) <= 0
                OR try_cast(amount_cents AS bigint) IS NULL) AS bad_amounts
FROM roadmap_sales.raw_orders;
```

Expect seven and one. Raw fields are strings so malformed source values can be identified explicitly instead of silently discarded by permissive deserialization. CSV does not provide a complete schema contract; embedded newlines and type handling depend on the SerDe. A crawler can infer a schema but cannot decide that a changed business field is valid.

For CLI execution, save one statement per UTF-8 file and call `aws athena start-query-execution --query-string file://query.sql --work-group $env:LAB_WORKGROUP --result-configuration OutputLocation=$env:LAB_RESULTS_URI`. Record `QueryExecutionId`, poll `get-query-execution` with a bounded deadline, and read results only after `SUCCEEDED`. A failed/cancelled query needs diagnostics, not an endless poll. Save scanned bytes and execution time, not only the returned rows.

## 4. Publish validated Parquet and reconcile again

First verify there are no conflicting duplicate IDs by grouping ID and counting distinct normalized payloads. The fixture contains only an identical duplicate. Stop publication if a key has different valid payloads; arbitrary `row_number` tie-breaking would hide a business conflict. Then run:

```sql
CREATE TABLE roadmap_sales.curated_orders
WITH (
  format='PARQUET',
  write_compression='SNAPPY',
  external_location='s3://YOUR_BUCKET/roadmap-sales/curated/RUN_UNIQUE/',
  partitioned_by=ARRAY['sale_day']
) AS
SELECT order_id, region, amount_cents, sale_day
FROM (
  SELECT order_id, region, try_cast(amount_cents AS bigint) AS amount_cents,
         sale_day,
         row_number() OVER (PARTITION BY order_id ORDER BY order_id) AS delivery_number
  FROM roadmap_sales.raw_orders
  WHERE order_id <> '' AND region IN ('north', 'south')
    AND try_cast(amount_cents AS bigint) > 0
    AND try_cast(sale_day AS date) IS NOT NULL
)
WHERE delivery_number=1;
```

```sql
SELECT region, count(*) AS orders, sum(amount_cents) AS revenue_cents
FROM roadmap_sales.curated_orders
GROUP BY region ORDER BY region;
```

Expect north: three orders/2250; south: two orders/3500. Query one date and compare scanned bytes with the raw query. Small fixtures may not show meaningful savings because billing minima/metadata dominate; use a representative controlled dataset before making performance claims. The partition column is last in the CTAS projection as required. Do not partition by every unique order ID.

Rerun the same CTAS unchanged: expect a table-exists or nonempty-location failure. This is the controlled failure. Use a new run-specific output prefix and publish a pointer/transactional table update only after validation, rather than appending another full snapshot and double-counting. Failed CTAS may leave orphan files; inspect its manifest/output prefix before cleanup.

## 5. Verify governance with a constrained analyst

Use separate publisher and analyst roles. Align IAM query permissions, Glue metadata, S3 locations, KMS keys, and Lake Formation grants. For a Lake Formation-managed exercise, register the dedicated data location with its appropriate role, grant the publisher required database/table/location permissions, and grant the analyst only SELECT/DESCRIBE on the intended curated table or supported filtered view. Review hybrid/IAMAllowedPrincipals behavior rather than assuming registration revokes every older access path.

Run three tests as the analyst: the approved aggregate succeeds; a prohibited raw-table/column query fails; direct S3 reading of disallowed raw objects fails. If direct S3 access succeeds, a query-layer filter alone is insufficient. Secure query results too, since they can expose otherwise restricted values. Test LF-tags, cross-account shares, and row/column filters on a separate sandbox table with the actual supported engine/table format before extending access.

## 6. Scale the processing without changing the contract

For a Glue/EMR Spark job, the equivalent pipeline is explicit schema read, cast/validate, quarantine, deduplicate after conflict checking, aggregate/reconcile, then publish. Start with this transformation shape in a supported Spark environment:

```python
from pyspark.sql import functions as F

# `raw` is a DataFrame read with the four source columns as strings.
typed = raw.withColumn("amount_cents", F.col("amount_cents").cast("long"))
valid = typed.filter((F.col("amount_cents") > 0) & F.col("region").isin("north", "south"))
# First enforce the same complete date/key/conflict rules as the local contract.
unique = valid.dropDuplicates(["order_id", "region", "amount_cents", "sale_day"])
summary = unique.groupBy("region").agg(F.sum("amount_cents").alias("revenue_cents"))
```

This excerpt demonstrates transformations, not a complete Glue job or full validator. Inspect lazy execution, shuffles, skew, partition counts, and plans; do not `collect()` an unbounded dataset. Glue bookmarks track supported source progress, not arbitrary end-to-end exactly-once output, and DataFrame/connector support differs. Step Functions, Glue workflows, or MWAA/Airflow should orchestrate readiness, retry, and publication, not redefine correctness. Avoid deploying an always-running orchestrator for a five-row exercise.

For mutable analytical tables, create a separate supported Iceberg table and test insert, update/delete, snapshot inspection, and time travel with your chosen Athena engine and permission model. Verify the old snapshot before expiring it. Iceberg/Hudi/Delta compatibility differs across writers/readers; Parquet alone supplies none of their table transactions. S3 Tables adds managed Iceberg maintenance with its own integration and charges. Do not delete referenced files with raw S3 lifecycle rules. Compare Avro for row-oriented exchange and ORC/Parquet for columnar scans based on real consumers.

## 7. Cleanup and explain

Drop dedicated catalog tables/database, revoke dedicated grants, deregister only the lab location, and remove query outputs, manifests, raw/curated/rejected files, object versions, and orphan CTAS files according to the resource ledger. Dropping an external table does not delete its S3 data. Remove dedicated roles/keys only after checking dependencies. Stop/delete any Glue/EMR jobs or applications and workgroup resources you created. Keep sanitized reconciliation evidence.

Explain why five accepted rows are correct, why a LIMIT does not guarantee a cheap scan, why a crawler is not a quality gate, and why a table permission may be bypassed by broad direct storage access.

Official references: [CSV SerDe](https://docs.aws.amazon.com/athena/latest/ug/csv-serde.html), [CTAS considerations](https://docs.aws.amazon.com/athena/latest/ug/ctas-considerations-limitations.html), [Athena/Lake Formation](https://docs.aws.amazon.com/lake-formation/latest/dg/athena-lf.html), [Glue connections](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-connect.html), [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html).
