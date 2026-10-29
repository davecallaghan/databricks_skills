---
name: liquid-cluster-advisor
description: Liquid Clustering key selection, migration from partitioning/Z-order, and performance tuning for Delta tables. Read this skill when the user asks about liquid clustering, cluster keys, partitioning strategy, or table optimization.
---

# Liquid Clustering Advisor

## When to Use Liquid Clustering
Liquid Clustering replaces partitioning and Z-ordering with a single, adaptive clustering mechanism. Use it when:
- Tables are queried by multiple dimensions (no single partition key)
- Write patterns are unpredictable
- You want to avoid partition maintenance (OPTIMIZE, ZORDER)

## Creating a Liquid-Clustered Table
```sql
CREATE TABLE coe_tpch.gold.lineitem_clustered (
  l_orderkey BIGINT,
  l_partkey BIGINT,
  l_suppkey BIGINT,
  l_shipdate DATE,
  l_quantity DOUBLE,
  l_extendedprice DOUBLE
)
USING DELTA
CLUSTER BY (l_shipdate, l_orderkey);
```

## Migrating from Partitioning/Z-Order
```sql
-- Convert an existing partitioned table
CREATE OR REPLACE TABLE coe_tpch.gold.orders_clustered
USING DELTA
CLUSTER BY (o_orderdate, o_custkey)
AS SELECT * FROM coe_tpch.gold.orders;

-- Drop the old table and rename
DROP TABLE coe_tpch.gold.orders;
ALTER TABLE coe_tpch.gold.orders_clustered RENAME TO orders;
```

## Key Selection Guidelines
| Pattern | Recommended Cluster Keys |
|---|---|
| Time-series queries (most recent data) | `(date_column)` |
| Multi-dimensional filtering | `(high_cardinality_col, low_cardinality_col)` |
| Range + point lookup | `(range_column, point_lookup_column)` |
| Join-heavy tables | `(join_key)` |

## Rules
- Max 4 cluster columns
- Order matters: first column is most important for pruning
- Avoid low-cardinality columns (like boolean) as first key
- Liquid clustering is adaptive — no manual OPTIMIZE/ZORDER needed
- Writes auto-cluster; background `OPTIMIZE` compacts and reclusters

## CoE Context
Q2 accelerator `liquid_cluster_advisor` demonstrates creating liquid-clustered TPC-H tables and comparing query performance vs partitioned tables. The TPC-H `lineitem` table (6M rows) is the primary candidate — clustered by `(l_shipdate, l_orderkey)` for shipping-date range queries and order-key joins.