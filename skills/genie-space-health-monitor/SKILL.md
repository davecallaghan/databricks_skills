---
name: genie-space-health-monitor
description: Trigger this skill when a user asks to monitor, check, or audit the health of a Genie Space — including query success rates, latency, table freshness, instruction drift, permission changes, or PII exposure events. Also triggers when setting up ongoing Genie Space monitoring or investigating degraded Genie Space performance.
---

# Skill: Genie Space Health Monitor

## 1. Intent & Trigger
- Invoke this skill when the user asks to check the health, status, or performance of a Genie Space.
- Also invoke when investigating why Genie Space queries are failing, slow, or returning unexpected results.
- Also invoke when setting up ongoing monitoring or alerting for Genie Spaces.
- Also invoke when auditing whether a Genie Space has been exposed to PII leakage.
- Do NOT invoke for general warehouse monitoring — this skill is Genie Space-specific.

## 2. Health Dimensions

| # | Dimension | What It Measures | Healthy | Unhealthy |
|---|---|---|---|---|
| 1 | **Query Success Rate** | % of Genie-generated queries that execute without error | ≥ 95% | < 80% |
| 2 | **Query Latency** | P50/P95 execution time of Genie queries | P50 < 5s, P95 < 30s | P50 > 15s, P95 > 60s |
| 3 | **Table Freshness** | How current the underlying tables are vs. expected cadence | Within SLA | Stale beyond SLA |
| 4 | **Instruction Drift** | Whether space instructions still match the underlying schema | No drift | Instructions reference removed/renamed columns |
| 5 | **Permission Integrity** | Whether run-as principal still has correct grants | Grants intact | Grants revoked or changed unexpectedly |
| 6 | **PII Exposure Risk** | Whether any query accessed PII columns without masking | No exposure events | Unmasked PII access detected |
| 7 | **User Adoption** | Query volume and active users over time | Growing or stable | Declining or zero activity |

## 3. Monitoring Queries (System Table Based)

### Query Success Rate & Latency

Genie Space queries execute on the assigned SQL warehouse. Query history is available in `system.query.history`:

```sql
SELECT 
  DATE(start_time) AS query_date,
  COUNT(*) AS total_queries,
  SUM(CASE WHEN status = 'FINISHED' THEN 1 ELSE 0 END) AS successful_queries,
  ROUND(100.0 * SUM(CASE WHEN status = 'FINISHED' THEN 1 ELSE 0 END) / COUNT(*), 1) AS success_rate_pct,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY execution_duration_ms) AS p50_latency_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY execution_duration_ms) AS p95_latency_ms
FROM system.query.history
WHERE start_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND warehouse_id = '<space_warehouse_id>'
  AND statement_text NOT LIKE 'DESCRIBE%'  -- exclude metadata queries
  AND statement_text NOT LIKE 'SHOW%'       -- exclude metadata queries
GROUP BY DATE(start_time)
ORDER BY query_date DESC;
```

### Failed Query Analysis

```sql
SELECT 
  start_time,
  statement_text,
  status,
  error_message,
  execution_duration_ms
FROM system.query.history
WHERE start_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND warehouse_id = '<space_warehouse_id>'
  AND status != 'FINISHED'
ORDER BY start_time DESC
LIMIT 20;
```

**Common failure patterns:**
- `TABLE_OR_VIEW_NOT_FOUND` → Table was dropped or renamed; instruction drift.
- `COLUMN_NOT_FOUND` → Column was dropped/renamed; instruction drift.
- `PERMISSION_DENIED` → Run-as principal lost grant; permission integrity failure.
- `ANALYSIS_EXCEPTION` → Genie generated invalid SQL; may need instruction updates or relationship docs.

### Table Freshness Check

```sql
-- Check the most recent data in each table backing the Genie Space
SELECT 
  '<catalog>.<schema>.<table>' AS table_name,
  (SELECT MAX(<time_column>) FROM <catalog>.<schema>.<table>) AS latest_record,
  CURRENT_TIMESTAMP() AS check_time,
  DATEDIFF(CURRENT_TIMESTAMP(), (SELECT MAX(<time_column>) FROM <catalog>.<schema>.<table>)) AS days_since_latest
;
-- Repeat for each table in the space
```

Flag as unhealthy if `days_since_latest` exceeds the expected update cadence (e.g., > 2 days for a daily-refresh table).

### Instruction Drift Detection

Compare the columns referenced in Genie Space instructions/SQL examples against the actual schema:
```sql
-- Get all columns referenced in space instructions (parse from instruction text)
-- Compare against actual columns
SELECT 
  c.table_name, c.column_name,
  CASE WHEN c.column_name IS NULL THEN 'MISSING' ELSE 'PRESENT' END AS schema_status
FROM <catalog>.governance.genie_space_instructions i
LEFT JOIN system.information_schema.columns c
  ON i.referenced_column = c.column_name
  AND i.referenced_table = c.table_name
WHERE i.space_id = '<space_id>'
  AND c.column_name IS NULL;
-- Any MISSING results = instruction drift
```

If no instruction tracking table exists, manually compare instruction SQL examples against `DESCRIBE TABLE` output for each table in the space.

### Permission Integrity Check

```sql
-- Verify the run-as principal still has SELECT on all space tables
SELECT 
  table_catalog, table_schema, table_name,
  principal, action_type, direct_permissions
FROM system.access.table_permissions
WHERE table_catalog = '<catalog>'
  AND table_schema = '<schema>'
  AND principal = '<run_as_principal>'
  AND action_type = 'SELECT'
ORDER BY table_name;
-- Compare against the list of tables in the space
-- Any missing tables = permission integrity failure
```

