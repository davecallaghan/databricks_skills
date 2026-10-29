---
name: pii-classifier
description: Trigger this skill when a user asks to scan, classify, or identify PII or PHI columns in a Unity Catalog table or schema. Also triggers when auditing data for privacy risk, recommending UC governed tags, or assessing GDPR/CCPA/HIPAA exposure across a dataset.
---

# Skill: PII / PHI Column Classifier

## 1. Intent & Trigger
- Invoke this skill when the user asks to scan a table or schema for personal data, PII, PHI, sensitive data, or regulated data.
- Also invoke when the user asks for a privacy risk assessment, data inventory, or PII tagging recommendations on UC assets.

## 2. Classification Taxonomy

Classify every column into one of these categories:

| Category | Tag Key | Examples | Regulatory Trigger |
|---|---|---|---|
| **Direct Identifier** | `pii.direct_identifier` | email, ssn, passport_number, phone, full_name, national_id | GDPR Art. 4(1), CCPA "identifiers" |
| **Quasi-Identifier** | `pii.quasi_identifier` | zip_code, date_of_birth, gender, ip_address, device_id | GDPR re-identification risk, CCPA "household" |
| **Sensitive/Special Category** | `pii.special_category` | race, religion, health_data, biometric, sexual_orientation, genetic | GDPR Art. 9, CCPA "sensitive PI" |
| **Financial Data** | `pii.financial_data` | credit_card_number, bank_account, salary, transaction_amount | CCPA "financial", PCI-DSS |
| **Health Data (PHI)** | `phi.health_data` | diagnosis_code, medical_record_number, prescription, lab_result | HIPAA, GDPR Art. 9 |
| **Non-Personal** | *(no tag)* | aggregate counts, system timestamps, weather data | None |

## 3. Execution Workflow

### Step 1: Schema Extraction
Run `DESCRIBE TABLE EXTENDED <catalog>.<schema>.<table>` to capture all column names, types, and comments.

### Step 2: Heuristic Column Name Matching
Match column names against known PII/PHI patterns:
- **Direct identifiers:** `%email%`, `%ssn%`, `%passport%`, `%phone%`, `%mobile%`, `%national_id%`, `%full_name%`, `%first_name%`, `%last_name%`
- **Quasi-identifiers:** `%zip%`, `%postal%`, `%date_of_birth%`, `%dob%`, `%gender%`, `%ip_address%`, `%device_id%`
- **Sensitive:** `%race%`, `%ethnicity%`, `%religion%`, `%health%`, `%diagnosis%`, `%biometric%`, `%genetic%`, `%sexual%`
- **Financial:** `%credit_card%`, `%card_number%`, `%bank_account%`, `%salary%`, `%compensation%`
- **Health (PHI):** `%medical_record%`, `%mrn%`, `%prescription%`, `%lab_result%`, `%diagnosis_code%`, `%icd%`

### Step 3: Data Sample Inspection (Optional)
If column names are ambiguous, sample up to 5 non-null values to confirm classification:
```sql
SELECT <column_name> FROM <catalog>.<schema>.<table> 
WHERE <column_name> IS NOT NULL LIMIT 5
```
Apply format-based detection: email regex, SSN pattern (XXX-XX-XXXX), credit card (13-19 digits), ICD-10 codes, etc.

### Step 4: Tag Recommendation
For each classified column, generate the UC `ALTER TABLE` statement to apply the governed tag:
```sql
ALTER TABLE <catalog>.<schema>.<table> 
SET TAGS ('pii.direct_identifier' = 'true');
```
For column-level tags:
```sql
ALTER TABLE <catalog>.<schema>.<table> 
ALTER COLUMN <column_name> SET TAGS ('pii.direct_identifier' = 'true');
```

### Step 5: Risk Scoring
Score each table on a 0-5 privacy risk scale:
- **0:** No PII detected
- **1:** Quasi-identifiers only (low re-identification risk)
- **2:** Single direct identifier (moderate risk)
- **3:** Multiple direct identifiers or financial data (high risk)
- **4:** Special category or PHI present (very high risk — GDPR Art. 9 / HIPAA)
- **5:** Direct identifiers + special category + PHI (critical risk)

## 4. Reference Output Template

### 🔒 Privacy Classification Report: [catalog].[schema].[table]

**Overall Risk Score:** [0-5] — [Risk Level]

**Classification Summary:**

| Column Name | Data Type | Category | Tag Key | Confidence | Sample Value (masked) |
|---|---|---|---|---|---|
| | | | | | |

**Recommended Tagging SQL:**
```sql
-- Column-level governed tag application
ALTER TABLE ...;
```

**Recommended Controls:**
- [List columns that need column masks, row filters, or access restrictions based on category and risk score]

**Regulatory Implications:**
- [GDPR / CCPA / HIPAA — which articles/rights are triggered by the detected data categories]

## 5. Edge Cases
- **Aggregated columns** (e.g., `total_customers`): Classify as non-personal — the aggregate does not identify individuals.
- **Tokenized columns** (e.g., `customer_token`): Classify as quasi-identifier if the tokenization map exists; non-personal if truly irreversible and the mapping is destroyed.
- **Free-text columns** (e.g., `notes`, `comments`): Flag for manual review — may contain embedded PII not detectable by column name alone.
- **Nested struct fields:** Flatten and classify each leaf field individually.
