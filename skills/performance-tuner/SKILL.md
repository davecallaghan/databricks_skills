---
name: performance-tuner
description: Spark and Delta Lake query performance tuning — Adaptive Query Execution (AQE), broadcast joins, shuffle partition sizing, scan optimization, and query profile analysis. Read this skill when the user asks to tune query performance, optimize Spark jobs, or analyze slow queries.
---

# Performance Tuning Patterns

## Top Performance Levers
| Lever | Setting | When |
|---|---|---|
| AQE | `spark.sql.adaptive.enabled = true` | Always (default on DBR 15+) |
| Broadcast join threshold | `spark.sql.autoBroadcastJoinThreshold = 100MB` | Small dim tables (<100MB) |
| Shuffle partitions | `spark.sql.shuffle.partitions = 200` (tune to data size) | Large shuffles |
| File compaction | `OPTIMIZE table` | Many small files |
| Liquid clustering | `CLUSTER BY (col)` | Multi-dimension queries |

## Query Analysis Workflow
1. Get the query profile from the executed statement
2. Check for: full table scans, large shuffles, broadcast timeouts, skew
3. Apply the appropriate optimization

## Common Anti-Patterns and Fixes

### Full Table Scan on Large Table
```sql
-- BAD: No filter on 6M-row lineitem table
SELECT * FROM coe_tpch.source.lineitem;

-- GOOD: Push down filter with partition/cluster pruning
SELECT * FROM coe_tpch.source.lineitem
WHERE l_shipdate BETWEEN '1997-01-01' AND '1997-12-31';
```

### Skewed Join
```python
# Detect skew: one partition has most of the data
skew_check = spark.sql(f"""
SELECT l_suppkey, COUNT(*) as cnt
FROM coe_tpch.source.lineitem
GROUP BY l_suppkey
ORDER BY cnt DESC LIMIT 10
""")
display(skew_check)

# Fix: Salting or broadcast join for small dimension
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 100 * 1024 * 1024)  # 100MB
```

### Small File Problem
```sql
-- Check file count and sizes
DESCRIBE DETAIL coe_tpch.silver.lineitem;

-- Compact if >10K files or avg file size <100MB
OPTIMIZE coe_tpch.silver.lineitem;
```

## TPC-H Benchmark Queries
The CoE program uses TPC-H queries for performance benchmarking (Q2 accelerator: `performance_benchmark`). Key queries to benchmark:
- Q1: Pricing summary (aggregate-heavy on lineitem)
- Q3: Shipping priority (join orders + lineitem + customer)
- Q5: Local supplier volume (5-table join through nation)
- Q10: Returned item reporting (join customer + orders + lineitem)

## Spark UI / Profile Analysis
When analyzing a slow query, look for:
- **Scan time**: Are we reading more data than needed? Add filters or clustering.
- **Shuffle bytes**: Large shuffles indicate need for broadcast or better partitioning.
- **Spill to disk**: Increase `spark.sql.shuffle.partitions` or add memory.
- **Skew**: One task takes much longer than others. Use salting or broadcast.
- **Join type**: Avoid sort-merge join for small tables. Use broadcast join.