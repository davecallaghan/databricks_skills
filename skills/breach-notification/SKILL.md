---
name: breach-notification
description: Trigger this skill when a user asks to handle, document, or respond to a data breach or security incident involving PII, PHI, or regulated data. Also triggers when drafting GDPR 72-hour breach notifications, assessing breach scope, identifying affected data subjects, or packaging regulatory evidence. Covers GDPR Art. 33-34, HIPAA Breach Notification Rule, and CCPA breach response requirements.
---

# Skill: Breach Notification & Incident Response

## 1. Intent & Trigger
- Invoke this skill when the user reports or asks about a data breach, security incident, unauthorized access, or PII/PHI exposure event.
- Also invoke when drafting breach notification documents for supervisory authorities or affected data subjects.
- Also invoke when assessing breach scope, identifying affected data subjects, or packaging evidence for regulatory submissions.
- Do NOT invoke for general security monitoring — this skill is for active or post-incident breach response.

## 2. Regulatory Notification Requirements

| Regulation | Authority Notification | Subject Notification | Deadline | Key Thresholds |
|---|---|---|---|---|
| **GDPR Art. 33** | Supervisory Authority | — | **72 hours** from awareness | All personal data breaches must be notified unless "unlikely to result in risk to rights and freedoms" |
| **GDPR Art. 34** | — | Affected data subjects | **Without undue delay** | Required when "high risk to rights and freedoms" — clear, plain language |
| **HIPAA Breach Notification Rule** | HHS Secretary | Affected individuals | **60 days** from discovery | ≥500 individuals: notify HHS + media; <500: log in annual report |
| **CCPA / CA Civil Code 1798.82** | California Attorney General | Affected residents | **Without unreasonable delay** | ≥500 CA residents affected: notify AG; specific content requirements |
| **State breach laws (US)** | Varies by state | Affected residents | Varies (typically 30-60 days) | Check applicable state statutes |

## 3. Breach Response Lifecycle

```
Detection → Containment → Scope Assessment → Risk Classification → Authority Notification → Subject Notification → Evidence Packaging → Post-Incident Review
```

### Phase 1: Detection & Initial Triage

Identify the breach source from system tables and audit logs:

```sql
-- Detect anomalous access patterns (spike in queries on PII tables)
SELECT 
  DATE(event_time) AS event_date,
  action_type,
  COUNT(*) AS event_count,
  COUNT(DISTINCT user_name) AS distinct_users
FROM system.access.audit
WHERE event_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND (
    -- PII table access
    (action_type = 'SELECT' AND table_full_name LIKE '%customers%')
    OR (action_type = 'SELECT' AND table_full_name LIKE '%patients%')
    -- Permission changes on regulated tables
    OR (action_type IN ('GRANT', 'REVOKE') AND table_full_name LIKE '%pii%')
    -- Mask removal
    OR (action_type = 'ALTER' AND statement_text LIKE '%DROP MASK%')
  )
GROUP BY DATE(event_time), action_type
ORDER BY event_date DESC, event_count DESC;
```

```sql
-- Identify off-hours or unusual user access to PII tables
SELECT 
  event_time, user_name, action_type, 
  table_full_name, statement_text
FROM system.access.audit
WHERE event_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND action_type = 'SELECT'
  AND HOUR(event_time) NOT BETWEEN 7 AND 19  -- off-hours
  AND table_full_name IN (
    SELECT table_full_name FROM system.access.audit 
    WHERE statement_text LIKE '%email%' 
       OR statement_text LIKE '%ssn%' 
       OR statement_text LIKE '%medical%'
  )
ORDER BY event_time DESC;
```

### Phase 2: Containment Assessment

Determine if the breach is ongoing. Check for:
- Active sessions from unauthorized users.
- Recently modified permissions or removed masks.
- Data exports or volume spikes.

