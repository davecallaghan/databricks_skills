---
name: genie-readiness-scorer
description: Trigger this skill when a user asks to assess, score, or evaluate Genie readiness for a table, schema, or catalog. Also triggers when the user wants a readiness audit, certification check, or multi-dimensional quality score for Genie Spaces or the Ontology layer. Covers the 8-dimension scoring framework with weighted composite and per-dimension minimums.
---

# Skill: Genie Readiness Scorer (8-Dimension Framework)

## 1. Intent & Trigger
- Invoke this skill when the user asks to assess Genie readiness for any UC table, schema, or catalog.
- Also invoke when the user asks for a certification check, readiness score, or quality audit with respect to Genie Spaces.
- Also invoke when the user asks "is this table ready for Genie?" or "what's the readiness score?".
- Do NOT invoke for general data quality checks unrelated to Genie/Ontology readiness.

## 2. The 8-Dimension Scoring Framework

Each dimension is scored 0-5. The composite score is a weighted average. A table is **certification-eligible** only if it meets ALL per-dimension minimums.

| # | Dimension | Weight | What It Measures | Score 0 | Score 5 |
|---|---|---|---|---|---|
| 1 | **Foundation** | 15% | UC registration, correct table types, partitioning, clustering | Not registered in UC | Fully registered, optimized partition/clustering strategy |
| 2 | **Documentation** | 15% | Column descriptions, table comments, synonyms, display names | No descriptions anywhere | Every column has comment, display_name, and synonyms |
| 3 | **Data Quality** | 15% | Null checks, referential integrity, uniqueness, value distributions | No DQ checks, high null rates | Enforced DQ constraints, clean distributions, CDF enabled |
| 4 | **Governance** | 15% | PII tags, access controls, column masks, row filters, lineage | No governance controls | Full tag coverage, masks on all PII, row filters, lineage complete |
| 5 | **Semantic Clarity** | 10% | Business-meaningful names, metric views, relationship docs | Cryptic column names, no semantic context | Metric views published, relationships documented, synonyms mapped |
| 6 | **Lineage** | 10% | Upstream/downstream lineage visibility in UC | No lineage captured | Full lineage from source to consumption via system tables |
| 7 | **Freshness** | 10% | Data freshness vs. expected update cadence | Stale beyond SLA, no update mechanism | Streaming or scheduled refresh, meets SLA consistently |
| 8 | **Volume** | 10% | Row count and data volume stability | Empty or erratically fluctuating | Stable volume, sufficient for meaningful Genie queries |

**Composite Score** = Σ(dimension_score × weight) / Σ(weights)

**Certification Threshold:** Composite ≥ 3.5 AND every dimension ≥ 2.0

## 3. Scoring Rubric (Per Dimension)

### Dimension 1: Foundation
| Score | Criteria |
|---|---|
| 0 | Not registered in Unity Catalog |
| 1 | Registered but external/non-Delta, no metadata |
| 2 | Delta table, registered, no partitioning/clustering |
| 3 | Delta, partitioned OR liquid clustered |
| 4 | Delta, partitioned AND clustered, CDF enabled |
| 5 | Optimized Delta with partitioning, clustering, CDF, and Z-ordering |

### Dimension 2: Documentation
| Score | Criteria |
|---|---|
| 0 | No table comment, no column comments |
| 1 | Table comment exists, <25% columns commented |
| 2 | 25-50% columns commented |
| 3 | 50-75% columns commented, table comment present |
| 4 | >75% columns commented, some have display_name or synonyms |
| 5 | 100% columns have comment + display_name + synonyms |

### Dimension 3: Data Quality
| Score | Criteria |
|---|---|
| 0 | No DQ checks, high null rates on business keys |
| 1 | Some manual checks, no enforcement |
| 2 | SDP expectations or constraints defined, not all enforced |
| 3 | Enforced constraints on critical columns (NOT NULL, uniqueness) |
| 4 | Comprehensive DQ (null, uniqueness, referential integrity, range checks) |
| 5 | Comprehensive DQ + automated monitoring/alerting + CDF for audit |

### Dimension 4: Governance (Privacy-Critical)
| Score | Criteria |
|---|---|
| 0 | No tags, no access controls, no masks |
| 1 | Basic GRANT/DENY, no PII tags |
| 2 | PII tags on some columns, basic access controls |
| 3 | PII tags on all identified columns, column masks on direct identifiers |
| 4 | Full tag coverage, masks + row filters, lineage visible in UC |
| 5 | Full tag coverage, masks + row filters, lineage, audit logging verified, retention policy set |

### Dimension 5: Semantic Clarity
| Score | Criteria |
|---|---|
| 0 | Cryptic names (e.g., `col1`, `tbl_a`), no business context |
| 1 | Some meaningful names, no metric views |
| 2 | Most names meaningful, table comment explains purpose |
| 3 | All names meaningful, some columns have synonyms/display_name |
| 4 | Metric views published for key measures, relationship documentation exists |
| 5 | Full metric view coverage, all columns have semantic metadata, relationships documented in ontology |

