---
name: cross-border-transfer-validator
description: Trigger this skill when a user asks to validate, assess, or document cross-border data transfers of PII, PHI, or regulated data. Also triggers when checking GDPR Chapter V compliance, SCC adequacy, transfer impact assessments (TIA), data residency requirements, or third-country risk. Covers Schrems II implications and supplementary measures.
---

# Skill: Cross-Border Transfer Validator (GDPR Chapter V)

## 1. Intent & Trigger
- Invoke this skill when the user asks whether a data transfer crosses jurisdictional borders and whether it complies with GDPR Chapter V.
- Also invoke when performing a Transfer Impact Assessment (TIA), verifying SCC coverage, or checking adequacy decisions.
- Also invoke when designing pipelines that move PII/PHI across regions and need to validate the transfer mechanism.
- Also invoke when a Schrems II supplementary measures assessment is needed.
- Do NOT invoke for domestic data movement — this skill is specifically for cross-border transfers of regulated data.

## 2. GDPR Chapter V Transfer Mechanisms

| Mechanism | Article | When Applicable | Key Requirement |
|---|---|---|---|
| **Adequacy Decision** | Art. 45 | Destination country has been deemed adequate by the EU Commission | No additional safeguards needed (but Schrems II may still require assessment) |
| **Standard Contractual Clauses (SCCs)** | Art. 46(2)(c) | No adequacy decision for the destination | Must assess if SCCs alone provide effective protection (Schrems II) |
| **Binding Corporate Rules (BCRs)** | Art. 47 | Intra-organization transfers | Approved by competent supervisory authority |
| **Derogations** | Art. 49 | Occasional, non-repetitive transfers | Explicit consent, contract performance, public interest, etc. |
| **Supplementary Measures** | Schrems II ruling | When SCCs/BCRs alone are insufficient due to destination country law | Encryption, pseudonymization, split processing, contractual measures |

### Current EU Adequacy Decisions (as of 2024)
- Andorra, Argentina, Canada (commercial organizations), Faroe Islands, Guernsey, Israel, Isle of Man, Japan, Jersey, New Zealand, Republic of Korea, Switzerland, United Kingdom, Uruguay, United States (Data Privacy Framework — certified organizations only)
- **Not adequate:** China, India, Russia, Brazil, most of Africa/Asia/Middle East
- **Partial:** Canada (commercial only), US (DPF-certified organizations only — non-certified US organizations require SCCs + TIA)

## 3. Transfer Validation Workflow

### Step 1: Identify the Transfer

Determine if the data movement constitutes a cross-border transfer:
- **Storage location:** Where is the Delta table physically stored? (check cloud region)
- **Compute location:** Where does the processing occur? (check Databricks workspace region)
- **Access location:** Where do users/services access the data from? (check via audit logs)
- **Subprocessor location:** Do any Databricks subprocessors process the data in a third country?

```sql
-- Check table storage location (if available via table properties)
SHOW TBLPROPERTIES <catalog>.<schema>.<table>;
-- Look for 'delta.storage.location' or cloud provider path

-- Check compute region
-- This is available via the Databricks workspace configuration, not via SQL
-- Flag if the workspace region differs from the data subject's jurisdiction
```

### Step 2: Classify the Data

```sql
-- Identify what PII/PHI is in the transferred data
SELECT column_name, tag_key, 
  CASE 
    WHEN tag_key LIKE 'pii.direct%' THEN 'HIGH'
    WHEN tag_key LIKE 'pii.special%' THEN 'CRITICAL'
    WHEN tag_key LIKE 'phi.%' THEN 'CRITICAL'
    WHEN tag_key LIKE 'pii.financial%' THEN 'HIGH'
    WHEN tag_key LIKE 'pii.quasi%' THEN 'MEDIUM'
    ELSE 'LOW'
  END AS transfer_risk
FROM <catalog>.governance.pii_inventory
WHERE table_name = '<table>'
ORDER BY transfer_risk DESC, column_name;
```

### Step 3: Determine the Transfer Mechanism

