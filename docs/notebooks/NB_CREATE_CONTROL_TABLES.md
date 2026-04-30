# NB_CREATE_CONTROL_TABLES

Artifact Type: Notebook
Build Method: Notebook
Dependencies: Fabric Lakehouse attached to notebook
Grain: One notebook run; creates multiple ctl_* tables
Last Updated: April 27, 2026
Layer: Control / Audit
Notes: Exact DDL provided in uploaded NB_CREATE_CONTROL_TABLES export.
Primary / Merge Key: Not applicable; table-specific keys documented on Control / Audit Tables page
Purpose: Create and confirm required control and audit Delta tables, then log the setup run.
Source Objects: Fabric Spark / Delta Lake
Status: Documented
Target Object: ctl_pipeline_run; ctl_watermark; ctl_endpoint_window_log; ctl_rowcount_audit; ctl_hash_change_audit; ctl_recon_result; ctl_unresolved_member_match; ctl_manual_override_member_match; ctl_data_quality_issue

## Notebook Purpose

Creates/Confirms required `ctl_*` control and audit tables, validates their existence, then logs the setup into `ctl_pipeline_run`.

## Notebook Code Source

Uploaded file: `NB_CREATE_CONTROL_TABLES.md.txt`

## Key Outputs

- `ctl_pipeline_run`
- `ctl_watermark`
- `ctl_endpoint_window_log`
- `ctl_rowcount_audit`
- `ctl_hash_change_audit`
- `ctl_recon_result`
- `ctl_unresolved_member_match`
- `ctl_manual_override_member_match`
- `ctl_data_quality_issue`

## Paste Notebook Code Here

```python
ddl_statements = [

"""
CREATE TABLE IF NOT EXISTS ctl_pipeline_run (
    run_id STRING,
    pipeline_name STRING,
    pipeline_stage STRING,
    run_type STRING,
    trigger_type STRING,
    source_system STRING,
    restaurant_guid STRING,
    status STRING,
    started_at_utc TIMESTAMP,
    ended_at_utc TIMESTAMP,
    duration_seconds BIGINT,
    rows_read BIGINT,
    rows_written BIGINT,
    rows_inserted BIGINT,
    rows_updated BIGINT,
    rows_deleted BIGINT,
    error_message STRING,
    created_by STRING,
    created_at_utc TIMESTAMP,
    updated_at_utc TIMESTAMP
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_watermark (
    source_system STRING,
    entity_name STRING,
    restaurant_guid STRING,
    watermark_type STRING,
    last_successful_start_utc TIMESTAMP,
    last_successful_end_utc TIMESTAMP,
    last_successful_business_date INT,
    last_successful_payload_hash STRING,
    last_run_id STRING,
    status STRING,
    notes STRING,
    created_at_utc TIMESTAMP,
    updated_at_utc TIMESTAMP
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_endpoint_window_log (
    window_log_id STRING,
    run_id STRING,
    source_system STRING,
    source_endpoint STRING,
    entity_name STRING,
    restaurant_guid STRING,
    request_window_start_utc TIMESTAMP,
    request_window_end_utc TIMESTAMP,
    request_business_date INT,
    request_url STRING,
    request_method STRING,
    response_status_code INT,
    response_record_count BIGINT,
    response_payload_hash STRING,
    started_at_utc TIMESTAMP,
    ended_at_utc TIMESTAMP,
    status STRING,
    error_message STRING,
    created_at_utc TIMESTAMP
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_rowcount_audit (
    rowcount_audit_id STRING,
    run_id STRING,
    layer_name STRING,
    table_name STRING,
    source_system STRING,
    source_entity STRING,
    restaurant_guid STRING,
    business_date INT,
    row_count BIGINT,
    distinct_key_count BIGINT,
    duplicate_key_count BIGINT,
    null_key_count BIGINT,
    audit_status STRING,
    checked_at_utc TIMESTAMP,
    notes STRING
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_hash_change_audit (
    hash_change_audit_id STRING,
    run_id STRING,
    layer_name STRING,
    table_name STRING,
    source_system STRING,
    source_entity STRING,
    restaurant_guid STRING,
    natural_key STRING,
    previous_hash STRING,
    current_hash STRING,
    change_type STRING,
    changed_at_utc TIMESTAMP,
    notes STRING
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_recon_result (
    recon_result_id STRING,
    run_id STRING,
    recon_name STRING,
    business_date INT,
    restaurant_guid STRING,
    metric_name STRING,
    metric_group STRING,
    source_name STRING,
    source_value DECIMAL(18,4),
    warehouse_value DECIMAL(18,4),
    variance_value DECIMAL(18,4),
    variance_pct DECIMAL(18,6),
    tolerance_value DECIMAL(18,4),
    tolerance_pct DECIMAL(18,6),
    status STRING,
    checked_at_utc TIMESTAMP,
    notes STRING
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_unresolved_member_match (
    unresolved_match_id STRING,
    run_id STRING,
    restaurant_guid STRING,
    payment_guid STRING,
    check_guid STRING,
    order_guid STRING,
    business_date INT,
    paid_business_date INT,
    house_account_reference STRING,
    toast_tab_name STRING,
    payment_amount DECIMAL(18,4),
    candidate_member_id STRING,
    candidate_account_id STRING,
    match_method_attempted STRING,
    match_confidence DECIMAL(9,6),
    unresolved_reason STRING,
    status STRING,
    first_seen_at_utc TIMESTAMP,
    last_seen_at_utc TIMESTAMP,
    resolved_at_utc TIMESTAMP,
    resolution_method STRING,
    notes STRING
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_manual_override_member_match (
    manual_override_id STRING,
    restaurant_guid STRING,
    payment_guid STRING,
    check_guid STRING,
    order_guid STRING,
    house_account_reference STRING,
    toast_tab_name STRING,
    resolved_member_id STRING,
    resolved_account_id STRING,
    override_reason STRING,
    effective_start_date DATE,
    effective_end_date DATE,
    active_flag BOOLEAN,
    created_by STRING,
    created_at_utc TIMESTAMP,
    updated_by STRING,
    updated_at_utc TIMESTAMP,
    notes STRING
)
USING DELTA
""",

"""
CREATE TABLE IF NOT EXISTS ctl_data_quality_issue (
    dq_issue_id STRING,
    run_id STRING,
    severity STRING,
    issue_status STRING,
    layer_name STRING,
    table_name STRING,
    source_system STRING,
    source_entity STRING,
    restaurant_guid STRING,
    business_date INT,
    natural_key STRING,
    issue_type STRING,
    issue_description STRING,
    expected_value STRING,
    actual_value STRING,
    detected_at_utc TIMESTAMP,
    resolved_at_utc TIMESTAMP,
    resolution_notes STRING
)
USING DELTA
"""

]

for ddl in ddl_statements:
    spark.sql(ddl)

print("✅ Control and audit tables created/confirmed successfully.")
```