### PII Exposure Detection

```sql
-- Detect queries that accessed PII columns (if PII tags are applied)
SELECT 
  q.start_time,
  q.statement_text,
  q.user_name,
  c.column_name,
  c.tag_key AS pii_category
FROM system.query.history q
-- This requires parsing column references from statement_text and cross-referencing with PII tags
-- In practice, use a UDF or Python to parse SQL column references
WHERE q.start_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND q.warehouse_id = '<space_warehouse_id>'
  AND q.status = 'FINISHED'
-- Filter to queries that reference columns tagged as PII
ORDER BY q.start_time DESC;
```

**Simplified approach:** If column masks are properly applied, any successful query that includes PII columns will return masked values — exposure is mitigated by design. The monitor should verify that masks are still active:
```sql
-- Check if masks are still applied to PII columns
SHOW COLUMN MASKS ON <catalog>.<schema>.<table>;
-- Any PII column without a mask = exposure risk
```

### User Adoption Trends

```sql
SELECT 
  DATE(start_time) AS activity_date,
  COUNT(DISTINCT user_name) AS active_users,
  COUNT(*) AS query_count
FROM system.query.history
WHERE start_time >= DATE_SUB(CURRENT_DATE(), 30)
  AND warehouse_id = '<space_warehouse_id>'
  AND status = 'FINISHED'
GROUP BY DATE(start_time)
ORDER BY activity_date DESC;
```

## 4. Health Score Calculation

Each dimension scored 0-2:
- **0** = Unhealthy (action required)
- **1** = Degraded (monitor)
- **2** = Healthy (no action)

**Overall Health Score** = Σ(dimension scores) / (dimensions × 2) × 100

| Score Range | Status | Action |
|---|---|---|
| 85-100% | 🟢 Healthy | Continue monitoring |
| 60-84% | 🟡 Degraded | Investigate failing dimensions |
| < 60% | 🔴 Unhealthy | Immediate remediation required |

## 5. Reference Output Template

### 🩺 Genie Space Health Report: [Space Name]

**Overview:**

| Metric | Value |
|---|---|
| Space ID | [ID] |
| Warehouse | [warehouse name/ID] |
| Run-As Principal | [principal] |
| Tables in Space | [count] |
| Monitoring Period | [last 7 days] |
| Overall Health Score | [XX%] — [🟢/🟡/🔴] |

**Dimension Scores:**

| # | Dimension | Score | Status | Key Finding |
|---|---|---|---|---|
| 1 | Query Success Rate | [0-2] | [🟢/🟡/🔴] | [e.g., '93% success — 7 failures from missing column'] |
| 2 | Query Latency | [0-2] | [🟢/🟡/🔴] | [e.g., 'P50=4s, P95=22s — within thresholds'] |
| 3 | Table Freshness | [0-2] | [🟢/🟡/🔴] | [e.g., 'orders table 3 days stale — SLA is daily'] |
| 4 | Instruction Drift | [0-2] | [🟢/🟡/🔴] | [e.g., '2 SQL examples reference removed columns'] |
| 5 | Permission Integrity | [0-2] | [🟢/🟡/🔴] | [e.g., 'All grants intact'] |
| 6 | PII Exposure Risk | [0-2] | [🟢/🟡/🔴] | [e.g., 'All PII columns masked — no exposure detected'] |
| 7 | User Adoption | [0-2] | [🟢/🟡/🔴] | [e.g., '12 active users, 340 queries/week — stable'] |

**Failed Query Analysis (Top 5):**

| Time | Error | Query Fragment | Likely Cause |
|---|---|---|---|
| | | | |

**Remediation Actions (prioritized):**
1. [Action — dimension affected — effort estimate]
2. ...
3. ...

**Recommended Monitoring Cadence:**
- Daily: query success rate + latency (automated)
- Weekly: table freshness + instruction drift + permission integrity
- Monthly: full health audit + PII exposure review + adoption trends

## 6. Alerting Recommendations

Set up alerts for critical health degradations:
- **Query success rate < 80%** over a 1-hour window — immediate alert.
- **PII column mask removed** from any space table — immediate alert.
- **Run-as principal grant revoked** — immediate alert.
- **Table freshness > 2× expected SLA** — same-day alert.
- **Instruction drift detected** (column referenced in instructions no longer exists) — weekly alert.

## 7. Guardrails
- **Always use system tables for evidence** — never rely on manual observation or assumptions about Genie Space health.
- **PII exposure checks are the highest-priority dimension** — if masks are removed or PII columns are accessed without masking, flag as 🔴 immediately.
- **Differentiate Genie queries from manual queries** — filter `system.query.history` by the space's warehouse ID and exclude `DESCRIBE`/`SHOW` metadata queries to get an accurate success rate.
- **Instruction drift is a silent killer** — a space can appear healthy (queries succeed) while instructions are stale, leading to incorrect Genie-generated SQL. Always check drift even when other dimensions look fine.
- **Monitor the run-as principal quarterly** — service principal credentials and grants can change silently. Verify that the principal still exists and has the expected permissions.
- **Do not auto-remediate permission or mask issues** — present findings and wait for user sign-off before applying fixes, per .assistant_instructions.md governance guardrails.