| Origin → Destination | Adequacy? | Required Mechanism |
|---|---|---|
| EU → EU/EEA | Yes | None — intra-EU transfer, no Chapter V issue |
| EU → Adequate country (e.g., UK, Switzerland, Japan) | Yes | None — but document the adequacy basis |
| EU → US (DPF-certified org) | Conditional | DPF certification must be verified for the recipient |
| EU → US (non-DPF) | No | SCCs + Transfer Impact Assessment |
| EU → Non-adequate country (e.g., India, China) | No | SCCs + TIA + Supplementary Measures |
| CA (California) → Any US state | N/A (CCPA) | CCPA applies regardless of state; if EU data subjects involved, GDPR Chapter V also applies |

### Step 4: Transfer Impact Assessment (TIA)

Per Schrems II (C-311/18), when using SCCs to a non-adequate country, assess:

**4a. Destination Country Law Assessment:**
- Does the destination country have surveillance laws that could compel disclosure of personal data to government authorities? (e.g., US CLOUD Act, China PIPL, India IT Act)
- Are there legal remedies available to data subjects in the destination country?
- Is there an independent supervisory authority in the destination country?

**4b. Practical Ability to Honor SCCs:**
- Can the importer technically comply with SCC obligations?
- Are there contractual clauses that override or conflict with SCCs?
- Can the importer refuse government access requests, or are they legally compelled to comply?

**4c. Supplementary Measures Assessment:**

| Measure | Type | Effectiveness | Use Case |
|---|---|---|---|
| **End-to-end encryption** | Technical | High — data unreadable in transit and at rest | Protects confidentiality even if intercepted |
| **Pseudonymization** | Technical | High — identifiers removed, only tokens transferred | Reduces risk if data intercepted |
| **Split processing** | Technical/Organizational | Medium — data split across jurisdictions | No single recipient has complete dataset |
| **Data minimization** | Technical | Medium — only necessary data transferred | Reduces scope of potential breach |
| **Contractual transparency** | Contractual | Low alone, high with technical | Importer must notify of government access requests |
| **Access controls + audit** | Technical/Organizational | Medium — limits who can access data | Reduces insider risk |

```sql
-- Check if supplementary measures are in place for the transferred table
SELECT 
  table_name, 
  CASE WHEN column_mask IS NOT NULL THEN 'YES' ELSE 'NO' END AS column_masking,
  CASE WHEN row_filter IS NOT NULL THEN 'YES' ELSE 'NO' END AS row_filtering,
  CASE WHEN TBLPROPERTIES LIKE '%delta.enableDeletionVectors%' THEN 'YES' ELSE 'UNKNOWN' END AS encryption_status
FROM (
  SELECT table_name, 
    MAX(CASE WHEN property_key = 'column_mask' THEN property_value END) AS column_mask,
    MAX(CASE WHEN property_key = 'row_filter' THEN property_value END) AS row_filter,
    MAX(CASE WHEN property_key = 'delta.enableDeletionVectors' THEN property_value END) AS TBLPROPERTIES
  FROM system.information_schema.tables
  WHERE table_name = '<table>'
  GROUP BY table_name
);
```

### Step 5: Risk Score

Score the transfer on a 0-5 scale:

| Score | Risk Level | Criteria |
|---|---|---|
| 0 | Minimal | Adequacy decision covers the destination; no special category data |
| 1 | Low | Adequacy decision + special category data (still requires documentation) |
| 2 | Moderate | SCCs to a country with strong rule of law; supplementary measures in place |
| 3 | Elevated | SCCs to a country with surveillance laws; some supplementary measures |
| 4 | High | SCCs to a country with broad surveillance powers; insufficient supplementary measures |
| 5 | Critical | No valid transfer mechanism; or TIA shows SCCs cannot be honored; transfer should be suspended |

### Step 6: Transfer Register Entry

Document the transfer in a governance register:

