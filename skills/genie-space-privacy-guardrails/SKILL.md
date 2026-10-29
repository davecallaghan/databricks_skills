---
name: genie-space-privacy-guardrails
description: Trigger this skill when a user asks to create, configure, or review a Genie Space that includes tables containing PII, PHI, or regulated data. Also triggers when setting up Genie Space instructions, selecting tables/columns for a space, or configuring run-as permissions for privacy-sensitive datasets.
---

# Skill: Genie Space Privacy Guardrails

## 1. Intent & Trigger
- Invoke this skill when the user asks to create or configure a Genie Space on any table that may contain PII, PHI, or regulated data.
- Also invoke when reviewing an existing Genie Space for privacy exposure or recommending table/column exclusion.
- Do NOT invoke for general Genie Space setup on non-regulated data — only when privacy/governance is in scope.

## 2. Privacy Risk in Genie Spaces

Genie Spaces expose tables to natural-language querying. Without guardrails, a user could ask Genie to `SELECT email, ssn FROM customers` and receive raw PII. The risk is amplified because Genie generates and executes SQL on behalf of the user — the user may not even know which columns are sensitive.

### Attack Surface
- **Direct table access:** Genie runs as a service principal; if the principal has SELECT on the base table, all columns are accessible.
- **Instruction leakage:** Genie Space instructions and SQL examples may inadvertently reveal column names or data patterns.
- **Derived exposure:** Even if PII columns are excluded from the space, Genie may infer their existence from schema metadata or join paths.

## 3. Guardrail Framework

### Guardrail 1: Table Selection — Prefer Derived Over Raw

Never add a raw table containing direct identifiers or PHI directly to a Genie Space. Instead:
- **Prefer metric views** (see uc-metric-view-author skill) that pre-aggregate to a level where individual records are not identifiable.
- **Prefer dynamic views** with column masks applied so PII is redacted at query time.
- **If raw tables must be used**, ensure column masks and row filters are already applied via UC policies (see masking-policy-author skill) before the table is added to the space.

**Decision tree:**
```
Table contains PII/PHI?
  → YES: Can it be pre-aggregated into a metric view?
    → YES: Use metric view in Genie Space
    → NO: Can column masks be applied?
      → YES: Apply masks, then add table to space
      → NO: DO NOT add to Genie Space — flag as ineligible
  → NO: Safe to add directly
```

### Guardrail 2: Instruction-Level Privacy Disclaimers

Include explicit privacy instructions in the Genie Space's instructions block:

```
## Privacy Guardrails (Mandatory)
- This space contains data subject to GDPR and/or CCPA/CPRA regulations.
- Never return raw PII columns (email, phone, SSN, address, date of birth) in query results.
- If a user requests personal data about a specific individual, decline and explain that individual-level data is not available through this space.
- Only return aggregated or anonymized results.
- Do not generate SQL that joins to tables outside this space to access PII.
```

### Guardrail 3: Run-As Principal — Least Privilege

The Genie Space's `run_as` principal must have the minimum permissions needed:
- **Use a dedicated service principal** per space — never reuse a human account or admin principal.
- **Grant SELECT only on the specific tables/views added to the space** — not schema-level or catalog-level grants.
- **Verify the principal has no inherited grants** that would bypass column masks (e.g., `BYPASS` privileges or account admin).
- **Preferred permission level:** `VIEWER` — Genie only needs read access.

### Guardrail 4: Column Exclusion via Views

If a table must be in the space but some columns are sensitive, create a projection view that excludes PII columns:

```sql
CREATE OR REPLACE VIEW <catalog>.<schema>.<table>_genie_safe
AS SELECT
  -- non-PII columns only
  customer_id,  -- tokenized or hashed, not raw
  region,
  product_category,
  order_date,
  amount
  -- email, phone, ssn, address EXCLUDED
FROM <catalog>.<schema>.<table>;

-- Add the _genie_safe view to the Genie Space, NOT the base table
```

### Guardrail 5: SQL Example Review

Every SQL example in the Genie Space instructions must be audited:
- No example should `SELECT *` from a table with PII columns.
- No example should reference PII column names even in a WHERE clause (e.g., `WHERE email = '...'`).
- Examples should use aggregated patterns (`COUNT`, `SUM`, `GROUP BY`) that demonstrate non-individual-level querying.

### Guardrail 6: Warehouse Isolation

- Use a dedicated SQL warehouse for Genie Spaces handling regulated data — separate from ad-hoc analytics warehouses.
- Configure the warehouse with appropriate auto-stop and query timeout settings to limit exposure windows.
- Ensure warehouse access is restricted to the Genie service principal and authorized governance reviewers.

## 4. Execution Workflow

1. **Inventory tables for the proposed space:** Use `readTable` or `DESCRIBE TABLE EXTENDED` to identify all columns.
2. **Classify columns:** Run the pii-classifier skill on each candidate table.
3. **Apply the decision tree:** Determine which tables are safe, which need views/masks, and which are ineligible.
4. **Create safe views:** Generate `_genie_safe` projection views for tables with mixed PII/non-PII columns.
5. **Draft instructions:** Generate the privacy guardrail instructions block and privacy-safe SQL examples.
6. **Configure run-as:** Recommend the dedicated service principal and minimal grants.
7. **Produce a Genie Space Readiness Checklist** (see output template).

## 5. Reference Output Template

### 🧞 Genie Space Privacy Readiness: [Space Name]

**Table Assessment:**

| Table | PII Risk | Action | Object to Add |
|---|---|---|---|
| | | [Safe / Create view / Apply masks / Ineligible] | |

**Instructions Block:**
```
[Generated privacy guardrail instructions]
```

**Run-As Configuration:**
- Service Principal: [recommended name]
- Permission Level: VIEWER
- Grants needed: [list of specific table/view grants]

**SQL Examples (privacy-audited):**
```sql
-- Example 1: Aggregated query (safe)
SELECT region, COUNT(*) AS customer_count
FROM ... GROUP BY region;
```

**Compliance Notes:**
- [Which GDPR/CCPA/HIPAA requirements are addressed by these guardrails]

## 6. Guardrails for the Architect
- **Never add a table with unmasked direct identifiers to a Genie Space** — always route through a view or apply UC column masks first.
- **Always include the privacy guardrail instruction block** in the space's instructions.
- **Never use `SELECT *` in SQL examples** — explicitly list non-PII columns.
- **Review Genie Space access logs quarterly** — check `system.access.audit` for the space's warehouse to detect unexpected query patterns.
