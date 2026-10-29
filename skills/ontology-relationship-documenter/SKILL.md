---
name: ontology-relationship-documenter
description: Trigger this skill when a user asks to document, map, or discover relationships between Unity Catalog objects — tables, views, metric views, foreign keys, or join paths. Also triggers when building an ontology relationship graph, documenting lineage as semantic context, or preparing a knowledge-graph data model for Genie discoverability.
---

# Skill: Ontology Relationship Documenter

## 1. Intent & Trigger
- Invoke this skill when the user asks to document relationships between UC tables, views, or metric views.
- Also invoke when the user asks to build a join-path map, relationship graph, or knowledge-graph data model.
- Also invoke when the user asks to enrich the ontology with relationship metadata so Genie can infer join paths.
- Do NOT invoke for simple lineage tracing (use system table lineage directly) — this skill is about documenting relationships as semantic context.

## 2. Why Relationships Matter for Genie

Genie can answer cross-table questions only if it knows how tables connect. Without documented relationships, Genie may:
- Attempt incorrect joins (wrong column, wrong cardinality).
- Fail to answer questions that require joining two tables.
- Generate invalid SQL because it guesses the join key.

Documenting relationships in the ontology teaches Genie the correct join paths, cardinalities, and business context — making cross-table natural-language queries reliable.

## 3. Relationship Types

| Relationship | Notation | Example | Cardinality |
|---|---|---|---|
| **Foreign Key** | `FK(source_table.col → target_table.col)` | `orders.customer_id → customers.customer_id` | Many-to-one |
| **Bridge/Junction** | `BRIDGE(table1.col ↔ bridge.col ↔ table2.col)` | `orders.order_id ↔ order_items.order_id` | One-to-many |
| **Derivation** | `DERIVES(view → source_table)` | `gold.customer_metrics → silver.orders` | View-to-source |
| **Shared Dimension** | `CONFORMS(dim_table → fact_table)` | `dim_date.date_id → orders.order_date_id` | One-to-many |
| **Hierarchy** | `HIERARCHY(child → parent)` | `dim_product.sku → dim_product.category` | Many-to-one |
| **Semantic Equivalence** | `EQUIVALENT(col_a ≡ col_b)` | `customers.cust_id ≡ orders.customer_id` | Identity |

## 4. Discovery Workflow

### Step 1: Automated Lineage Discovery

Use `system.access.table_lineage` to discover existing upstream/downstream relationships:
```sql
SELECT 
  source_table_full_name AS source,
  target_table_full_name AS target,
  'DERIVES' AS relationship_type
FROM system.access.table_lineage
WHERE target_table_catalog = '<catalog>'
  AND target_table_schema = '<schema>'
ORDER BY source, target;
```

### Step 2: Foreign Key Inference

Infer foreign keys from column naming conventions and data overlap:
```sql
-- Heuristic: columns ending in _id that match a PK in another table
SELECT 
  c1.table_catalog, c1.table_schema, c1.table_name, 
  c1.column_name AS fk_column,
  c2.table_catalog AS ref_catalog, c2.table_schema AS ref_schema, 
  c2.table_name AS ref_table, c2.column_name AS ref_column
FROM system.information_schema.columns c1
JOIN system.information_schema.columns c2 
  ON c1.column_name = c2.column_name 
  AND c1.table_catalog = c2.table_catalog
  AND c1.table_name != c2.table_name
WHERE c1.column_name LIKE '%_id'
  AND c1.table_schema = '<schema>'
  AND c2.column_name = c1.column_name
ORDER BY c1.table_name, c1.column_name;
```

### Step 3: Cardinality Verification

For each inferred relationship, verify cardinality by checking distinct value overlap:
```sql
-- Verify: orders.customer_id → customers.customer_id (many-to-one)
SELECT 
  COUNT(DISTINCT o.customer_id) AS distinct_fk_values,
  COUNT(*) AS total_fk_rows,
  (SELECT COUNT(DISTINCT c.customer_id) FROM <catalog>.<schema>.customers c) AS distinct_pk_values
FROM <catalog>.<schema>.orders o;
-- If distinct_fk_values <= distinct_pk_values and total_fk_rows > distinct_fk_values → many-to-one
```

### Step 4: Document as Ontology Entry

Store discovered relationships in a governance metadata table that Genie can reference:

```sql
CREATE TABLE IF NOT EXISTS <catalog>.governance.ontology_relationships (
  relationship_id STRING,
  relationship_type STRING,  -- FK, BRIDGE, DERIVES, CONFORMS, HIERARCHY, EQUIVALENT
  source_catalog STRING,
  source_schema STRING,
  source_table STRING,
  source_column STRING,
  target_catalog STRING,
  target_schema STRING,
  target_table STRING,
  target_column STRING,
  cardinality STRING,       -- one-to-one, one-to-many, many-to-one, many-to-many
  business_description STRING,
  is_verified BOOLEAN,
  documented_by STRING,
  documented_at TIMESTAMP
);

-- Example insert
INSERT INTO <catalog>.governance.ontology_relationships
VALUES (
  'rel_001', 'FK',
  '<catalog>', '<schema>', 'orders', 'customer_id',
  '<catalog>', '<schema>', 'customers', 'customer_id',
  'many-to-one',
  'Each order belongs to exactly one customer; a customer can have many orders',
  true, current_user(), current_timestamp()
);
```

### Step 5: Genie Space Integration

Include relationship documentation in Genie Space instructions so Genie knows the join paths:

```
## Table Relationships
- orders.customer_id → customers.customer_id (many-to-one): Each order belongs to one customer
- order_items.order_id → orders.order_id (many-to-one): Each line item belongs to one order
- dim_date.date_id → orders.order_date_id (one-to-many): Date dimension conforms to order dates
- gold.customer_metrics → silver.orders (derivation): Customer metrics are aggregated from orders

When a user asks about customer spending, join orders to customers on customer_id.
When a user asks about product details in an order, join orders to order_items on order_id, then to dim_product on product_id.
```

## 5. Knowledge Graph Foundation

The relationship table is the seed for a future knowledge graph. Model it as nodes and edges:

```sql
-- Nodes: one per UC object
SELECT DISTINCT 
  source_catalog || '.' || source_schema || '.' || source_table AS node_id,
  source_table AS node_label,
  'table' AS node_type
FROM <catalog>.governance.ontology_relationships
UNION
SELECT DISTINCT 
  target_catalog || '.' || target_schema || '.' || target_table AS node_id,
  target_table AS node_label,
  'table' AS node_type
FROM <catalog>.governance.ontology_relationships;

-- Edges: one per relationship
SELECT 
  source_catalog || '.' || source_schema || '.' || source_table AS source_node,
  target_catalog || '.' || target_schema || '.' || target_table AS target_node,
  relationship_type AS edge_label,
  cardinality AS edge_property
FROM <catalog>.governance.ontology_relationships;
```

For Lakebase-backed knowledge graphs, these node/edge tables can be synced via reverse ETL or loaded directly into a graph store.

## 6. Reference Output Template

### 🕸️ Ontology Relationship Map: [catalog].[schema]

**Relationship Summary:**

| # | Type | Source | Target | Cardinality | Verified? | Description |
|---|---|---|---|---|---|---|
| 1 | FK | orders.customer_id | customers.customer_id | many-to-one | ✅ | Each order belongs to one customer |
| 2 | BRIDGE | orders.order_id ↔ order_items.order_id | order_items.product_id → products.product_id | one-to-many | ✅ | Order line items bridge orders to products |
| 3 | DERIVES | gold.customer_metrics | silver.orders | view-to-source | ✅ | Aggregated customer metrics |

**Genie Space Instruction Block:**
```
## Table Relationships
[Generated join-path documentation for Genie instructions]
```

**Knowledge Graph Seed SQL:**
```sql
-- Node and edge queries for graph store ingestion
```

**Coverage Assessment:**
- Tables with documented relationships: [X / Y]
- Tables with no relationships found: [list — may need manual review]
- Unverified relationships: [list — cardinality not yet confirmed]

## 7. Guardrails
- **Always verify cardinality before documenting** — inferred relationships may be wrong. Check distinct value overlap.
- **Include business descriptions** — a join key without a business description is not useful for Genie. Explain what the relationship means in business terms.
- **Flag unverified relationships separately** — Genie should not rely on unverified join paths.
- **Check for PII in relationship documentation** — do not document relationships that reveal PII column names or values in Genie Space instructions if the space is shared with unauthorized users.
- **Keep the relationship table in a governance schema** — not alongside business data. Access should be restricted to governance roles and Genie service principals.
- **Re-run discovery when schema changes** — new tables or columns may introduce new relationships. Recommend scheduling periodic rediscovery.