```sql
INSERT INTO <catalog>.governance.transfer_register
VALUES (
  '<transfer_id>',
  '<table_name>',
  '<data_categories>',         -- e.g., 'pii.direct_identifier, pii.financial_data'
  '<origin_jurisdiction>',     -- e.g., 'EU-Ireland'
  '<destination_jurisdiction>', -- e.g., 'US-California'
  '<transfer_mechanism>',      -- e.g., 'SCC', 'Adequacy-UK', 'DPF-US'
  '<tia_status>',              -- e.g., 'Completed', 'Not Required (adequacy)', 'Required'
  '<supplementary_measures>',  -- e.g., 'encryption, pseudonymization'
  <risk_score>,                -- 0-5
  '<assessment_date>',
  '<assessed_by>',
  current_timestamp()
);
```

## 4. Reference Output Template

### 🌍 Cross-Border Transfer Validation: [table → destination]

**Transfer Overview:**

| Field | Value |
|---|---|
| Table | [catalog.schema.table] |
| Data Categories | [PII categories / PHI / financial] |
| Origin Jurisdiction | [e.g., EU-Ireland] |
| Destination Jurisdiction | [e.g., US-California] |
| Transfer Mechanism | [Adequacy / SCC / DPF / BCR / Derogation] |
| Adequacy Status | [Adequate / Not adequate / Conditional (DPF)] |
| Risk Score | [0-5] — [Risk Level] |

**TIA Assessment:**

| Assessment Area | Finding | Risk |
|---|---|---|
| Destination country surveillance laws | [e.g., 'CLOUD Act applies — government can compel disclosure'] | High |
| Legal remedies for data subjects | [e.g., 'Limited — no equivalent to GDPR remedies'] | Medium |
| SCC honorability | [e.g., 'Importer can technically comply but may be legally compelled to override'] | High |

**Supplementary Measures:**

| Measure | In Place? | Effectiveness |
|---|---|---|
| End-to-end encryption | [YES/NO] | [High/Medium/None] |
| Pseudonymization | [YES/NO] | [High/Medium/None] |
| Split processing | [YES/NO] | [High/Medium/None] |
| Data minimization | [YES/NO] | [High/Medium/None] |
| Contractual transparency | [YES/NO] | [High/Medium/None] |
| Access controls + audit | [YES/NO] | [High/Medium/None] |

**Recommendation:**
- [Transfer approved / Transfer requires supplementary measures / Transfer should be suspended]
- [Specific actions to reduce risk — e.g., 'Enable end-to-end encryption before transfer', 'Pseudonymize direct identifiers at source']

**Transfer Register Entry:** [SQL generated]

**Compliance Notes:**
- [Which GDPR articles apply — Art. 44-49, Schrems II]
- [Which CCPA requirements apply if California residents' data is involved]
- [Next review date — TIAs should be reviewed annually or when destination country law changes]

## 5. Guardrails
- **Never approve a transfer to a non-adequate country without a completed TIA** — the Schrems II ruling makes this a hard requirement, not a recommendation.
- **The US Data Privacy Framework is conditional** — only organizations certified under the DPF are covered by the adequacy decision. Always verify the recipient's DPF certification status; non-certified US organizations require SCCs + TIA.
- **Adequacy decisions can be revoked** — the EU-US Privacy Shield was invalidated by Schrems II. Monitor for adequacy decision changes and re-assess if revoked.
- **Pseudonymization is not anonymization** — pseudonymized data is still personal data under GDPR. Transfers of pseudonymized data still require a valid Chapter V mechanism.
- **Compute location matters, not just storage** — if EU data is stored in the EU but processed on US-based compute, that is still a cross-border transfer requiring Chapter V compliance.
- **Subprocessor transfers must be disclosed** — Databricks subprocessors may process data in third countries. Verify the Databricks subprocessor list and ensure BAAs/DPAs cover all processing locations.
- **Review TIAs annually** — destination country laws change. A transfer that was compliant last year may not be compliant today.
- **Never auto-suspend or block a transfer** — present the assessment and recommendation, and wait for user/DPO sign-off before any action.
