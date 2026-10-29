---
name: scd2-handler
description: Slowly Changing Dimension Type 2 (SCD2) implementation patterns for Delta Lake — effective date ranges, is_current flags, temporal joins, and MERGE-based upserts. Read this skill when the user asks to implement SCD2, track dimension history, or build type 2 slowly changing dimensions.
---

# SCD Type 2 Patterns

## When to Use SCD2
Use SCD2 when you need to track the full history of dimension changes over time (e.g., customer address changes, supplier status changes). Each change creates a new row with an effective date range.

## Schema Pattern
Every SCD2 table must have:
- `effective_from` (TIMESTAMP) — when this version became active
- `effective_to` (TIMESTAMP) — when this version ended (NULL = current)
- `is_current` (BOOLEAN) — true if this is the latest version
- Business key (e.g., `c_custkey`) — the natural key that stays constant
- Surrogate key (e.g., `customer_sk`) — unique per version

## MERGE-Based SCD2 Pattern
```python
from pyspark.sql.functions import current_timestamp, lit, when, col

# Step 1: Identify changed records
changes_df = source_df.alias("src").join(
    target_df.filter(col("is_current") == True).alias("tgt"),
    on="c_custkey",
    how="inner"
).where(
    (col("src.c_address") != col("tgt.c_address")) |
    (col("src.c_acctbal") != col("tgt.c_acctbal"))
)

# Step 2: Expire old versions
spark.sql(f"""
MERGE INTO {catalog}.{schema}.customer_scd2 AS tgt
USING changes_df AS src
ON tgt.c_custkey = src.c_custkey AND tgt.is_current = true
WHEN MATCHED THEN UPDATE SET
  tgt.effective_to = current_timestamp(),
  tgt.is_current = false
""")

# Step 3: Insert new versions
new_versions = changes_df.select(
    "c_custkey", "c_name", "c_address", "c_nationkey", "c_phone",
    "c_acctbal", "c_mktsegment", "c_comment",
    current_timestamp().alias("effective_from"),
    lit(None).cast("timestamp").alias("effective_to"),
    lit(True).alias("is_current")
)
new_versions.write.mode("append").saveAsTable(f"{catalog}.{schema}.customer_scd2")
```

## Temporal (Point-in-Time) Join
```sql
-- Get the customer dimension record that was active at the time of the order
SELECT o.o_orderkey, o.o_totalprice, c.c_name, c.c_address
FROM coe_tpch.gold.orders o
LEFT JOIN coe_tpch.gold.customer_scd2 c
  ON o.o_custkey = c.c_custkey
  AND o.o_orderdate >= c.effective_from
  AND (o.o_orderdate < c.effective_to OR c.effective_to IS NULL)
```

## CoE Context
Q2 accelerator `scd2_dimension_handler` demonstrates SCD2 on the TPC-H customer table. The source table `coe_tpch.source.customer` is the driving source; the SCD2 target is `coe_tpch.gold.customer_scd2`.

## DBSQL SCD2 Pattern
For Databricks SQL, use `CREATE STREAMING TABLE ... FLOW AUTO CDC` for CDC-driven SCD2. See the `auto-cdc-dbsql` skill for DBSQL-specific syntax.