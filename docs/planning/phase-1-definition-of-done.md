# Phase 1 Definition of Done

## Status

Active Completion Criteria

## Purpose

Defines when Phase 1 of the SEIA Toast / Fabric data platform is considered complete.

Prevents ambiguity around “done” and ensures the system is production-ready, not just functional.

---

## Phase 1 Objective

Build a fully functional, validated, and auditable reporting pipeline that replaces manual Toast reporting with automated Fabric + Power BI outputs.

---

## Core Requirements

### 1. Data Ingestion (Bronze Layer)

| Requirement | Complete? |
| --- | --- |
| All required Toast endpoints ingested | ☐ |
| Bronze tables exist and are populated | ☐ |
| Incremental logic implemented | ☐ |
| Replay windows implemented | ☐ |
| Payload audit fields populated | ☐ |

---

### 2. Data Transformation (Silver Layer)

| Requirement | Complete? |
| --- | --- |
| All stg_toast_* tables built | ☐ |
| Grain documented and validated | ☐ |
| Keys populated and deduplicated | ☐ |
| Rowcount audits recorded | ☐ |
| Hash-change audits recorded | ☐ |

---

### 3. Dimensions & Reference Data

| Requirement | Complete? |
| --- | --- |
| Core dimension tables built | ☐ |
| Menu and modifier dimensions built | ☐ |
| Bridge tables built | ☐ |
| Dimension merges working (upsert logic) | ☐ |

---

### 4. Gold Layer (Reporting)

| Requirement | Complete? |
| --- | --- |
| mart_reconciliation_daily built | ☐ |
| Mart grain validated | ☐ |
| Merge/upsert logic confirmed | ☐ |
| Rowcount audit recorded | ☐ |
| Pipeline run logged | ☐ |

---

### 5. Reconciliation

| Requirement | Complete? |
| --- | --- |
| Sales totals match Toast reports | ☐ |
| Payment totals match Toast reports | ☐ |
| Discounts validated | ☐ |
| Taxes validated | ☐ |
| Service charges validated | ☐ |
| Variances explained | ☐ |
| Tolerance thresholds defined | ☐ |

---

### 6. KPI Layer

| Requirement | Complete? |
| --- | --- |
| KPI Catalog defined | ☐ |
| KPI formulas aligned with warehouse logic | ☐ |
| KPIs validated against Toast reports | ☐ |

---

### 7. Power BI

| Requirement | Complete? |
| --- | --- |
| Semantic model built | ☐ |
| Finance dashboard created | ☐ |
| Core KPIs displayed | ☐ |
| Data refresh configured | ☐ |
| Dashboard validated with stakeholders | ☐ |

---

### 8. Control / Audit Framework

| Requirement | Complete? |
| --- | --- |
| ctl_pipeline_run logging working | ☐ |
| ctl_rowcount_audit populated | ☐ |
| ctl_hash_change_audit populated | ☐ |
| ctl_watermark implemented | ☐ |
| ctl_endpoint_window_log implemented | ☐ |

---

### 9. Documentation

| Requirement | Complete? |
| --- | --- |
| Architecture documented | ☐ |
| Data Dictionary complete | ☐ |
| Query / Notebook Inventory complete | ☐ |
| Reconciliation rules documented | ☐ |
| Incremental rules documented | ☐ |
| QA checklist defined | ☐ |

---

### 10. Operational Readiness

| Requirement | Complete? |
| --- | --- |
| Pipeline can be rerun without breaking | ☐ |
| Incremental loads stable | ☐ |
| Errors are logged and traceable | ☐ |
| Known issues documented | ☐ |
| Ownership defined (who maintains system) | ☐ |

---

## Not Included in Phase 1

| Item | Reason |
| --- | --- |
| PeopleVine integration | Separate phase (Phase 2) |
| Member spend attribution | Depends on PeopleVine bridge |
| Advanced forecasting | Phase 3 |
| AI insights layer | Phase 4 |

---

## Final Definition of Done

Phase 1 is complete when:

- Data flows end-to-end automatically
- Daily sales and payments reconcile to Toast
- Core KPIs are trusted
- Dashboards are usable by Finance
- All validation checks pass
- Control/audit tables are populated
- Documentation reflects actual implementation

---

## Sign-Off

| Role | Status |
| --- | --- |
| Finance | ☐ |
| Operations | ☐ |
| IT | ☐ |
| Project Owner | ☐ |

---

## Next Step

Once Phase 1 is complete:

- Begin PeopleVine integration
- Build member spend mart
- Expand dashboard suite
- Introduce alerting and automation