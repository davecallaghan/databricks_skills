---
name: ontology-builder
description: Unity Catalog ontology building — Pages, business terms, domains, semantic metadata (synonyms, display names, comments), and knowledge graph construction. Read this skill when the user asks to build an ontology, create UC Pages, add business terms, or structure semantic metadata for Genie.
---

# Ontology Builder

## Why Ontology Matters
The Genie Ontology is the semantic layer that connects UC tables to business meaning. Without it, Genie can answer SQL questions but cannot understand business terminology. The CoE program treats ontology as a first-class architectural concern.

## Key Components
| Component | Purpose |
|---|---|
| Domains | Top-level grouping for tables (e.g., Sales, Finance, Operations) |
| Pages | Business term definitions linked to tables/columns |
| UC Metric Views | Reusable metric definitions with semantic metadata |
| Column Comments | Per-column business descriptions (Genie readiness requirement) |
| Synonyms | Alternative names for columns (Genie uses these for NL-to-SQL) |
| Display Names | Human-readable column names |

## Adding Semantic Metadata to Tables
```sql
-- Add column comment (Genie readiness requirement)
ALTER TABLE coe_tpch.source.customer
  ALTER COLUMN c_custkey COMMENT 'Unique customer identifier (natural key)';
ALTER TABLE coe_tpch.source.customer
  ALTER COLUMN c_mktsegment COMMENT 'Customer market segment (AUTOMOBILE, BUILDING, FURNITURE, HOUSEHOLD, MACHINERY)';
```

## Creating UC Metric Views
UC metric views are the Genie semantic layer — they define metrics with `MEASURE()` syntax:

```sql
CREATE OR REFRESH METRIC VIEW coe_tpch.gold.revenue_by_nation_metrics (
  dimension nation STRING COMMENT 'Nation name' LABEL 'Nation',
  dimension order_year INT COMMENT 'Order year' LABEL 'Order Year',
  measure total_revenue DOUBLE COMMENT 'Sum of all order total prices' LABEL 'Total Revenue',
  measure order_count INT COMMENT 'Number of orders' LABEL 'Order Count',
  measure avg_order_value DOUBLE COMMENT 'Average order value' LABEL 'Average Order Value'
) AS
  SELECT
    n.n_name AS nation,
    YEAR(o.o_orderdate) AS order_year,
    SUM(o.o_totalprice) AS total_revenue,
    COUNT(o.o_orderkey) AS order_count,
    AVG(o.o_totalprice) AS avg_order_value
  FROM coe_tpch.source.orders o
  JOIN coe_tpch.source.customer c ON o.o_custkey = c.c_custkey
  JOIN coe_tpch.source.nation n ON c.c_nationkey = n.n_nationkey
  GROUP BY n.n_name, YEAR(o.o_orderdate);
```

## TPC-H Business Terms (CoE Program)
Key business terms for the TPC-H ontology:
- **Customer**: Entity purchasing from suppliers (c_custkey, c_name, c_mktsegment)
- **Supplier**: Entity providing parts (s_suppkey, s_name, s_nationkey)
- **Order**: Purchase order from a customer (o_orderkey, o_custkey, o_orderstatus)
- **Line Item**: Individual item in an order (l_orderkey, l_partkey, l_linenumber)
- **Part**: Catalog item available for purchase (p_partkey, p_name, p_brand)
- **Revenue**: Sum of order total prices (o_totalprice)
- **Discount**: Line item discount (l_discount, range 0.00-0.10)
- **Market Segment**: Customer category (AUTOMOBILE, BUILDING, FURNITURE, HOUSEHOLD, MACHINERY)

## CoE Context
Q3 accelerator `ontology_kg_builder` creates the TPC-H ontology: UC metric views for business metrics, column comments for Genie readiness, and business term definitions. The Q1 `genie-readiness-scorer` skill evaluates the ontology coverage dimension.

## Integration with Q1
The Q1 genie-readiness-scorer assesses 8 dimensions including Ontology Coverage (10% weight). The ontology-builder skill provides the patterns to improve that score: column comments, metric views, business terms, and Pages.