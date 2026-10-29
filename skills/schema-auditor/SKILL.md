---
name: schema-auditor
description: Trigger this skill when a user asks to inspect, check data quality, or audit a new table schema or view in Unity Catalog.
---

# Skill: Unity Catalog Schema Compliance Auditor

## 1. Intent & Trigger
- You must invoke this skill whenever the user explicitly requests an audit, schema summary, or initial data quality check on a targeted Unity Catalog table or view.

## 2. Objective Execution Steps
When this skill triggers, execute the following precise operational workflow:
1. **Schema Extraction:** Run `DESCRIBE TABLE EXTENDED <table_name>` to capture column types and metadata properties.
2. **Profile Profiling:** Execute a non-blocking `SELECT` statement to sample the top 5 rows.
3. **Data Quality Profile:** Calculate basic counts including total rows, total distinct keys, and null/empty string counts for critical structural columns.
4. **Compliance Check:** Verify if columns adhere to lowercase `snake_case` patterns and contain explicit business description comments.

## 3. Reference Output Template
Format the final analysis cleanly in Markdown without preamble text:

### 📋 Table Compliance Audit: [catalog].[schema].[table]

**Basic Profile:**
- **Total Rows:** [Insert Row Count]
- **Distinct Business Keys:** [Insert Count]

**Quality Summary:**

| Column Name | Data Type | Null Count | Snake Case? | Comment Populated? |
|---|---|---|---|---|
| | | | | |

**Architectural Recommendations:**
- [Provide a bulleted list highlighting missing descriptions or non-standard formatting constraints found during step 4.]
