---
name: retention-compliance-auditor
description: Trigger this skill when a user asks to audit, check, or enforce data retention policies for PII, PHI, or regulated data against GDPR, CCPA, or HIPAA retention timelines. Also triggers when the user needs to identify stale data, plan a purge or vacuum, or check legal-hold status.
---

# Skill: Data Retention Compliance Auditor

## 1. Intent & Trigger
- Invoke this skill when the user asks to check retention compliance, identify data past its retention period, or plan a data purge.
- Also invoke when the user asks about legal holds, data minimization lifecycle, or GDPR/CCPA deletion timelines.
- Do NOT invoke for general `VACUUM` or storage optimization — only when retention regulations are in scope.

## 2. Regulatory Retention Baselines

| Regulation | Data Category | Default Retention | Key Constraint |
|---|---|---|---|
| **GDPR Art. 5(1)(e)** | Personal data (general) | Only as long as necessary for the purpose | Storage limitation principle — no indefinite retention |
| **GDPR Art. 17** | Data subject erasure request | Immediate upon valid request (within 1 month) | Right to be forgotten |
| **CCPA/CPRA §1798.105** | Consumer personal information | Until consumer deletion request fulfilled | 45-day response window (extendable by 45 days) |
| **HIPAA** | PHI — designated record set | Minimum necessary; retain 6 years for audit logs (45 CFR 164.530(j)) | Retention of audit logs is separate from PHI itself |
| **PCI-DSS** | Cardholder data | Minimize storage; truncate/encrypt if retained | Requirement 3.1 — retain only what's needed |
| **EU Data Retention** | Telecommunications metadata | Varies by member state; ECJ invalidated mass retention | Check national law |

## 3. Execution Workflow

### Step 1: Identify Retention-Relevant Tables
Query `system.information_schema.tables` for tables containing PII/PHI (use tags if applied, or scan column names per pii-classifier skill):
```sql
SELECT table_catalog, table_schema, table_name, table_type
FROM system.information_schema.tables
WHERE table_schema = '<schema>'
ORDER BY table_catalog, table_schema, table_name;
```

### Step 2: Check Table Properties for Retention Metadata
```sql
SHOW TBLPROPERTIES <catalog>.<schema>.<table>;
```
Look for properties like `retention.days`, `retention.policy`, `legal_hold`, `data_category`.

### Step 3: Identify the Time Column
Determine which column represents the data's effective date (e.g., `created_at`, `updated_at`, `event_timestamp`, `_ingested_at`). If unsure, check the table's Delta history:
```sql
DESCRIBE HISTORY <catalog>.<schema>.<table>;
```

### Step 4: Calculate Stale Data Volume
Count rows past the retention threshold:
```sql
SELECT COUNT(*) AS stale_rows,
       MIN(<time_column>) AS oldest_record,
       MAX(<time_column>) AS newest_stale_record
FROM <catalog>.<schema>.<table>
WHERE <time_column> < DATE_SUB(CURRENT_DATE(), <retention_days>);
```

### Step 5: Legal-Hold Check
Before generating any purge SQL, verify no legal hold exists on the table:
- Check for a `legal_hold = true` table property.
- If a retention catalog table exists (e.g., `<catalog>.governance.retention_policies`), cross-reference the table name.
- If any legal hold is active, **halt and flag** — do not generate purge SQL.

### Step 6: Generate Purge Plan
Produce a safe, staged purge plan — never a blind `DELETE`:

```sql
-- Stage 1: Archive stale data to a retention volume (optional but recommended)
CREATE TABLE IF NOT EXISTS <catalog>.<schema>.<table>_archive
AS SELECT * FROM <catalog>.<schema>.<table>
WHERE <time_column> < DATE_SUB(CURRENT_DATE(), <retention_days>);

-- Stage 2: Verify archive row count matches
SELECT COUNT(*) FROM <catalog>.<schema>.<table>_archive;
-- Compare against the stale_rows count from Step 4

-- Stage 3: Delete stale rows (requires user sign-off)
DELETE FROM <catalog>.<schema>.<table>
WHERE <time_column> < DATE_SUB(CURRENT_DATE(), <retention_days>);

-- Stage 4: Vacuum to reclaim storage (after retention check window)
VACUUM <catalog>.<schema>.<table> RETAIN 7 HOURS;
```

## 4. Reference Output Template

### ⏳ Retention Compliance Audit: [catalog].[schema].[table]

**Table Retention Profile:**

| Property | Value |
|---|---|
| Data Category | [PII / PHI / Financial / Non-personal] |
| Applicable Regulation | [GDPR / CCPA / HIPAA / PCI-DSS] |
| Retention Policy | [N days / Not set] |
| Time Column | [column name] |
| Legal Hold | [Active / None] |

**Stale Data Assessment:**

| Metric | Value |
|---|---|
| Total Rows | [count] |
| Stale Rows (past retention) | [count] |
| % Stale | [percentage] |
| Oldest Record | [date] |

**Compliance Status:** [✅ Compliant / ⚠️ Action Required / 🚫 Legal Hold — Do Not Purge]

**Recommended Actions:**
- [Bulleted list of actions: set retention property, purge stale data, apply retention tag, etc.]

**Purge Plan SQL:** *(requires explicit user sign-off before execution)*
```sql
-- Archive, verify, delete, vacuum sequence
```

## 5. Guardrails
- **Never auto-execute `DELETE` or `VACUUM`** on regulated data — always present the plan and wait for explicit sign-off.
- **Always check for legal holds first** — a legal hold overrides any retention-based deletion.
- **Always archive before deleting** — deleted Delta rows are recoverable only within the log retention window; archiving provides a longer-term safety net.
- **Set retention metadata as table properties** to make future audits self-documenting:
  ```sql
  ALTER TABLE <catalog>.<schema>.<table>
  SET TBLPROPERTIES (
    'retention.days' = '365',
    'retention.regulation' = 'GDPR',
    'data.category' = 'pii'
  );
  ```
- **Delta log retention must exceed data retention** — set `delta.logRetentionDuration` to be longer than the retention period so CDF and time travel remain available for audit:
  ```sql
  ALTER TABLE <catalog>.<schema>.<table>
  SET TBLPROPERTIES ('delta.logRetentionDuration' = 'interval 400 days');
  ```