### Dimension 6: Lineage
| Score | Criteria |
|---|---|
| 0 | No lineage visible |
| 1 | Manual lineage docs exist, not in UC |
| 2 | Some lineage in `system.access.table_lineage` |
| 3 | Upstream lineage complete in UC |
| 4 | Upstream + downstream lineage visible |
| 5 | Full lineage graph, including notebook/job provenance |

### Dimension 7: Freshness
| Score | Criteria |
|---|---|
| 0 | No data, or stale > 30 days beyond expected cadence |
| 1 | Sporadic updates, no defined SLA |
| 2 | Batch updates, meets SLA sometimes |
| 3 | Scheduled refresh, meets SLA most of the time |
| 4 | Streaming or frequent batch, meets SLA consistently |
| 5 | Real-time/streaming, meets SLA with monitoring and alerting |

### Dimension 8: Volume
| Score | Criteria |
|---|---|
| 0 | Empty table |
| 1 | <100 rows (insufficient for meaningful Genie queries) |
| 2 | 100-1K rows, stable |
| 3 | 1K-100K rows, stable |
| 4 | 100K-10M rows, stable |
| 5 | >10M rows, stable volume, supports diverse query patterns |

## 4. Execution Workflow

1. **Scope:** Determine whether to score a single table, a schema, or an entire catalog.
2. **Collect signals:** For each table, query system tables and metadata:
   - `DESCRIBE TABLE EXTENDED` — types, comments, properties
   - `SHOW TBLPROPERTIES` — CDF, retention, governance metadata
   - `system.information_schema.columns` — column comments, data types
   - `system.access.table_lineage` — lineage presence
   - PII tag coverage — check for `pii.*` / `phi.*` tags
   - Row count and freshness — `SELECT COUNT(*)`, `MAX(updated_at)`
3. **Score each dimension:** Apply the rubric above to the collected signals.
4. **Calculate composite:** Weighted average across 8 dimensions.
5. **Check certification eligibility:** Composite ≥ 3.5 AND every dimension ≥ 2.0.
6. **Generate remediation plan:** For any dimension scoring below 3, list specific actions to raise the score.

## 5. Reference Output Template

### 📊 Genie Readiness Assessment: [catalog].[schema].[table]

**Composite Score:** [X.XX / 5.00] — [Certification Eligible / Not Eligible]

**Dimension Scores:**

| # | Dimension | Weight | Score | Min Met? | Key Finding |
|---|---|---|---|---|---|
| 1 | Foundation | 15% | [0-5] | ✅/❌ | [e.g., 'Delta table, no clustering'] |
| 2 | Documentation | 15% | [0-5] | ✅/❌ | [e.g., '40% columns commented'] |
| 3 | Data Quality | 15% | [0-5] | ✅/❌ | [e.g., 'No enforced constraints'] |
| 4 | Governance | 15% | [0-5] | ✅/❌ | [e.g., 'No PII tags, no masks'] |
| 5 | Semantic Clarity | 10% | [0-5] | ✅/❌ | [e.g., 'No metric views, no synonyms'] |
| 6 | Lineage | 10% | [0-5] | ✅/❌ | [e.g., 'Upstream lineage only'] |
| 7 | Freshness | 10% | [0-5] | ✅/❌ | [e.g., 'Last updated 12 days ago'] |
| 8 | Volume | 10% | [0-5] | ✅/❌ | [e.g., '15K rows, stable'] |

**Certification Status:** [✅ Eligible / ❌ Not Eligible — N dimension(s) below minimum]

**Weakest Dimension:** [Dimension name] — [score] — [primary reason]

**Remediation Plan (prioritized by impact):**
1. [Action to raise Dimension X from score Y to score Z — estimated effort]
2. ...
3. ...

**Recommended Next Steps:**
- [Specific, actionable steps — e.g., 'Apply PII tags via pii-classifier skill', 'Create metric views via uc-metric-view-author skill']

## 6. Batch Assessment Guidance

When scoring an entire schema or catalog:
1. Iterate over all tables in `system.information_schema.tables` filtered by `table_schema`.
2. Score each table independently.
3. Produce a summary table with all tables sorted by composite score DESC.
4. Identify the top-N certification candidates (composite ≥ 3.5, all dimensions ≥ 2.0).
5. Identify common weaknesses across the schema (e.g., 'Governance is universally the weakest dimension — no PII tagging exists').
6. Recommend schema-level remediation actions that benefit all tables.

## 7. Guardrails
- **Governance dimension (Dim 4) is non-negotiable for tables containing PII/PHI** — if Dim 4 < 2, the table is ineligible for Genie regardless of composite score.
- **Always check for PII/PHI before scoring** — use the pii-classifier skill or verify existing tags. A table that appears governance-ready may have untagged PII.
- **Never score a table you cannot read** — if `DESCRIBE TABLE EXTENDED` fails due to permissions, flag it as 'Unable to Assess' rather than scoring it 0.
- **Freshness check must use the correct time column** — verify which column represents the data's effective date before scoring freshness. Using `_ingested_at` for a slowly-changing dimension will produce misleading scores.
