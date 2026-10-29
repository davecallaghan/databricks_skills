---
name: medallion-architect
description: Bronze-to-Silver-to-Gold medallion architecture patterns using Spark Declarative Pipelines (SDP), streaming tables, materialized views, and Delta Live Tables. Read this skill when the user asks to design a medallion architecture, build a multi-layer pipeline, or structure data for the CoE program.
---

# Medallion Architecture Patterns

## Layer Responsibilities
| Layer | Purpose | Patterns |
|---|---|---|
| Bronze | Raw ingestion, preserve source | Auto Loader, append-only, `_ingested_at` metadata |
| Silver | Clean, validate, conform | Deduplicate, type-cast, DQ constraints, CDF enabled |
| Gold | Business aggregates, facts/dims | Star schema, SCD2 dimensions, metric views |

## SDP (Spark Declarative Pipelines) Pattern
Use SDP for streaming + batch medallion pipelines with built-in data quality:

```python
import dlt

# Bronze: Streaming ingestion
@dlt.table(name="bronze_orders", comment="Raw orders from Auto Loader")
@dlt.expect("valid_orderkey", "o_orderkey IS NOT NULL")
def bronze_orders():
    return (spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("/path/to/orders"))

# Silver: Cleaned and validated
@dlt.table(name="silver_orders", comment="Cleaned orders with DQ constraints")
@dlt.expect_all_or_drop({
    "valid_orderkey": "o_orderkey IS NOT NULL",
    "valid_custkey": "o_custkey IS NOT NULL",
    "valid_status": "o_orderstatus IN ('O', 'P', 'F')",
    "positive_totalprice": "o_totalprice >= 0",
})
def silver_orders():
    return dlt.read("bronze_orders")

# Gold: Business aggregate
@dlt.table(name="gold_revenue_by_nation", comment="Revenue by nation")
def gold_revenue_by_nation():
    return (dlt.read("silver_orders")
        .join(dlt.read("silver_customer"), "o_custkey = c_custkey")
        .groupBy("c_nationkey")
        .agg(F.sum("o_totalprice").alias("total_revenue")))
```

## CoE Program Structure
The CoE program uses `coe_tpch` catalog with four schemas:
- `coe_tpch.source` — Raw TPC-H tables (nation, region, part, supplier, partsupp, customer, orders, lineitem)
- `coe_tpch.bronze` — Auto Loader ingestion (Q2 accelerator: bronze_ingestion)
- `coe_tpch.silver` — Cleaned and validated (Q2 accelerator: silver_transform)
- `coe_tpch.gold` — Business aggregates (Q2 accelerator: gold_aggregation)

## Key Principles
- **Idempotent pipelines**: Re-running should not produce duplicates — use MERGE or `availableNow` triggers
- **CDF on silver tables**: `ALTER TABLE ... SET TBLPROPERTIES (delta.enableChangeDataFeed = true)`
- **Column comments on every layer**: Required for Genie readiness scoring (Q1 integration)
- **Governance tags inherited**: PII tags from source propagate to bronze/silver (Q4 integration)