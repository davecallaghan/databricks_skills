---
name: data-lineage-visualizer
description: Trigger this skill when a user asks to trace, map, visualize, or document data lineage for PII, PHI, or regulated data — including upstream sources, downstream consumers, column-level lineage, PII flow paths, or audit evidence for regulators. Also triggers when producing lineage evidence for DSR responses, breach investigations, or compliance audits.
---

# Skill: Data Lineage Visualizer (Audit Evidence Layer)

## 1. Intent & Trigger
- Invoke this skill when the user asks to trace lineage for a table, column, or data flow involving PII/PHI.
- Also invoke when producing lineage evidence for regulators, auditors, DSR responses, or breach investigations.
- Also invoke when mapping where personal data comes from (upstream) and where it goes (downstream).
- Also invoke when the user asks "where does this column's data come from?" or "who consumes this table?".
- Do NOT invoke for the ontology-relationship-documenter's join-path mapping — this skill traces actual data flow, not semantic relationships.

## 2. Why Lineage Matters for Compliance

| Use Case | Regulatory Driver | What Lineage Provides |
|---|---|---|
| **DSR Access Request** | GDPR Art. 15, CCPA §1798.100 | Proof of all tables/columns containing the subject's data and how it got there |
| **Breach Scope** | GDPR Art. 33-34 | Which upstream tables fed the breached table, and which downstream tables received its data |
| **Cross-Border Transfer Evidence** | GDPR Chapter V | Proof of where data originated, where it was processed, and where it was transferred |
| **Data Minimization Audit** | GDPR Art. 5(1)(c) | Which PII columns flow from source to consumption — identify unnecessary persistence |
| **Right to Erasure** | GDPR Art. 17 | All tables and downstream consumers that must be purged or pseudonymized |
| **Regulator Audit** | GDPR Art. 30 (records of processing) | Complete processing chain from ingestion to consumption |

## 3. Lineage Discovery via System Tables

### Table-Level Upstream Lineage

```sql
-- All tables that feed INTO the target table (recursive upstream)
SELECT 
  source_table_full_name AS upstream_table,
  target_table_full_name AS target_table,
  'TABLE' AS entity_type
FROM system.access.table_lineage
WHERE target_table_full_name = '<catalog>.<schema>.<table>'
ORDER BY source_table_full_name;
```

### Table-Level Downstream Lineage

```sql
-- All tables that consume FROM the target table (recursive downstream)
SELECT 
  source_table_full_name AS source_table,
  target_table_full_name AS downstream_table,
  'TABLE' AS entity_type
FROM system.access.table_lineage
WHERE source_table_full_name = '<catalog>.<schema>.<table>'
ORDER BY target_table_full_name;
```

### Notebook/Job Provenance

```sql
-- Which notebooks and jobs read/write the target table
SELECT 
  source_table_full_name AS source,
  target_table_full_name AS target,
  source_notebook_full_name,
  target_notebook_full_name,
  source_workflow_id,
  target_workflow_id
FROM system.access.table_lineage
WHERE source_table_full_name = '<catalog>.<schema>.<table>'
   OR target_table_full_name = '<catalog>.<schema>.<table>'
ORDER BY source_table_full_name, target_table_full_name;
```

### Recursive Upstream Traversal (Multi-Hop)

```sql
-- Trace N hops upstream using a recursive CTE
WITH RECURSIVE upstream_chain AS (
  -- Base: direct upstream of the target
  SELECT 
    source_table_full_name AS ancestor,
    target_table_full_name AS descendant,
    1 AS hop
  FROM system.access.table_lineage
  WHERE target_table_full_name = '<catalog>.<schema>.<table>'

  UNION ALL

  -- Recursive: upstream of upstream
  SELECT 
    l.source_table_full_name AS ancestor,
    c.descendant,
    c.hop + 1 AS hop
  FROM system.access.table_lineage l
  JOIN upstream_chain c ON l.target_table_full_name = c.ancestor
  WHERE c.hop < 10  -- safety limit
)
SELECT DISTINCT ancestor, descendant, hop
FROM upstream_chain
ORDER BY hop, ancestor;
```

## 4. Column-Level Lineage

When table-level lineage is insufficient (e.g., a DSR needs to know exactly which source column feeds a specific PII column):

```sql
-- Column-level lineage (if available in system.access.column_lineage)
SELECT 
  source_table_full_name,
  source_column_name,
  target_table_full_name,
  target_column_name
FROM system.access.column_lineage
WHERE target_table_full_name = '<catalog>.<schema>.<table>'
  AND target_column_name IN ('<pii_column_1>', '<pii_column_2>')
ORDER BY source_table_full_name, source_column_name;
```

If `system.access.column_lineage` is not available, infer column lineage from notebook/job code:
1. Use `system.access.table_lineage` to identify notebooks that write the target table.
2. Read the notebook source code.
3. Parse the `SELECT` statements to identify which source columns map to which target columns.

## 5. PII Flow Mapping

Combine lineage with PII tags to produce a PII flow map — the chain of custody for personal data:

```sql
-- PII flow: upstream tables → target table → downstream consumers
WITH pii_tables AS (
  SELECT DISTINCT table_catalog, table_schema, table_name
  FROM <catalog>.governance.pii_inventory
  WHERE tag_key LIKE 'pii.%' OR tag_key LIKE 'phi.%'
),
upstream AS (
  SELECT source_table_full_name AS source, target_table_full_name AS target
  FROM system.access.table_lineage
  WHERE target_table_full_name IN (
    SELECT CONCAT(table_catalog, '.', table_schema, '.', table_name) FROM pii_tables
  )
),
downstream AS (
  SELECT source_table_full_name AS source, target_table_full_name AS target
  FROM system.access.table_lineage
  WHERE source_table_full_name IN (
    SELECT CONCAT(table_catalog, '.', table_schema, '.', table_name) FROM pii_tables
  )
)
SELECT 'UPSTREAM' AS direction, source, target FROM upstream
UNION ALL
SELECT 'DOWNSTREAM' AS direction, source, target FROM downstream
ORDER BY direction, source, target;
```

## 6. Evidence Export

Produce a structured evidence package for regulators or auditors:

```python
# Export complete lineage for a regulated table
upstream = spark.sql("""
  WITH RECURSIVE upstream_chain AS (
    SELECT source_table_full_name AS ancestor, target_table_full_name AS descendant, 1 AS hop
    FROM system.access.table_lineage
    WHERE target_table_full_name = '<catalog>.<schema>.<table>'
    UNION ALL
    SELECT l.source_table_full_name, c.descendant, c.hop + 1
    FROM system.access.table_lineage l
    JOIN upstream_chain c ON l.target_table_full_name = c.ancestor
    WHERE c.hop < 10
  )
  SELECT DISTINCT ancestor, descendant, hop FROM upstream_chain ORDER BY hop, ancestor
""")

downstream = spark.sql("""
  SELECT source_table_full_name, target_table_full_name
  FROM system.access.table_lineage
  WHERE source_table_full_name = '<catalog>.<schema>.<table>'
""")

pii_inventory = spark.sql("""
  SELECT table_catalog, table_schema, table_name, column_name, tag_key
  FROM <catalog>.governance.pii_inventory
  WHERE table_name = '<table>'
""")

# Write to governance evidence volume
evidence_path = f"/Volumes/<catalog>/governance/lineage_evidence/<table>_{datetime.now().strftime('%Y%m%d')}"
upstream.coalesce(1).write.mode("overwrite").json(f"{evidence_path}/upstream/")
downstream.coalesce(1).write.mode("overwrite").json(f"{evidence_path}/downstream/")
pii_inventory.coalesce(1).write.mode("overwrite").json(f"{evidence_path}/pii_inventory/")
```

## 7. Reference Output Template

### 🔍 Lineage Audit Report: [catalog].[schema].[table]

**Lineage Summary:**

| Direction | Tables in Chain | Max Depth | PII Tables in Chain |
|---|---|---|---|
| Upstream | [count] | [N hops] | [count] |
| Downstream | [count] | [N hops] | [count] |

**Upstream Lineage (Data Origins):**

| Hop | Source Table | PII Columns | Tag | Writes Via |
|---|---|---|---|---|
| 1 | [upstream table] | [columns] | [tag] | [notebook/job] |
| 2 | [upstream table] | [columns] | [tag] | [notebook/job] |

**Downstream Lineage (Data Consumers):**

| Hop | Target Table | PII Columns | Tag | Consumed By |
|---|---|---|---|---|
| 1 | [downstream table] | [columns] | [tag] | [notebook/job/dashboard] |

**PII Flow Map:**

```
[source_table.email] → [bronze.raw_customers.email] → [silver.customers.email (masked)] → [gold.customer_metrics (aggregated, no PII)]
[source_table.ssn] → [bronze.raw_customers.ssn] → [silver.customers.ssn (hashed)] → [no downstream consumers]
```

**Column-Level Lineage (for specific PII column):**

| Source Table | Source Column | Target Table | Target Column | Transformation |
|---|---|---|---|---|
| | | | | [passthrough / hash / mask / aggregate] |

**Evidence Export:**
- Path: `/Volumes/<catalog>/governance/lineage_evidence/<table>_<date>/`
- Contents: upstream.json, downstream.json, pii_inventory.json

**Compliance Relevance:**
- [Which regulatory use case this lineage evidence supports — DSR, breach scope, cross-border, minimization, erasure, records of processing]

## 8. Guardrails
- **Always include hop depth** — a single-hop lineage is insufficient for compliance evidence. Trace at least 5 hops upstream or until no further ancestors exist.
- **Cross-reference with PII inventory** — lineage without PII tag context is incomplete. Always overlay PII classification onto the lineage graph.
- **Note where transformations occur** — a passthrough column (raw PII flowing unchanged) is a higher risk than a hashed/aggregated column. Document the transformation at each hop.
- **Export evidence in machine-readable format** — JSON or Parquet, not screenshots. Regulators and auditors need queryable evidence.
- **Timestamp the evidence** — lineage is point-in-time. The lineage graph may change as pipelines evolve. Always include a snapshot date.
- **Check for orphan tables** — tables with no upstream lineage (no known source) are a compliance gap. Flag them for investigation.
- **Lineage gaps are evidence gaps** — if `system.access.table_lineage` returns no results for a table, it may mean CDF was not enabled or the table was created outside the lineage capture window. Flag as 'Lineage Incomplete'.
