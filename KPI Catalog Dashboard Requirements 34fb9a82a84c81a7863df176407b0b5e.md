# Control Tables

## Purpose

Defines the control, audit, lineage, and data-quality tables used in the SEIA Toast / Fabric pipeline.

These tables support pipeline monitoring, incremental loading, reconciliation, debugging, and member attribution workflows.

## Source Notebook

Primary implementation notebook: `NB_CREATE_CONTROL_TABLES`.

This notebook creates or confirms the required `ctl_*` Delta tables, validates their existence, and logs the setup run into `ctl_pipeline_run`.

## Control Table Inventory

| Table | Purpose | Primary Use |
| --- | --- | --- |
| ctl_pipeline_run | Tracks notebook and pipeline executions | Run history, status, errors, timing |
| ctl_watermark | Stores incremental load state | Incremental refresh and replay control |
| ctl_endpoint_window_log | Logs API request windows and responses | API debugging and load completeness |
| ctl_rowcount_audit | Tracks row counts, distinct keys, duplicate keys, and null keys | Data quality validation |
| ctl_hash_change_audit | Tracks row-level insert/update changes by hash comparison | Change audit and lineage |
| ctl_recon_result | Stores metric-level reconciliation results | Finance validation and variance tracking |
| ctl_unresolved_member_match | Tracks payments/member records that could not be matched | PeopleVine bridge QA |
| ctl_manual_override_member_match | Stores manual corrections for member attribution | Controlled member-match overrides |
| ctl_data_quality_issue | Central table for data-quality issues | QA monitoring and issue resolution |

## ctl_pipeline_run

Tracks each notebook or pipeline execution.

| Field | Meaning |
| --- | --- |
| run_id | Unique run identifier |
| pipeline_name | Notebook or pipeline name |
| pipeline_stage | Specific stage of the process |
| run_type | Manual, scheduled, repair, or other run type |
| trigger_type | Manual or automated trigger source |
| source_system | Source system such as Toast or Fabric |
| status | SUCCEEDED / FAILED / other status |
| started_at_utc | Run start timestamp |
| ended_at_utc | Run end timestamp |
| duration_seconds | Runtime duration |
| rows_read | Rows read, if applicable |
| rows_written | Rows written, if applicable |
| error_message | Failure detail |

Used by `NB_CREATE_CONTROL_TABLES`, `NB_BUILD_CORE_DIMENSIONS`, and `NB_BUILD_RECON_MART`.

## ctl_watermark

Tracks incremental state by source system, entity, restaurant, and watermark type.

| Field | Meaning |
| --- | --- |
| source_system | Source system name |
| entity_name | Entity being refreshed |
| restaurant_guid | Restaurant/location identifier |
| watermark_type | Type of watermark, such as modified window or business date |
| last_successful_start_utc | Last successful request start |
| last_successful_end_utc | Last successful request end |
| last_successful_business_date | Last completed business date |
| last_successful_payload_hash | Latest payload hash when applicable |
| last_run_id | Last successful pipeline run |
| status | Current status |
| notes | Implementation notes |

## ctl_endpoint_window_log

Tracks each API request window and response.

| Field | Meaning |
| --- | --- |
| window_log_id | Unique window log row |
| run_id | Related pipeline run |
| source_endpoint | API endpoint called |
| entity_name | Entity being loaded |
| request_window_start_utc | Request window start |
| request_window_end_utc | Request window end |
| request_business_date | Business date requested, if applicable |
| response_status_code | API response status |
| response_record_count | Number of records returned |
| response_payload_hash | Hash of response payload |
| status | Success/failure status |
| error_message | Failure detail |

## ctl_rowcount_audit

Tracks row counts and key health.

| Field | Meaning |
| --- | --- |
| rowcount_audit_id | Unique audit row |
| run_id | Related pipeline run |
| layer_name | Bronze, Silver, Gold, etc. |
| table_name | Table audited |
| source_system | Source system |
| source_entity | Source entity/table |
| row_count | Total rows |
| distinct_key_count | Distinct key count |
| duplicate_key_count | Duplicate count |
| null_key_count | Null key count |
| audit_status | PASS / WARNING / FAIL |
| checked_at_utc | Audit timestamp |
| notes | Audit notes |

Used by `NB_BUILD_CORE_DIMENSIONS` and `NB_BUILD_RECON_MART`.

## ctl_hash_change_audit

Tracks row-level insert and update events using row hashes.

| Field | Meaning |
| --- | --- |
| hash_change_audit_id | Unique audit row |
| run_id | Related pipeline run |
| table_name | Table audited |
| natural_key | Business/natural key |
| previous_hash | Existing row hash |
| current_hash | Incoming row hash |
| change_type | INSERT / UPDATE |
| changed_at_utc | Change timestamp |
| notes | Audit notes |

Used by `NB_BUILD_CORE_DIMENSIONS`.

## ctl_recon_result

Stores reconciliation results at the metric level.

| Field | Meaning |
| --- | --- |
| recon_result_id | Unique result row |
| run_id | Related run |
| recon_name | Name of reconciliation test |
| business_date | Business date checked |
| metric_name | Metric being reconciled |
| source_value | External/source value |
| warehouse_value | Fabric warehouse value |
| variance_value | Dollar/absolute variance |
| variance_pct | Percentage variance |
| tolerance_value | Allowed dollar tolerance |
| tolerance_pct | Allowed percentage tolerance |
| status | PASS / FAIL / REVIEW |

## ctl_unresolved_member_match

Tracks Toast payments that cannot yet be confidently attributed to a PeopleVine member/account.

| Field | Meaning |
| --- | --- |
| unresolved_match_id | Unique unresolved record |
| payment_guid | Toast payment identifier |
| check_guid | Toast check identifier |
| order_guid | Toast order identifier |
| house_account_reference | House account reference from Toast |
| toast_tab_name | Tab/check name |
| payment_amount | Payment amount |
| candidate_member_id | Attempted member match |
| candidate_account_id | Attempted account match |
| match_confidence | Confidence score |
| unresolved_reason | Why match failed |
| status | Open / resolved / ignored |

## ctl_manual_override_member_match

Stores manual member/account attribution corrections.

| Field | Meaning |
| --- | --- |
| manual_override_id | Unique override row |
| payment_guid | Toast payment identifier |
| check_guid | Toast check identifier |
| order_guid | Toast order identifier |
| resolved_member_id | Final member assignment |
| resolved_account_id | Final account assignment |
| override_reason | Why override was made |
| effective_start_date | Start date of override |
| effective_end_date | End date of override |
| active_flag | Whether override is active |
| created_by | User who created override |

## ctl_data_quality_issue

Central issue table for data quality problems.

| Field | Meaning |
| --- | --- |
| dq_issue_id | Unique issue ID |
| severity | High / Medium / Low |
| issue_status | Open / resolved / ignored |
| layer_name | Bronze, Silver, Gold, etc. |
| table_name | Affected table |
| issue_type | Type of issue |
| issue_description | Description of problem |
| expected_value | Expected value |
| actual_value | Actual value |
| detected_at_utc | Detection timestamp |
| resolved_at_utc | Resolution timestamp |
| resolution_notes | Fix notes |

## Development Rule

No production pipeline or Gold mart should be considered complete unless its run is logged and its core row counts or validation checks are recorded in the appropriate control/audit tables.

## Current Status

The control table creation notebook exists and has been registered in the Query / Notebook Inventory.

Next improvement: add automated writes to `ctl_recon_result` once reconciliation thresholds are finalized.