```sql
-- Check for recent permission grants on PII tables (potential privilege escalation)
SELECT 
  event_time, user_name, action_type, 
  table_full_name, statement_text
FROM system.access.audit
WHERE event_time >= DATE_SUB(CURRENT_DATE(), 7)
  AND action_type IN ('GRANT', 'REVOKE')
  AND table_full_name LIKE '%<affected_schema>%'
ORDER BY event_time DESC;
```

### Phase 3: Scope Assessment

Identify what data was accessed and which data subjects are affected.

```sql
-- Identify all PII/PHI columns in the affected tables
SELECT table_catalog, table_schema, table_name, column_name, tag_key
FROM <catalog>.governance.pii_inventory
WHERE table_name IN ('<affected_table_1>', '<affected_table_2>')
ORDER BY table_name, column_name;
```

```sql
-- Count affected data subjects (distinct individuals whose data was accessed)
SELECT 
  COUNT(DISTINCT <subject_id_column>) AS affected_subject_count,
  MIN(event_time) AS earliest_access,
  MAX(event_time) AS latest_access
FROM system.access.audit a
JOIN <catalog>.<schema>.<affected_table> t
  -- The breach accessed these records
WHERE a.event_time BETWEEN '<breach_start>' AND '<breach_end>'
  AND a.action_type = 'SELECT'
  AND a.table_full_name = '<catalog>.<schema>.<affected_table>'
  AND a.user_name IN ('<unauthorized_user_1>', '<unauthorized_user_2>');
```

### Phase 4: Risk Classification

Classify the breach severity per GDPR criteria:

| Risk Level | Criteria | Notification Required |
|---|---|---|
| **Low** | Data encrypted at rest, no exfiltration, no special category data | Document internally; no authority notification (GDPR Art. 33 exemption if "unlikely to result in risk") |
| **Medium** | Unencrypted PII accessed, limited scope (<500 subjects) | Authority notification within 72 hours; subject notification if risk assessment warrants |
| **High** | Unencrypted special category data (GDPR Art. 9), PHI, or >500 subjects | Authority notification within 72 hours + subject notification without undue delay |
| **Critical** | Large-scale special category data, financial data, or data exfiltrated to external systems | All notifications + potential public disclosure + law enforcement |

### Phase 5: Authority Notification Draft

Produce a GDPR Art. 33 notification document containing the required elements:

```
## Breach Notification to Supervisory Authority
### GDPR Article 33

**1. Nature of the breach:**
- Description: [What happened — unauthorized access, exfiltration, etc.]
- Date/time of breach: [detected on, occurred between]
- Type of breach: [confidentiality / integrity / availability]

**2. Data categories and approximate number of records/subjects affected:**
- Categories of personal data: [direct identifiers, financial, health, etc.]
- Approximate number of data subjects: [count]
- Approximate number of records: [count]

**3. Likely consequences:**
- [e.g., identity theft, financial fraud, reputational damage]

**4. Measures taken or proposed:**
- Containment: [what was done to stop the breach]
- Mitigation: [what was done to mitigate effects]
- Remediation: [long-term measures to prevent recurrence]

**5. DPO contact:**
- Name: [DPO name]
- Contact: [email, phone]

**6. Notification timing:**
- Breach detected: [date/time]
- This notification: [date/time] — [N hours after detection — within 72-hour requirement]
```

### Phase 6: Subject Notification Draft

Produce a GDPR Art. 34 notification (when high risk):

```
## Notification to Affected Data Subjects
### GDPR Article 34

Dear [Data Subject],

We are writing to inform you of a personal data breach that may affect your personal information.

**What happened:** [Plain-language description]
**When it happened:** [Date/time range]
**What data was involved:** [Categories of personal data — avoid including the actual data]
**What we are doing:** [Containment and mitigation steps]
**What you can do:** [Recommended protective actions — e.g., monitor accounts, change passwords]

**Our contact:** [DPO name and contact information]
**Your rights:** You may lodge a complaint with [relevant supervisory authority].
```

