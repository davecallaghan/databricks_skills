---
name: masking-policy-author
description: Trigger this skill when a user asks to create, generate, or apply Unity Catalog column masks, row filters, or tag-based access control (ABAC) policies for PII, PHI, or regulated data. Also triggers when the user needs GDPR, CCPA, or HIPAA enforcement controls as executable SQL.
---

# Skill: Unity Catalog Masking & Row Filter Policy Author

## 1. Intent & Trigger
- Invoke this skill when the user asks to create column masks, row filters, or dynamic views to protect PII/PHI data in Unity Catalog.
- Also invoke when the user asks for role-based redaction, jurisdictional filtering, or tag-driven access control policies.
- Do NOT invoke for general SQL help or non-governance data quality constraints.

## 2. Policy Patterns

### Pattern A: Column Mask — Role-Based Redaction

Use when a column contains PII/PHI that should be partially or fully hidden based on the caller's group membership.

```sql
-- Create the mask function in a governance schema
CREATE OR REPLACE FUNCTION <catalog>.<schema>.mask_email(column_value STRING)
RETURNS STRING
RETURN CASE
  WHEN is_member('<authorized_role>') THEN column_value
  WHEN is_member('<analyst_role>') THEN CONCAT(LEFT(column_value, 2), '****@', SPLIT(column_value, '@')[1])
  ELSE NULL
END;

-- Apply the mask to the table column
ALTER TABLE <catalog>.<schema>.<table>
ALTER COLUMN <email_column>
SET MASK <catalog>.<schema>.mask_email;
```

**GDPR alignment:** Supports data minimization (Art. 5(1)(c)) and access limitation (Art. 5(1)(f)).
**HIPAA alignment:** Supports minimum necessary access (45 CFR 164.502(b)).

### Pattern B: Column Mask — Hash-Based Pseudonymization

Use when analysts need join capability without seeing raw identifiers.

```sql
CREATE OR REPLACE FUNCTION <catalog>.<schema>.hash_pii(column_value STRING)
RETURNS STRING
RETURN CASE
  WHEN is_member('<authorized_role>') THEN column_value
  ELSE SHA2(column_value, 256)
END;

ALTER TABLE <catalog>.<schema>.<table>
ALTER COLUMN <ssn_column>
SET MASK <catalog>.<schema>.hash_pii;
```

**GDPR alignment:** Pseudonymization under Art. 4(5) — data is still personal data but risk is reduced.

### Pattern C: Column Mask — Partial Redaction for Quasi-Identifiers

Use for zip codes, dates of birth, or other quasi-identifiers that need generalization for k-anonymity.

```sql
CREATE OR REPLACE FUNCTION <catalog>.<schema>.mask_zip(column_value STRING)
RETURNS STRING
RETURN CASE
  WHEN is_member('<authorized_role>') THEN column_value
  ELSE CONCAT(SUBSTRING(column_value, 1, 3), '**')
END;

ALTER TABLE <catalog>.<schema>.<table>
ALTER COLUMN <zip_column>
SET MASK <catalog>.<schema>.mask_zip;
```

### Pattern D: Row Filter — Jurisdictional Isolation

Use when data subjects in different jurisdictions must only be visible to authorized regional teams.

```sql
CREATE OR REPLACE FUNCTION <catalog>.<schema>.filter_eu_data(jurisdiction STRING)
RETURNS BOOLEAN
RETURN is_member('<eu_authorized_group>') OR jurisdiction != 'EU';

ALTER TABLE <catalog>.<schema>.<table>
SET ROW FILTER <catalog>.<schema>.filter_eu_data(<jurisdiction_column>);
```

**GDPR alignment:** Supports cross-border transfer restrictions (Chapter V) and access limitation.

### Pattern E: Row Filter — Tenant Isolation

Use for multi-tenant SaaS tables where each tenant's data must be isolated.

```sql
CREATE OR REPLACE FUNCTION <catalog>.<schema>.filter_tenant(tenant_id STRING)
RETURNS BOOLEAN
RETURN is_member(CONCAT('tenant_', tenant_id, '_access')) OR is_member('<super_admin_group>');

ALTER TABLE <catalog>.<schema>.<table>
SET ROW FILTER <catalog>.<schema>.filter_tenant(<tenant_id_column>);
```

### Pattern F: Dynamic View — Composed Mask + Filter

Use when multiple masks and filters must be composed on the same table, or when the base table should never be granted directly.

```sql
CREATE OR REPLACE VIEW <catalog>.<schema>.<table>_secure
AS SELECT
  id,
  <catalog>.<schema>.mask_email(email) AS email,
  <catalog>.<schema>.hash_pii(ssn) AS ssn,
  <catalog>.<schema>.mask_zip(zip_code) AS zip_code,
  -- non-PII columns pass through
  order_date,
  amount
FROM <catalog>.<schema>.<table>
WHERE <catalog>.<schema>.filter_eu_data(jurisdiction);

-- Grant on the view only, never the base table
GRANT SELECT ON VIEW <catalog>.<schema>.<table>_secure TO `<analyst_group>`;
```

## 3. Execution Workflow

1. **Identify columns to protect:** Use the pii-classifier skill output or the user's specification to determine which columns need masks or filters.
2. **Determine access tiers:** Ask the user (or infer from context) which groups need full access, partial access, and no access.
3. **Generate functions:** Produce `CREATE OR REPLACE FUNCTION` statements in a dedicated governance schema (e.g., `<catalog>.governance.masks`).
4. **Apply policies:** Produce `ALTER TABLE ... SET MASK` / `SET ROW FILTER` statements.
5. **Verify:** Suggest a verification query using `current_user()` impersonation or a test role to confirm masks apply correctly.

## 4. Reference Output Template

### 🛡️ Policy Authoring: [catalog].[schema].[table]

**Protected Columns:**

| Column | Category | Policy Type | Access Tiers |
|---|---|---|---|
| | | | |

**Generated SQL:**
```sql
-- 1. Mask functions
CREATE OR REPLACE FUNCTION ...;

-- 2. Apply masks
ALTER TABLE ...;

-- 3. Grant (on view, not base table)
GRANT ...;
```

**Regulatory Coverage:**
- [Which GDPR/CCPA/HIPAA requirements are satisfied by these policies]

**Verification Query:**
```sql
-- Run as a test user to confirm masking
SELECT ...;
```

## 5. Guardrails
- **Never grant SELECT on a base table containing PII/PHI directly** — always route through a masked view or apply column masks first.
- **Always create mask/filter functions in a dedicated governance schema** — not alongside business data schemas.
- **Always include a default-deny branch** in mask functions (`ELSE NULL`) — the safest default is no access.
- **Do not auto-execute `ALTER TABLE ... SET MASK` or `SET ROW FILTER`** — present the SQL and wait for user sign-off.
- **Test before production:** Always suggest running verification queries as a non-privileged user to confirm masks are active.
