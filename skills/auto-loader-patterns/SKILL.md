---
name: auto-loader-patterns
description: Auto Loader (cloudFiles) ingestion patterns for Delta Lake — schema evolution, directory listing modes, file notification, incremental ingestion into Unity Catalog. Read this skill when the user asks to set up Auto Loader, configure cloudFiles, handle schema evolution, or build bronze-layer ingestion pipelines.
---

# Auto Loader Patterns

## When to Use Auto Loader
Auto Loader (`cloudFiles`) incrementally processes new files from cloud storage into Delta tables. Use it for:
- Bronze-layer ingestion in the CoE medallion architecture
- Streaming or batch file ingestion from S3/GCS/Azure Data Lake
- Schema-evolving sources where column additions are expected

## Core Configuration

```python
# Streaming Auto Loader into a Delta table
(spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "json")  # json, csv, parquet, xml, text, binaryFile
  .option("cloudFiles.schemaLocation", f"{catalog}.{schema}.schema_evolution")
  .option("cloudFiles.inferColumn", True)
  .option("cloudFiles.schemaEvolutionMode", "addNewColumns")  # addNewColumns | rescue | none
  .option("cloudFiles.useNotifications", True)  # file notification (faster) vs directory listing
  .load(source_path)
  .writeStream
  .format("delta")
  .option("checkpointLocation", checkpoint_path)
  .trigger(availableNow=True)  # batch-like processing for scheduled jobs
  .toTable(f"{catalog}.{schema}.{table_name}"))
```

## Schema Evolution Modes
| Mode | Behavior |
|---|---|
| `addNewColumns` | New columns appended to schema; existing columns unchanged |
| `rescue` | Unexpected columns go to `_rescued_data` column |
| `none` | Schema is fixed; unexpected columns dropped |

## Directory Listing vs File Notification
- **Directory listing** (`useNotifications=False`): No cloud infra needed, slower for >1K files. Default.
- **File notification** (`useNotifications=True`): Uses cloud queue (SQS, GCS Pub/Sub, Event Grid). Faster, requires setup.

## CoE Pattern: Bronze Ingestion with Metadata
Always append ingestion metadata columns:
```python
from pyspark.sql.functions import current_timestamp, input_file_name

raw_df = (spark.readStream
  .format("cloudFiles")
  .option("cloudFiles.format", "csv")
  .option("cloudFiles.schemaLocation", schema_loc)
  .option("header", True)
  .load(source_path)
  .withColumn("_ingested_at", current_timestamp())
  .withColumn("_source_file", input_file_name()))
```

## Trigger Modes
- `availableNow=True`: Process all available files then stop. Best for scheduled batch jobs.
- `processingTime='30s'`: Micro-batch every 30 seconds. Best for near-real-time.
- `once=True`: Process once and stop. Best for one-time loads.

## TPC-H Context
The CoE program uses TPC-H source tables in `coe_tpch.source` as the raw layer. Bronze ingestion notebooks (Q2 accelerators) use Auto Loader to simulate ingesting these tables as if they arrived as files from an external source.