### Phase 7: Evidence Packaging

Compile the evidence trail for regulatory submission:

```sql
-- Complete audit trail of the breach event
SELECT 
  event_time, user_name, action_type, 
  table_full_name, statement_text, 
  source_ip_address, user_agent
FROM system.access.audit
WHERE event_time BETWEEN '<breach_start>' AND '<breach_end>'
  AND (
    table_full_name LIKE '%<affected_table>%'
    OR statement_text LIKE '%<affected_table>%'
  )
ORDER BY event_time;
```

Export evidence to a governance volume:
```python
evidence_df = spark.sql("""<above query>""")
evidence_df.coalesce(1).write.mode("overwrite") \
  .option("header", "true") \
  .csv(f"/Volumes/<catalog>/governance/breach_evidence/<incident_id>/")
```

### Phase 8: Breach Register Entry

GDPR Art. 33(5) requires documenting all breaches regardless of notification:

```sql
INSERT INTO <catalog>.governance.breach_register
VALUES (
  '<incident_id>',
  '<breach_detected_at>',
  '<breach_occurred_between>',
  '<breach_type>',  -- confidentiality / integrity / availability
  '<data_categories>',
  <affected_subject_count>,
  <affected_record_count>,
  '<risk_level>',
  '<authority_notified>',  -- Y / N / N/A
  '<authority_notified_at>',
  '<subjects_notified>',  -- Y / N / N/A
  '<subjects_notified_at>',
  '<containment_actions>',
  '<remediation_actions>',
  current_user(),
  current_timestamp()
);
```

## 4. Reference Output Template

### 🚨 Breach Incident Report: [Incident ID]

**Incident Overview:**

| Field | Value |
|---|---|
| Incident ID | [ID] |
| Detected At | [timestamp] |
| Occurred Between | [start – end] |
| Breach Type | [Confidentiality / Integrity / Availability] |
| Risk Level | [Low / Medium / High / Critical] |
| Authority Notification Deadline | [72 hours from detection — expires at: timestamp] |
| Subject Notification Required? | [Yes / No — rationale] |

**Scope:**

| Table | PII/PHI Columns Accessed | Affected Subjects | Access Method |
|---|---|---|---|
| | | | [query / export / permission escalation] |

**Timeline:**

| Time | Event | Actor | Action |
|---|---|---|---|
| | | | |

**Authority Notification:** [Draft attached — GDPR Art. 33]
**Subject Notification:** [Draft attached — GDPR Art. 34, if applicable]

**Evidence Package:**
- Export path: `/Volumes/<catalog>/governance/breach_evidence/<incident_id>/`
- Contents: [audit log export, PII inventory snapshot, permission change log]

**Breach Register Entry:** [SQL generated — to be inserted into governance.breach_register]

**Deadline Status:** [✅ Within 72 hours / ⚠️ N hours remaining / 🚫 OVERDUE]

## 5. Guardrails
- **Start the 72-hour clock from the moment of awareness** — not from the start of the breach. Document exactly when the breach was detected.
- **Never assume the breach is contained** — always verify via audit logs that unauthorized access has stopped before declaring containment.
- **Never omit a breach from the register** — GDPR Art. 33(5) requires documentation even when authority notification is not required.
- **Do not include actual PII values in notification drafts** — reference data categories only (e.g., "email addresses and phone numbers" not the actual addresses/numbers).
- **Notify the DPO immediately** — this skill assists in drafting, but the DPO (or equivalent) must review and approve all notifications before submission.
- **Preserve evidence before remediation** — do not alter permissions or revoke access until the full audit trail has been exported. Changing permissions may destroy evidence of the breach.
- **HIPAA has a 60-day clock but GDPR has 72 hours** — when both apply, the shorter deadline governs.
- **CCPA notification to the CA AG is required for breaches affecting ≥500 residents** — include the specific content elements required by CA Civil Code 1798.82(d).
