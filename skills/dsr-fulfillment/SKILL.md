---
name: dsr-fulfillment
description: Trigger this skill when a user asks to handle, process, or fulfill a Data Subject Request (DSR) under GDPR or CCPA — including access, erasure, portability, or rectification requests. Also triggers when building a DSR data map, generating deletion/pseudonymization plans, tracking response deadlines, or producing audit trails for compliance evidence.
---

# Skill: Data Subject Request (DSR) Fulfillment

## 1. Intent & Trigger
- Invoke this skill when the user asks to fulfill a data subject's request to access, delete, correct, or export their personal data.
- Also invoke when building or maintaining a DSR data map (which tables/columns contain personal data and how they link to a subject).
- Also invoke when tracking DSR response deadlines or generating compliance audit trails.
- Do NOT invoke for general data quality or retention tasks — this skill is specifically for individual data subject requests.

## 2. Regulatory Context

| Regulation | Right | Response Deadline | Key Constraint |
|---|---|---|---|
| **GDPR Art. 15** | Access — obtain copy of personal data + processing info | 1 month (extendable by 2 months for complex requests) | Must provide data in structured, commonly used, machine-readable format |
| **GDPR Art. 16** | Rectification — correct inaccurate data | 1 month | Must notify recipients of corrected data |
| **GDPR Art. 17** | Erasure — right to be forgotten | 1 month ("without undue delay") | Exemptions: legal obligation, freedom of expression, public interest |
| **GDPR Art. 20** | Portability — receive data in structured format + transmit to another controller | 1 month | Applies to data provided by the subject, processed by automated means |
| **CCPA §1798.100** | Know — categories and specific pieces of PI collected | 45 days (extendable by 45 days) | Must be verifiable consumer request |
| **CCPA §1798.105** | Delete — remove PI | 45 days (extendable by 45 days) | Must notify consumer of deletion and recipients |
| **CCPA §1798.106** | Correct — amend inaccurate PI | 45 days (extendable by 45 days) | Must notify recipients of correction |
| **CCPA §1798.110** | Know — categories of sources, business purpose, third parties shared with | 45 days | Must provide in a readily useable format |

## 3. DSR Lifecycle

```
Request Received → Identity Verification → Data Discovery → Fulfillment Action → Verification → Audit Trail → Deadline Tracking
```

### Step 1: Identity Verification
Before fulfilling any DSR, verify the data subject's identity. Do not proceed without explicit user confirmation that identity has been verified through the organization's established process.

### Step 2: Data Discovery (The Data Map)

The foundation of every DSR is knowing where personal data lives. Use the data map to locate all tables/columns containing the subject's data.

**Query the PII inventory** (built by the pii-classifier skill or maintained as a governance table):
```sql
SELECT 
  table_catalog, table_schema, table_name, column_name, 
  tag_key, join_key
FROM <catalog>.governance.pii_inventory
WHERE tag_key LIKE 'pii.%' OR tag_key LIKE 'phi.%'
ORDER BY table_catalog, table_schema, table_name;
```

**If no PII inventory exists**, build one dynamically:
1. Scan `system.information_schema.columns` for the target schema(s).
2. Run the pii-classifier skill to identify PII/PHI columns.
3. Identify the join key that links each table to the data subject (e.g., `customer_id`, `user_id`, `email`).

### Step 3: Fulfillment by Request Type

#### Access Request (GDPR Art. 15 / CCPA §1798.100)

Find all records for the subject across all PII tables:
```sql
-- Step 1: Identify the subject's identifier
SELECT customer_id FROM <catalog>.<schema>.customers 
WHERE email = '<subject_email>';  -- use the verified identifier

-- Step 2: Find all records linked to this subject across tables
SELECT 'customers' AS source_table, 'personal_info' AS data_type, 
       customer_id, full_name, email, phone, address AS data_fields
FROM <catalog>.<schema>.customers 
WHERE customer_id = '<subject_id>'

UNION ALL

SELECT 'orders' AS source_table, 'transaction_history' AS data_type,
       customer_id, order_id, order_date, amount, product_id AS data_fields
FROM <catalog>.<schema>.orders
WHERE customer_id = '<subject_id>'

UNION ALL
-- ... continue for all PII tables identified in the data map
;
```

Export the result set to JSON or Parquet for delivery to the data subject:
```python
df = spark.sql("""<above query>""")
df.write.mode("overwrite").json("/Volumes/<catalog>/<schema>/dsr_exports/<request_id>/")
```

#### Erasure Request (GDPR Art. 17 / CCPA §1798.105)

**Preferred approach: Pseudonymization** (GDPR Art. 4(5) — reduces risk while preserving referential integrity):

```sql
-- Step 1: Generate a pseudonymization token
SELECT SHA2(CONCAT(customer_id, request_id), 256) AS token;

-- Step 2: Replace identifiers with token across all affected tables
-- This preserves join paths and analytics while removing identity
UPDATE <catalog>.<schema>.customers 
SET full_name = NULL,
    email = NULL,
    phone = NULL,
    address = NULL,
    customer_id = '<pseudonym_token>'
WHERE customer_id = '<subject_id>';

UPDATE <catalog>.<schema>.orders
SET customer_id = '<pseudonym_token>'
WHERE customer_id = '<subject_id>';

-- Step 3: Verify pseudonymization
SELECT COUNT(*) AS remaining_identifiable 
FROM <catalog>.<schema>.customers 
WHERE email = '<subject_email>' OR phone = '<subject_phone>';
-- Expected: 0
```

