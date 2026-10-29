---
name: uc-metric-view-author
description: Trigger this skill when a user asks to create, author, or generate Unity Catalog metric views for Genie Spaces or the Ontology semantic layer. Also triggers when the user needs to add semantic metadata (synonyms, display names, comments, format) to columns, or when building privacy-safe aggregation views for Genie.
---

# Skill: Unity Catalog Metric View Author (Ontology Layer)

## 1. Intent & Trigger
- Invoke this skill when the user asks to create UC metric views as the semantic layer for Genie Spaces or the broader Ontology.
- Also invoke when adding semantic metadata (synonyms, display_name, comment, format strings) to columns in existing tables or views.
- Also invoke when the user needs privacy-safe pre-aggregated views that eliminate re-identification risk for Genie consumption.
- Do NOT invoke for standard materialized views or plain SQL views without semantic metadata requirements.

## 2. What Is a Metric View?

A Unity Catalog metric view is a SQL DDL construct that defines dimensions and measures with rich semantic metadata so Genie (and other AI tools) can discover, understand, and query them through natural language. Unlike a plain view, a metric view carries:
- **Dimensions** — columns used for grouping/filtering (e.g., `region`, `order_date`)
- **Measures** — aggregatable columns (e.g., `COUNT(*)`, `SUM(amount)`), queried via `MEASURE()` syntax
- **Semantic metadata** — `display_name`, `comment`, `synonyms`, `format` on every column

This is the Ontology layer — it is how Genie knows what data means, not just what it is.

## 3. Metric View DDL Structure

```sql
CREATE OR REPLACE VIEW <catalog>.<schema>.<metric_view_name>
AS SELECT
  -- Dimensions (with semantic metadata)
  table_catalog,
  table_schema,
  assessment_date,
  -- Measures (with semantic metadata)
  COUNT(*) AS table_count,
  AVG(composite_score) AS avg_composite_score
FROM <catalog>.<schema>.<source_table>
GROUP BY table_catalog, table_schema, assessment_date
WITH COLUMNS (
  -- Dimensions
  table_catalog COMMENT 'Unity Catalog name',
  table_schema COMMENT 'Schema within the catalog',
  assessment_date COMMENT 'Date the readiness assessment was run',
  -- Measures
  table_count COMMENT 'Total number of tables assessed',
  avg_composite_score COMMENT 'Average composite readiness score (0-5 scale)'
);
```

### Column Metadata Properties

| Property | Purpose | Example |
|---|---|---|
| `COMMENT` | Business description of what the column means | `'Total number of tables assessed'` |
| `display_name` | Human-readable name for UI display | `'Average Composite Score'` |
| `synonyms` | Alternative names Genie should recognize | `'avg score, mean score, overall score'` |
| `format` | Display format pattern | `'#,##0.00'` for decimals, `'%'` for percentages |

### Applying Metadata After Creation

If the view already exists, apply metadata via `ALTER VIEW`:
```sql
ALTER VIEW <catalog>.<schema>.<metric_view_name>
ALTER COLUMN avg_composite_score SET METADATA ('display_name' = 'Average Composite Score');

ALTER VIEW <catalog>.<schema>.<metric_view_name>
ALTER COLUMN avg_composite_score SET METADATA ('synonyms' = 'avg score, mean score, overall score');

ALTER VIEW <catalog>.<schema>.<metric_view_name>
ALTER COLUMN avg_composite_score SET METADATA ('format' = '#,##0.00');
```

## 4. Privacy-Safe Metric View Design

When the source table contains PII or PHI, the metric view must be designed to eliminate re-identification risk:

### Rule 1: Never Include Direct Identifiers as Dimensions
- Do not expose `email`, `customer_id` (if unique), `ssn`, `phone` as dimensions.
- Replace with generalized groupings: `customer_segment` instead of `customer_id`, `region` instead of `zip_code`.