```python
required_tables = [
    "ctl_pipeline_run",
    "ctl_watermark",
    "ctl_endpoint_window_log",
    "ctl_rowcount_audit",
    "ctl_hash_change_audit",
    "ctl_recon_result",
    "ctl_unresolved_member_match",
    "ctl_manual_override_member_match",
    "ctl_data_quality_issue"
]

existing_tables = [row.tableName for row in spark.sql("SHOW TABLES").collect()]
missing_tables = [table for table in required_tables if table not in existing_tables]

if missing_tables:
    print("❌ Missing control tables:")
    for table in missing_tables:
        print(table)
else:
    print("✅ All required control/audit tables exist.")

display(
    spark.sql("SHOW TABLES")
    .filter("tableName LIKE 'ctl_%'")
    .select("tableName", "namespace", "isTemporary")
    .orderBy("tableName")
)
```

```python
from datetime import datetime
import uuid

run_id = str(uuid.uuid4())
started_at_utc = datetime.utcnow()
ended_at_utc = datetime.utcnow()

required_tables = [
    "ctl_pipeline_run",
    "ctl_watermark",
    "ctl_endpoint_window_log",
    "ctl_rowcount_audit",
    "ctl_hash_change_audit",
    "ctl_recon_result",
    "ctl_unresolved_member_match",
    "ctl_manual_override_member_match",
    "ctl_data_quality_issue"
]

existing_tables = [row.tableName for row in spark.sql("SHOW TABLES").collect()]
missing_tables = [table for table in required_tables if table not in existing_tables]

status = "FAILED" if missing_tables else "SUCCEEDED"
error_message = "Missing tables: " + ", ".join(missing_tables) if missing_tables else None
safe_error_message = error_message.replace("'", "''") if error_message else None

spark.sql(f"""
    INSERT INTO ctl_pipeline_run
    SELECT
        '{run_id}' AS run_id,
        'NB_CREATE_CONTROL_TABLES' AS pipeline_name,
        'control_table_setup' AS pipeline_stage,
        'manual_setup' AS run_type,
        'manual' AS trigger_type,
        'fabric' AS source_system,
        NULL AS restaurant_guid,
        '{status}' AS status,
        TIMESTAMP('{started_at_utc}') AS started_at_utc,
        TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
        0 AS duration_seconds,
        0 AS rows_read,
        {len(required_tables) - len(missing_tables)} AS rows_written,
        {len(required_tables) - len(missing_tables)} AS rows_inserted,
        0 AS rows_updated,
        0 AS rows_deleted,
        {f"'{safe_error_message}'" if safe_error_message else "NULL"} AS error_message,
        current_user() AS created_by,
        current_timestamp() AS created_at_utc,
        current_timestamp() AS updated_at_utc
""")

print(f"✅ Control-table setup logged. run_id={run_id}")
```

## Validation Notes

- All required control tables exist: ☐
- Setup logged to `ctl_pipeline_run`: ☐
- No schema drift from Control / Audit Tables page: ☐