**Alternative: Hard deletion** (only when pseudonymization is not viable — requires explicit user sign-off):
```sql
-- Hard delete requires explicit sign-off per .assistant_instructions.md guardrails
DELETE FROM <catalog>.<schema>.customers WHERE customer_id = '<subject_id>';
DELETE FROM <catalog>.<schema>.orders WHERE customer_id = '<subject_id>';
-- ... for all tables containing the subject's data

VACUUM <catalog>.<schema>.customers RETAIN 7 HOURS;
VACUUM <catalog>.<schema>.orders RETAIN 7 HOURS;
```

#### Portability Request (GDPR Art. 20 / CCPA §1798.110)

Export only the data the subject provided directly (not derived/inferred data):
```sql
SELECT customer_id, full_name, email, phone, address, 
       order_date, product_id, amount
FROM <catalog>.<schema>.customers c
JOIN <catalog>.<schema>.orders o ON c.customer_id = o.customer_id
WHERE c.customer_id = '<subject_id>';
```

Write to a portable format:
```python
df = spark.sql("<above query>")
df.coalesce(1).write.mode("overwrite") \
  .option("compression", "gzip") \
  .json(f"/Volumes/<catalog>/<schema>/dsr_exports/{request_id}/portability/")
```

#### Rectification Request (GDPR Art. 16 / CCPA §1798.106)

Correct inaccurate data and log the change:
```sql
-- Step 1: Capture before-state for audit
CREATE TABLE IF NOT EXISTS <catalog>.governance.dsr_audit_log (
  request_id STRING, subject_id STRING, table_name STRING, 
  column_name STRING, old_value STRING, new_value STRING, 
  action STRING, performed_by STRING, performed_at TIMESTAMP
);

-- Step 2: Log the old value
INSERT INTO <catalog>.governance.dsr_audit_log
SELECT '<request_id>', '<subject_id>', 'customers', 'address',
       address, '<corrected_value>', 'RECTIFY', 
       current_user(), current_timestamp()
FROM <catalog>.<schema>.customers 
WHERE customer_id = '<subject_id>';

-- Step 3: Apply the correction
UPDATE <catalog>.<schema>.customers
SET address = '<corrected_value>'
WHERE customer_id = '<subject_id>';
```

### Step 4: Audit Trail

Every DSR action must be recorded in a governance audit log:
```sql
INSERT INTO <catalog>.governance.dsr_audit_log
VALUES (
  '<request_id>', '<subject_id>', '<table>', '<column>',
  '<old_value>', '<new_value>', '<action>',
  current_user(), current_timestamp()
);
```

Minimum audit fields: request_id, subject_id, request_type (access/erasure/portability/rectification), tables_affected, rows_affected, action_taken (export/pseudonymize/delete/correct), performed_by, performed_at, deadline, status.

### Step 5: Deadline Tracking

Maintain a DSR tracking table:
```sql
CREATE TABLE IF NOT EXISTS <catalog>.governance.dsr_tracker (
  request_id STRING, regulation STRING, request_type STRING,
  subject_id STRING, received_date DATE, deadline_date DATE,
  status STRING, fulfilled_date DATE, fulfilled_by STRING,
  notes STRING
);

INSERT INTO <catalog>.governance.dsr_tracker
VALUES (
  '<request_id>', 'GDPR', 'erasure', '<subject_id>',
  DATE '<received_date>', DATE_ADD(DATE '<received_date>', 30),
  'IN_PROGRESS', NULL, NULL, NULL
);
```

## 4. Reference Output Template

### 📋 DSR Fulfillment Report: [Request ID]

**Request Details:**

| Field | Value |
|---|---|
| Request ID | [ID] |
| Regulation | [GDPR / CCPA] |
| Request Type | [Access / Erasure / Portability / Rectification] |
| Subject ID | [masked/hashed] |
| Received Date | [date] |
| Deadline | [date — N days remaining] |
| Status | [In Progress / Complete / Overdue] |

**Data Discovery Results:**

| Table | PII Columns Found | Join Key | Row Count for Subject |
|---|---|---|---|
| | | | |

**Fulfillment Plan:**
- [Action type and approach — pseudonymization vs hard delete, export path, etc.]
- [Tables affected and estimated row counts]
- [Verification queries]

**Generated SQL:** *(erasure/deletion requires explicit user sign-off)*
```sql
-- Fulfillment SQL
```

**Audit Trail Entry:**
```sql
INSERT INTO <catalog>.governance.dsr_audit_log ...;
```

**Deadline Status:** [✅ Within deadline / ⚠️ Approaching deadline / 🚫 Overdue]

## 5. Guardrails
- **Never fulfill a DSR without identity verification** — confirm with the user that the subject's identity has been verified through the organization's established process.
- **Never auto-execute erasure SQL** — always present the pseudonymization or deletion plan and wait for explicit user sign-off.
- **Prefer pseudonymization over hard deletion** — it preserves referential integrity and analytics utility while removing identifiability.
- **Always produce an audit trail** — every DSR action (including access queries) must be logged with who/what/when.
- **Check for legal holds before erasure** — cross-reference with the retention-compliance-auditor skill's legal-hold check.
- **Track all deadlines** — GDPR defaults to 1 month, CCPA to 45 days. Flag requests approaching their deadline.
- **Notify downstream recipients** — for rectification and erasure, identify and document all downstream systems that received the subject's data.
- **Mask the subject's identifiers in the audit log itself** — the audit trail should not create new PII exposure.
