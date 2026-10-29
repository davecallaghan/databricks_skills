---
name: incremental-merge
description: Incremental MERGE patterns using Delta Change Data Feed (CDF) — idempotent upserts, watermarking, deduplication, and streaming MERGE into Delta tables. Read this skill when the user asks to build incremental pipelines, use MERGE with CDF, or implement idempotent data loads.
---

# Incremental MERGE Patterns

## When to Use Incremental MERGE
Use incremental MERGE when you need to:
- Process only new/changed rows from a source table
- Update existing rows in a target Delta table
- Maintain idempotency (re-running produces the same result)

## CDF-Based Incremental Read
```python
# Read only changes since the last checkpoint using Change Data Feed
changes_df = (spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", "latest")  # or a specific version number
    .table(f"{catalog}.{schema}.source_table"))
```

## MERGE Pattern for Upserts
```python
# Incremental MERGE: insert new, update existing, using CDF
def merge_changes(batch_df, batch_id):
    batch_df.createOrReplaceTempView("changes")
    spark.sql(f"""
    MERGE INTO {catalog}.{schema}.target AS tgt
    USING changes AS src
    ON tgt.id = src.id
    WHEN MATCHED AND src._change_type IN ('update', 'insert') THEN UPDATE SET *
    WHEN NOT MATCHED AND src._change_type IN ('insert', 'update') THEN INSERT *
    WHEN MATCHED AND src._change_type = 'delete' THEN DELETE
    """)

(spark.readStream
    .format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", "latest")
    .table(source_table)
    .writeStream
    .foreachBatch(merge_changes)
    .option("checkpointLocation", checkpoint_path)
    .trigger(availableNow=True)
    .start())
```

## Idempotency Rules
1. Always use `foreachBatch` with MERGE (not direct `writeStream`)
2. Use checkpoint locations per target table to avoid reprocessing
3. Handle `_change_type` from CDF: `insert`, `update_postimage`, `update_preimage`, `delete`
4. For non-CDF sources, use a watermark column (e.g., `updated_at`) to identify changes

## Batch Incremental (Non-Streaming)
```python
# For scheduled batch jobs without streaming
latest_version = spark.sql(f"DESCRIBE HISTORY {catalog}.{schema}.source LIMIT 1").select("version").collect()[0][0]
changes = spark.read.format("delta").option("readChangeFeed", "true").option("startingVersion", last_processed_version).table(source_table)
# Apply MERGE as above
```

## CoE Context
Q2 accelerator `incremental_merge_handler` demonstrates incremental MERGE from `coe_tpch.source` tables (CDF enabled) into `coe_tpch.silver` tables. CDF was enabled during TPC-H setup on: customer, orders, lineitem, part, supplier.