### Rule 2: Pre-Aggregate to k-Anonymity
- Ensure each row in the metric view represents at least k records (k ≥ 5 recommended).
- If a grouping would produce fewer than k rows, merge into a broader category or suppress.
- Example: instead of `COUNT(*)` per individual customer, aggregate per `customer_segment, month`.

### Rule 3: Exclude Quasi-Identifier Combinations
- Do not include combinations of quasi-identifiers that could re-identify individuals (e.g., `zip_code + date_of_birth + gender`).
- Use broader dimensions: `state` instead of `zip_code`, `age_band` instead of `date_of_birth`.

### Rule 4: Measures Over Raw Values
- Metric views should expose measures (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`), not raw column values.
- This ensures Genie can only return aggregated insights, never individual records.

```sql
-- PRIVACY-SAFE: Aggregated by segment, no individual-level data
CREATE OR REPLACE VIEW <catalog>.gold.customer_metrics
AS SELECT
  customer_segment,
  order_month,
  COUNT(DISTINCT customer_token) AS unique_customers,
  SUM(order_amount) AS total_revenue,
  AVG(order_amount) AS avg_order_value
FROM <catalog>.silver.orders
GROUP BY customer_segment, order_month
WITH COLUMNS (
  customer_segment COMMENT 'Customer tier (Bronze, Silver, Gold, Platinum)',
  order_month COMMENT 'Calendar month of the order (YYYY-MM)',
  unique_customers COMMENT 'Number of distinct customers in this segment/month',
  total_revenue COMMENT 'Total revenue for this segment/month',
  avg_order_value COMMENT 'Average order value for this segment/month'
);
```

## 5. Execution Workflow

1. **Identify the source table and its PII/PHI columns:** Run the pii-classifier skill or review existing tags.
2. **Determine dimensions:** Select non-PII grouping columns that are meaningful for business questions. Generalize quasi-identifiers.
3. **Determine measures:** Define aggregations that answer the business questions (counts, sums, averages).
4. **Check k-anonymity:** Verify that each dimension combination groups at least k records. Adjust granularity if needed.
5. **Generate DDL:** Produce the `CREATE OR REPLACE VIEW ... WITH COLUMNS (...)` statement with full semantic metadata.
6. **Apply metadata:** Generate `ALTER VIEW ... SET METADATA` for display_name, synonyms, and format if not inline.
7. **Produce Genie Space integration guidance:** List the metric view name and recommended SQL examples for the space instructions.

## 6. Reference Output Template

### 📐 Metric View: [catalog].[schema].[view_name]

**Source Table:** [catalog.schema.table]
**Privacy Assessment:** [Safe / k-anonymized (k=X) / Requires column masks]

**Dimensions:**

| Column | Source Column | Display Name | Synonyms | Comment |
|---|---|---|---|---|
| | | | | |

**Measures:**

| Column | Aggregation | Display Name | Synonyms | Format | Comment |
|---|---|---|---|---|---|
| | | | | | |

**DDL:**
```sql
CREATE OR REPLACE VIEW ...;
```

**Metadata Statements:**
```sql
ALTER VIEW ... SET METADATA ...;
```

**Genie Space SQL Examples:**
```sql
-- Example: What is the average readiness score by catalog?
SELECT table_catalog, MEASURE(avg_composite_score) AS avg_score
FROM <catalog>.<schema>.<view_name>
GROUP BY table_catalog
ORDER BY avg_score DESC;
```

**Privacy Notes:**
- [What PII columns were excluded and why]
- [What generalization was applied to quasi-identifiers]
- [k-anonymity verification result]

## 7. Guardrails
- **Never include raw direct identifiers as dimensions in a metric view** — always generalize or exclude.
- **Always include semantic metadata on every column** — without it, Genie cannot discover or understand the view.
- **Verify k-anonymity before publishing** — if any dimension combination groups fewer than 5 records, adjust the granularity.
- **Use `MEASURE()` syntax in Genie Space examples** — this teaches Genie users the correct query pattern for metric views.
- **Publish metric views in a dedicated gold/schema** — separate from raw and silver layers for clear access boundaries.
