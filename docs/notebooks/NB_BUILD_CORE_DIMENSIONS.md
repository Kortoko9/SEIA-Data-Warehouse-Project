# NB_BUILD_CORE_DIMENSIONS

Artifact Type: Notebook
Build Method: Notebook
Dependencies: Bronze Toast config/menu tables; control/audit tables
Grain: One row per dimension key; bridge grain varies by bridge table
Last Updated: April 27, 2026
Layer: Silver
Notes: Uses Delta merge/upsert, row hashes, rowcount audits, and hash-change audits.
Primary / Merge Key: Table-specific keys documented in Data Dictionary / Control notes
Purpose: Build and merge core restaurant, config, menu, item, modifier, and bridge dimensions from Toast Bronze tables.
Source Objects: brz_toast_restaurants; brz_toast_config_*; brz_toast_menus; ctl_rowcount_audit; ctl_hash_change_audit; ctl_pipeline_run
Status: Documented
Target Object: stg_dim_restaurant; stg_dim_revenue_center; stg_dim_sales_category; stg_dim_dining_option; stg_dim_discount; stg_dim_service_charge; stg_dim_tax_rate; stg_dim_alt_payment_type; stg_dim_menu_group; stg_dim_menu_item; stg_dim_modifier_group; stg_dim_modifier_option; stg_bridge_menu_item_modifier_group; stg_bridge_modifier_group_option

## Notebook Purpose

Builds Silver dimension and reference tables from Bronze Toast config/menu payloads using staged DataFrames, row hashes, Delta merge/upsert, rowcount audit, and hash-change audit.

## Notebook Code Source

Uploaded file: `NB_BUILD_CORE_DIMENSIONS.md.txt`

## Key Outputs

- Restaurant/config dimensions
- Menu group/item dimensions
- Modifier group/option dimensions
- Menu item → modifier group bridge
- Modifier group → option bridge

## Paste Notebook Code Here

```python
from pyspark.sql import functions as F
from pyspark.sql import types as T
from pyspark.sql.window import Window
from delta.tables import DeltaTable
from datetime import datetime
import json
import uuid

run_id = str(uuid.uuid4())
started_at_utc = datetime.utcnow()

def now_utc():
    return datetime.utcnow()

def safe_json_loads(value):
    if value is None:
        return None
    if isinstance(value, (dict, list)):
        return value
    try:
        return json.loads(value)
    except Exception:
        return None

def parse_ts(value):
    if value is None:
        return None
    if isinstance(value, datetime):
        return value.replace(tzinfo=None)
    try:
        return datetime.fromisoformat(str(value).replace("Z", "+00:00")).replace(tzinfo=None)
    except Exception:
        return None

def stringify_value(value):
    if value is None:
        return None
    if isinstance(value, (dict, list)):
        return json.dumps(value, sort_keys=True)
    if isinstance(value, bool):
        return str(value).lower()
    return str(value)

def get_path(record, path):
    current = record
    for part in path.split("."):
        if not isinstance(current, dict):
            return None
        current = current.get(part)
        if current is None:
            return None
    return current

def get_any(record, paths):
    if not isinstance(record, dict):
        return None
    for path in paths:
        value = get_path(record, path)
        if value is not None:
            return stringify_value(value)
    return None

def table_exists(table_name):
    return spark.catalog.tableExists(table_name)

def make_schema(cols):
    timestamp_cols = {
        "source_ingested_at_utc",
        "first_seen_at_utc",
        "last_seen_at_utc",
        "created_at_utc",
        "updated_at_utc"
    }

    return T.StructType([
        T.StructField(col, T.TimestampType() if col in timestamp_cols else T.StringType(), True)
        for col in cols
    ])

def dict_list(value):
    if isinstance(value, list):
        return [x for x in value if isinstance(x, dict)]
    return []

def ref_collection(value):
    refs = []

    if isinstance(value, list):
        for idx, item in enumerate(value):
            if isinstance(item, dict):
                copied = dict(item)
                copied["_referenceIndex"] = str(idx)
                copied["_referenceIndexOneBased"] = str(idx + 1)
                refs.append(copied)

    elif isinstance(value, dict):
        for key, item in value.items():
            if isinstance(item, dict):
                copied = dict(item)
                copied["_referenceKey"] = str(key)
                copied["_referenceIdFromKey"] = str(key)
                refs.append(copied)

    return refs

def ref_values(value):
    if not isinstance(value, list):
        return []

    output = []

    for item in value:
        if isinstance(item, dict):
            ref = (
                get_any(item, ["referenceId", "guid", "id", "masterId", "multiLocationId"])
                or stringify_value(item)
            )
            output.append(ref)
        else:
            output.append(stringify_value(item))

    return [x for x in output if x is not None]

def build_reference_lookup(refs):
    lookup = {}

    for idx, ref in enumerate(refs):
        possible_keys = set()

        for path in [
            "_referenceKey",
            "_referenceIdFromKey",
            "_referenceIndex",
            "_referenceIndexOneBased",
            "referenceId",
            "guid",
            "id",
            "masterId",
            "multiLocationId"
        ]:
            value = get_any(ref, [path])
            if value is not None:
                possible_keys.add(value)

        possible_keys.add(str(idx))
        possible_keys.add(str(idx + 1))

        for key in possible_keys:
            lookup[key] = ref

    return lookup

def get_stable_guid(ref, ref_id, prefix):
    guid = get_any(ref, ["guid", "id", "masterId", "multiLocationId"])

    if guid:
        return guid

    if ref_id:
        return f"{prefix}_ref_{ref_id}"

    return None

def append_rowcount_audit(table_name, source_entity, restaurant_guid, stage_df, key_cols, audit_status, notes):
    row_count = stage_df.count()

    if row_count > 0:
        distinct_key_count = stage_df.select(*key_cols).distinct().count()
        duplicate_key_count = row_count - distinct_key_count

        null_condition = None
        for key_col in key_cols:
            condition = F.col(key_col).isNull()
            null_condition = condition if null_condition is None else (null_condition | condition)

        null_key_count = stage_df.filter(null_condition).count()
    else:
        distinct_key_count = 0
        duplicate_key_count = 0
        null_key_count = 0

    audit_schema = T.StructType([
        T.StructField("rowcount_audit_id", T.StringType(), True),
        T.StructField("run_id", T.StringType(), True),
        T.StructField("layer_name", T.StringType(), True),
        T.StructField("table_name", T.StringType(), True),
        T.StructField("source_system", T.StringType(), True),
        T.StructField("source_entity", T.StringType(), True),
        T.StructField("restaurant_guid", T.StringType(), True),
        T.StructField("business_date", T.IntegerType(), True),
        T.StructField("row_count", T.LongType(), True),
        T.StructField("distinct_key_count", T.LongType(), True),
        T.StructField("duplicate_key_count", T.LongType(), True),
        T.StructField("null_key_count", T.LongType(), True),
        T.StructField("audit_status", T.StringType(), True),
        T.StructField("checked_at_utc", T.TimestampType(), True),
        T.StructField("notes", T.StringType(), True)
    ])

    audit_row = [(
        str(uuid.uuid4()),
        run_id,
        "silver",
        table_name,
        "toast",
        source_entity,
        restaurant_guid,
        None,
        row_count,
        distinct_key_count,
        duplicate_key_count,
        null_key_count,
        audit_status,
        now_utc(),
        notes
    )]

    spark.createDataFrame(audit_row, audit_schema).write.format("delta").mode("append").saveAsTable("ctl_rowcount_audit")

def append_hash_change_audit(table_name, source_entity, stage_df, key_cols):
    if stage_df.count() == 0:
        return

    if table_exists(table_name):
        existing_df = spark.table(table_name).select(*key_cols, F.col("row_hash").alias("previous_hash"))

        joined_df = (
            stage_df.alias("s")
            .join(existing_df.alias("t"), on=key_cols, how="left")
            .withColumn(
                "change_type",
                F.when(F.col("previous_hash").isNull(), F.lit("INSERT"))
                 .when(F.col("previous_hash") != F.col("s.row_hash"), F.lit("UPDATE"))
                 .otherwise(F.lit("NO_CHANGE"))
            )
            .filter(F.col("change_type") != "NO_CHANGE")
        )
    else:
        joined_df = (
            stage_df
            .withColumn("previous_hash", F.lit(None).cast("string"))
            .withColumn("change_type", F.lit("INSERT"))
        )

    if joined_df.count() == 0:
        return

    audit_df = (
        joined_df
        .withColumn("hash_change_audit_id", F.expr("uuid()"))
        .withColumn("run_id", F.lit(run_id))
        .withColumn("layer_name", F.lit("silver"))
        .withColumn("table_name", F.lit(table_name))
        .withColumn("source_system", F.lit("toast"))
        .withColumn("source_entity", F.lit(source_entity))
        .withColumn("natural_key", F.concat_ws("||", *[F.col(c).cast("string") for c in key_cols]))
        .withColumn("current_hash", F.col("row_hash"))
        .withColumn("changed_at_utc", F.current_timestamp())
        .withColumn("notes", F.lit(None).cast("string"))
        .select(
            "hash_change_audit_id",
            "run_id",
            "layer_name",
            "table_name",
            "source_system",
            "source_entity",
            "restaurant_guid",
            "natural_key",
            "previous_hash",
            "current_hash",
            "change_type",
            "changed_at_utc",
            "notes"
        )
    )

    audit_df.write.format("delta").mode("append").saveAsTable("ctl_hash_change_audit")

def enrich_with_hash_and_metadata(df):
    metadata_cols = {
        "row_hash",
        "source_system",
        "source_endpoint",
        "record_source",
        "source_payload_hash",
        "source_ingested_at_utc",
        "payload_record_json",
        "first_seen_at_utc",
        "last_seen_at_utc",
        "created_at_utc",
        "updated_at_utc",
        "run_id"
    }

    business_hash_cols = [c for c in df.columns if c not in metadata_cols]

    return (
        df
        .withColumn(
            "row_hash",
            F.sha2(
                F.concat_ws(
                    "||",
                    *[F.coalesce(F.col(c).cast("string"), F.lit("")) for c in business_hash_cols + ["payload_record_json"]]
                ),
                256
            )
        )
        .withColumn("first_seen_at_utc", F.current_timestamp())
        .withColumn("last_seen_at_utc", F.current_timestamp())
        .withColumn("created_at_utc", F.current_timestamp())
        .withColumn("updated_at_utc", F.current_timestamp())
        .withColumn("run_id", F.lit(run_id))
    )

def merge_table(table_name, source_entity, stage_df, key_cols, table_type_note):
    stage_count = stage_df.count()

    if stage_count == 0:
        empty_enriched_df = enrich_with_hash_and_metadata(stage_df)

        if not table_exists(table_name):
            empty_enriched_df.write.format("delta").mode("overwrite").saveAsTable(table_name)

        append_rowcount_audit(
            table_name=table_name,
            source_entity=source_entity,
            restaurant_guid=None,
            stage_df=stage_df,
            key_cols=key_cols,
            audit_status="WARNING",
            notes=f"No rows staged for {table_type_note}; empty table created/confirmed."
        )

        print(f"⚠️ {table_name}: no staged rows; empty table exists.")
        return

    window_spec = Window.partitionBy(*key_cols).orderBy(F.col("source_ingested_at_utc").desc_nulls_last())

    deduped_df = (
        stage_df
        .withColumn("_rn", F.row_number().over(window_spec))
        .filter(F.col("_rn") == 1)
        .drop("_rn")
    )

    deduped_df = enrich_with_hash_and_metadata(deduped_df)

    append_hash_change_audit(table_name, source_entity, deduped_df, key_cols)

    if not table_exists(table_name):
        deduped_df.write.format("delta").mode("overwrite").saveAsTable(table_name)
    else:
        delta_table = DeltaTable.forName(spark, table_name)
        merge_condition = " AND ".join([f"t.{c} = s.{c}" for c in key_cols])

        update_changed_set = {
            c: f"s.{c}" for c in deduped_df.columns
            if c not in ["first_seen_at_utc", "created_at_utc"]
        }

        update_changed_set["first_seen_at_utc"] = "t.first_seen_at_utc"
        update_changed_set["created_at_utc"] = "t.created_at_utc"

        update_same_set = {
            "last_seen_at_utc": "s.last_seen_at_utc",
            "updated_at_utc": "s.updated_at_utc",
            "source_ingested_at_utc": "s.source_ingested_at_utc",
            "run_id": "s.run_id"
        }

        (
            delta_table.alias("t")
            .merge(deduped_df.alias("s"), merge_condition)
            .whenMatchedUpdate(
                condition="t.row_hash <> s.row_hash",
                set=update_changed_set
            )
            .whenMatchedUpdate(
                set=update_same_set
            )
            .whenNotMatchedInsertAll()
            .execute()
        )

    append_rowcount_audit(
        table_name=table_name,
        source_entity=source_entity,
        restaurant_guid=None,
        stage_df=deduped_df,
        key_cols=key_cols,
        audit_status="PASS",
        notes=f"{table_type_note} merge/upsert completed."
    )

    print(f"✅ {table_name}: merge/upsert complete. Staged rows: {deduped_df.count()}")
```

```python
def normalize_payload_to_records(payload):
    obj = safe_json_loads(payload)

    if obj is None:
        return []

    if isinstance(obj, list):
        return [x for x in obj if isinstance(x, dict)]

    if isinstance(obj, dict):
        for key in ["data", "items", "results", "value"]:
            if isinstance(obj.get(key), list):
                return [x for x in obj[key] if isinstance(x, dict)]
        return [obj]

    return []

def pick_any(record, paths):
    return get_any(record, paths)

def build_dimension_from_bronze(config):
    bronze_table = config["bronze_table"]
    target_table = config["target_table"]
    key_cols = config["key_cols"]
    mappings = config["mappings"]

    bronze_df = spark.table(bronze_table).select(
        "ingested_at_utc",
        "source_system",
        "source_endpoint",
        "restaurant_guid",
        "record_source",
        "payload_hash",
        "payload_json"
    )

    output_cols = config["output_cols"] + [
        "source_system",
        "source_endpoint",
        "record_source",
        "source_payload_hash",
        "source_ingested_at_utc",
        "payload_record_json"
    ]

    output_rows = []

    for bronze_row in bronze_df.collect():
        row = bronze_row.asDict(recursive=True)
        records = normalize_payload_to_records(row.get("payload_json"))

        for record in records:
            out = {}

            for target_col, paths in mappings.items():
                out[target_col] = pick_any(record, paths)

            if "restaurant_guid" in config["output_cols"]:
                out["restaurant_guid"] = out.get("restaurant_guid") or row.get("restaurant_guid")

            entity_guid_col = config.get("entity_guid_col")
            if entity_guid_col and not out.get(entity_guid_col):
                continue

            out["source_system"] = row.get("source_system")
            out["source_endpoint"] = row.get("source_endpoint")
            out["record_source"] = row.get("record_source")
            out["source_payload_hash"] = row.get("payload_hash")
            out["source_ingested_at_utc"] = parse_ts(row.get("ingested_at_utc"))
            out["payload_record_json"] = json.dumps(record, sort_keys=True)

            output_rows.append(tuple(out.get(c) for c in output_cols))

    stage_df = spark.createDataFrame(output_rows, schema=make_schema(output_cols))

    merge_table(
        table_name=target_table,
        source_entity=bronze_table,
        stage_df=stage_df,
        key_cols=key_cols,
        table_type_note=f"{target_table} dimension"
    )

dimension_configs = [
    {
        "bronze_table": "brz_toast_restaurants",
        "target_table": "stg_dim_restaurant",
        "key_cols": ["restaurant_guid"],
        "entity_guid_col": None,
        "output_cols": [
            "restaurant_guid",
            "restaurant_name",
            "location_name",
            "location_code",
            "time_zone",
            "closeout_hour"
        ],
        "mappings": {
            "restaurant_guid": ["guid", "restaurantGuid"],
            "restaurant_name": ["general.name", "name"],
            "location_name": ["general.locationName", "locationName"],
            "location_code": ["general.locationCode", "locationCode"],
            "time_zone": ["general.timeZone", "timeZone"],
            "closeout_hour": ["general.closeoutHour", "closeoutHour"]
        }
    },
    {
        "bronze_table": "brz_toast_config_revenue_centers",
        "target_table": "stg_dim_revenue_center",
        "key_cols": ["restaurant_guid", "revenue_center_guid"],
        "entity_guid_col": "revenue_center_guid",
        "output_cols": [
            "restaurant_guid",
            "revenue_center_guid",
            "revenue_center_name",
            "description"
        ],
        "mappings": {
            "revenue_center_guid": ["guid"],
            "revenue_center_name": ["name"],
            "description": ["description"]
        }
    },
    {
        "bronze_table": "brz_toast_config_sales_categories",
        "target_table": "stg_dim_sales_category",
        "key_cols": ["restaurant_guid", "sales_category_guid"],
        "entity_guid_col": "sales_category_guid",
        "output_cols": [
            "restaurant_guid",
            "sales_category_guid",
            "sales_category_name"
        ],
        "mappings": {
            "sales_category_guid": ["guid"],
            "sales_category_name": ["name"]
        }
    },
    {
        "bronze_table": "brz_toast_config_dining_options",
        "target_table": "stg_dim_dining_option",
        "key_cols": ["restaurant_guid", "dining_option_guid"],
        "entity_guid_col": "dining_option_guid",
        "output_cols": [
            "restaurant_guid",
            "dining_option_guid",
            "dining_option_name",
            "behavior",
            "curbside"
        ],
        "mappings": {
            "dining_option_guid": ["guid"],
            "dining_option_name": ["name"],
            "behavior": ["behavior"],
            "curbside": ["curbside"]
        }
    },
    {
        "bronze_table": "brz_toast_config_discounts",
        "target_table": "stg_dim_discount",
        "key_cols": ["restaurant_guid", "discount_guid"],
        "entity_guid_col": "discount_guid",
        "output_cols": [
            "restaurant_guid",
            "discount_guid",
            "discount_name",
            "active_flag",
            "discount_type",
            "percentage",
            "amount"
        ],
        "mappings": {
            "discount_guid": ["guid"],
            "discount_name": ["name"],
            "active_flag": ["active"],
            "discount_type": ["type"],
            "percentage": ["percentage"],
            "amount": ["amount"]
        }
    },
    {
        "bronze_table": "brz_toast_config_service_charges",
        "target_table": "stg_dim_service_charge",
        "key_cols": ["restaurant_guid", "service_charge_guid"],
        "entity_guid_col": "service_charge_guid",
        "output_cols": [
            "restaurant_guid",
            "service_charge_guid",
            "service_charge_name",
            "amount_type",
            "amount",
            "percent",
            "gratuity_flag",
            "taxable_flag",
            "destination"
        ],
        "mappings": {
            "service_charge_guid": ["guid"],
            "service_charge_name": ["name"],
            "amount_type": ["amountType"],
            "amount": ["amount"],
            "percent": ["percent"],
            "gratuity_flag": ["gratuity"],
            "taxable_flag": ["taxable"],
            "destination": ["destination"]
        }
    },
    {
        "bronze_table": "brz_toast_config_tax_rates",
        "target_table": "stg_dim_tax_rate",
        "key_cols": ["restaurant_guid", "tax_rate_guid"],
        "entity_guid_col": "tax_rate_guid",
        "output_cols": [
            "restaurant_guid",
            "tax_rate_guid",
            "tax_rate_name",
            "rate",
            "tax_type",
            "rounding_type"
        ],
        "mappings": {
            "tax_rate_guid": ["guid"],
            "tax_rate_name": ["name"],
            "rate": ["rate"],
            "tax_type": ["type"],
            "rounding_type": ["roundingType"]
        }
    },
    {
        "bronze_table": "brz_toast_config_alt_payment_types",
        "target_table": "stg_dim_alt_payment_type",
        "key_cols": ["restaurant_guid", "alt_payment_type_guid"],
        "entity_guid_col": "alt_payment_type_guid",
        "output_cols": [
            "restaurant_guid",
            "alt_payment_type_guid",
            "alt_payment_type_name",
            "payment_type",
            "active_flag"
        ],
        "mappings": {
            "alt_payment_type_guid": ["guid"],
            "alt_payment_type_name": ["name"],
            "payment_type": ["type"],
            "active_flag": ["active"]
        }
    }
]

try:
    for config in dimension_configs:
        build_dimension_from_bronze(config)

    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_config_dimension_merge' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'SUCCEEDED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            NULL AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    print(f"✅ Restaurant/config dimensions completed successfully. run_id={run_id}")

except Exception as e:
    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())
    safe_error = str(e).replace("'", "''")[:4000]

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_config_dimension_merge' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'FAILED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            '{safe_error}' AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    raise
```

```python
def normalize_full_menu_payload(payload):
    obj = safe_json_loads(payload)

    if obj is None:
        return []

    if isinstance(obj, dict) and isinstance(obj.get("menus"), list):
        return [obj]

    if isinstance(obj, list):
        return [x for x in obj if isinstance(x, dict) and isinstance(x.get("menus"), list)]

    return []

def extract_menu_group_item_dimensions():
    bronze_df = spark.table("brz_toast_menus").select(
        "ingested_at_utc",
        "source_system",
        "source_endpoint",
        "restaurant_guid",
        "record_source",
        "payload_hash",
        "payload_json"
    )

    menu_group_rows = []
    menu_item_rows = []

    for bronze_row in bronze_df.collect():
        b = bronze_row.asDict(recursive=True)

        restaurant_guid = b.get("restaurant_guid")
        source_system = b.get("source_system")
        source_endpoint = b.get("source_endpoint")
        record_source = b.get("record_source")
        source_payload_hash = b.get("payload_hash")
        source_ingested_at_utc = parse_ts(b.get("ingested_at_utc"))

        payloads = normalize_full_menu_payload(b.get("payload_json"))

        for payload in payloads:
            for menu in dict_list(payload.get("menus")):
                menu_guid = get_any(menu, ["guid", "id", "menuGuid"])
                menu_name = get_any(menu, ["name"])

                menu_groups = dict_list(menu.get("menuGroups")) or dict_list(menu.get("groups"))

                for group in menu_groups:
                    menu_group_guid = get_any(group, ["guid", "id", "menuGroupGuid"])
                    menu_group_name = get_any(group, ["name"])
                    parent_menu_group_guid = get_any(group, ["parentGuid", "parent.guid"])

                    menu_items = dict_list(group.get("menuItems")) or dict_list(group.get("items"))

                    if menu_group_guid:
                        menu_group_rows.append((
                            restaurant_guid,
                            menu_guid,
                            menu_name,
                            menu_group_guid,
                            menu_group_name,
                            parent_menu_group_guid,
                            stringify_value(group.get("visibility")),
                            stringify_value(group.get("orderableOnline")),
                            stringify_value(len(menu_items)),
                            source_system,
                            source_endpoint,
                            record_source,
                            source_payload_hash,
                            source_ingested_at_utc,
                            json.dumps(group, sort_keys=True)
                        ))

                    for item in menu_items:
                        menu_item_guid = get_any(item, ["guid", "id", "itemGuid", "menuItemGuid"])
                        menu_item_name = get_any(item, ["name"])

                        if menu_item_guid:
                            menu_item_rows.append((
                                restaurant_guid,
                                menu_guid,
                                menu_name,
                                menu_group_guid,
                                menu_group_name,
                                menu_item_guid,
                                menu_item_name,
                                get_any(item, ["description"]),
                                get_any(item, ["salesCategory.guid", "salesCategoryGuid", "salesCategory.id"]),
                                get_any(item, ["price", "pricingRules.price", "basePrice"]),
                                get_any(item, ["pricingStrategy", "priceType"]),
                                get_any(item, ["sku"]),
                                get_any(item, ["plu"]),
                                stringify_value(item.get("visibility")),
                                stringify_value(item.get("orderableOnline")),
                                source_system,
                                source_endpoint,
                                record_source,
                                source_payload_hash,
                                source_ingested_at_utc,
                                json.dumps(item, sort_keys=True)
                            ))

    return menu_group_rows, menu_item_rows

menu_group_cols = [
    "restaurant_guid",
    "menu_guid",
    "menu_name",
    "menu_group_guid",
    "menu_group_name",
    "parent_menu_group_guid",
    "visibility",
    "orderable_online_flag",
    "item_count",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

menu_item_cols = [
    "restaurant_guid",
    "menu_guid",
    "menu_name",
    "menu_group_guid",
    "menu_group_name",
    "menu_item_guid",
    "menu_item_name",
    "description",
    "sales_category_guid",
    "price",
    "pricing_strategy",
    "sku",
    "plu",
    "visibility",
    "orderable_online_flag",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

try:
    menu_group_rows, menu_item_rows = extract_menu_group_item_dimensions()

    menu_group_df = spark.createDataFrame(menu_group_rows, make_schema(menu_group_cols))
    menu_item_df = spark.createDataFrame(menu_item_rows, make_schema(menu_item_cols))

    merge_table(
        table_name="stg_dim_menu_group",
        source_entity="brz_toast_menus",
        stage_df=menu_group_df,
        key_cols=["restaurant_guid", "menu_group_guid"],
        table_type_note="menu group dimension"
    )

    merge_table(
        table_name="stg_dim_menu_item",
        source_entity="brz_toast_menus",
        stage_df=menu_item_df,
        key_cols=["restaurant_guid", "menu_item_guid"],
        table_type_note="menu item dimension"
    )

    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_menu_group_item_dimension_merge' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'SUCCEEDED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            NULL AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    print(f"✅ Menu group/item dimensions completed successfully. run_id={run_id}")

except Exception as e:
    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())
    safe_error = str(e).replace("'", "''")[:4000]

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_menu_group_item_dimension_merge' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'FAILED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            '{safe_error}' AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    raise
```

```python
# ============================================================
# NB2 — Cell 5
# Build modifier dimensions + bridge tables from Toast menu refs
# ============================================================

def normalize_modifier_menu_payload(payload):
    obj = safe_json_loads(payload)

    if obj is None:
        return []

    # Expected Toast /menus shape:
    # {
    #   restaurantGuid,
    #   menus: [...],
    #   modifierGroupReferences: {... or [...]},
    #   modifierOptionReferences: {... or [...]}
    # }
    if isinstance(obj, dict) and isinstance(obj.get("menus"), list):
        return [obj]

    if isinstance(obj, list):
        return [
            x for x in obj
            if isinstance(x, dict) and isinstance(x.get("menus"), list)
        ]

    return []

def extract_modifier_reference_tables_v2():
    bronze_df = spark.table("brz_toast_menus").select(
        "ingested_at_utc",
        "source_system",
        "source_endpoint",
        "restaurant_guid",
        "record_source",
        "payload_hash",
        "payload_json"
    )

    modifier_group_rows = []
    modifier_option_rows = []
    item_modifier_group_bridge_rows = []
    modifier_group_option_bridge_rows = []

    diagnostic_counts = []

    for bronze_row in bronze_df.collect():
        b = bronze_row.asDict(recursive=True)

        restaurant_guid = b.get("restaurant_guid")
        source_system = b.get("source_system")
        source_endpoint = b.get("source_endpoint")
        record_source = b.get("record_source")
        source_payload_hash = b.get("payload_hash")
        source_ingested_at_utc = parse_ts(b.get("ingested_at_utc"))

        payloads = normalize_modifier_menu_payload(b.get("payload_json"))

        for payload in payloads:
            modifier_group_refs = ref_collection(payload.get("modifierGroupReferences"))
            modifier_option_refs = ref_collection(payload.get("modifierOptionReferences"))

            diagnostic_counts.append((
                restaurant_guid,
                len(modifier_group_refs),
                len(modifier_option_refs)
            ))

            modifier_group_lookup = build_reference_lookup(modifier_group_refs)
            modifier_option_lookup = build_reference_lookup(modifier_option_refs)

            # ----------------------------
            # Modifier group dimension
            # ----------------------------
            for idx, mod_group in enumerate(modifier_group_refs):
                modifier_group_reference_id = (
                    get_any(
                        mod_group,
                        [
                            "referenceId",
                            "_referenceIdFromKey",
                            "_referenceKey",
                            "_referenceIndex",
                            "_referenceIndexOneBased"
                        ]
                    )
                    or str(idx)
                )

                modifier_group_guid = get_stable_guid(
                    mod_group,
                    modifier_group_reference_id,
                    "modifier_group"
                )

                modifier_option_ref_values = ref_values(
                    mod_group.get("modifierOptionReferences")
                )

                if modifier_group_guid:
                    modifier_group_rows.append((
                        restaurant_guid,
                        modifier_group_guid,
                        modifier_group_reference_id,
                        get_any(mod_group, ["name"]),
                        get_any(mod_group, ["posName"]),
                        get_any(mod_group, ["minSelections", "minimumSelections", "min"]),
                        get_any(mod_group, ["maxSelections", "maximumSelections", "max"]),
                        get_any(mod_group, ["required", "isRequired"]),
                        get_any(mod_group, ["multiSelect", "allowsMultipleSelections"]),
                        get_any(mod_group, ["pricingMode"]),
                        stringify_value(len(modifier_option_ref_values)),
                        source_system,
                        source_endpoint,
                        record_source,
                        source_payload_hash,
                        source_ingested_at_utc,
                        json.dumps(mod_group, sort_keys=True)
                    ))

                # ----------------------------
                # Modifier group → option bridge
                # ----------------------------
                for option_position, option_ref_id in enumerate(modifier_option_ref_values):
                    option_obj = modifier_option_lookup.get(option_ref_id)

                    if not option_obj:
                        continue

                    modifier_option_reference_id = (
                        get_any(
                            option_obj,
                            [
                                "referenceId",
                                "_referenceIdFromKey",
                                "_referenceKey",
                                "_referenceIndex",
                                "_referenceIndexOneBased"
                            ]
                        )
                        or option_ref_id
                    )

                    modifier_option_guid = get_stable_guid(
                        option_obj,
                        modifier_option_reference_id,
                        "modifier_option"
                    )

                    if modifier_group_guid and modifier_option_guid:
                        modifier_group_option_bridge_rows.append((
                            restaurant_guid,
                            modifier_group_guid,
                            modifier_group_reference_id,
                            get_any(mod_group, ["name"]),
                            modifier_option_guid,
                            modifier_option_reference_id,
                            get_any(option_obj, ["name"]),
                            stringify_value(option_position),
                            source_system,
                            source_endpoint,
                            record_source,
                            source_payload_hash,
                            source_ingested_at_utc,
                            json.dumps(
                                {
                                    "modifierGroup": mod_group,
                                    "modifierOption": option_obj,
                                    "modifierOptionReferenceId": option_ref_id,
                                    "optionPosition": option_position
                                },
                                sort_keys=True
                            )
                        ))

            # ----------------------------
            # Modifier option dimension
            # ----------------------------
            for idx, option in enumerate(modifier_option_refs):
                modifier_option_reference_id = (
                    get_any(
                        option,
                        [
                            "referenceId",
                            "_referenceIdFromKey",
                            "_referenceKey",
                            "_referenceIndex",
                            "_referenceIndexOneBased"
                        ]
                    )
                    or str(idx)
                )

                modifier_option_guid = get_stable_guid(
                    option,
                    modifier_option_reference_id,
                    "modifier_option"
                )

                if modifier_option_guid:
                    modifier_option_rows.append((
                        restaurant_guid,
                        modifier_option_guid,
                        modifier_option_reference_id,
                        get_any(option, ["name"]),
                        get_any(option, ["posName"]),
                        get_any(option, ["price"]),
                        get_any(option, ["pricingStrategy", "priceType"]),
                        get_any(option, ["salesCategory.guid", "salesCategoryGuid"]),
                        get_any(option, ["sku"]),
                        get_any(option, ["plu"]),
                        stringify_value(option.get("visibility")),
                        source_system,
                        source_endpoint,
                        record_source,
                        source_payload_hash,
                        source_ingested_at_utc,
                        json.dumps(option, sort_keys=True)
                    ))

            # ----------------------------
            # Menu item → modifier group bridge
            # ----------------------------
            for menu in dict_list(payload.get("menus")):
                menu_guid = get_any(menu, ["guid", "id", "menuGuid"])
                menu_name = get_any(menu, ["name"])
                menu_groups = dict_list(menu.get("menuGroups")) or dict_list(menu.get("groups"))

                for menu_group in menu_groups:
                    menu_group_guid = get_any(menu_group, ["guid", "id", "menuGroupGuid"])
                    menu_group_name = get_any(menu_group, ["name"])
                    menu_items = dict_list(menu_group.get("menuItems")) or dict_list(menu_group.get("items"))

                    for item in menu_items:
                        menu_item_guid = get_any(item, ["guid", "id", "itemGuid", "menuItemGuid"])
                        menu_item_name = get_any(item, ["name"])
                        item_modifier_group_ref_values = ref_values(item.get("modifierGroupReferences"))

                        for modifier_group_position, modifier_group_ref_id in enumerate(item_modifier_group_ref_values):
                            mod_group_obj = modifier_group_lookup.get(modifier_group_ref_id)

                            if not mod_group_obj:
                                continue

                            modifier_group_reference_id = (
                                get_any(
                                    mod_group_obj,
                                    [
                                        "referenceId",
                                        "_referenceIdFromKey",
                                        "_referenceKey",
                                        "_referenceIndex",
                                        "_referenceIndexOneBased"
                                    ]
                                )
                                or modifier_group_ref_id
                            )

                            modifier_group_guid = get_stable_guid(
                                mod_group_obj,
                                modifier_group_reference_id,
                                "modifier_group"
                            )

                            if menu_item_guid and modifier_group_guid:
                                item_modifier_group_bridge_rows.append((
                                    restaurant_guid,
                                    menu_guid,
                                    menu_name,
                                    menu_group_guid,
                                    menu_group_name,
                                    menu_item_guid,
                                    menu_item_name,
                                    modifier_group_guid,
                                    modifier_group_reference_id,
                                    get_any(mod_group_obj, ["name"]),
                                    stringify_value(modifier_group_position),
                                    source_system,
                                    source_endpoint,
                                    record_source,
                                    source_payload_hash,
                                    source_ingested_at_utc,
                                    json.dumps(
                                        {
                                            "menu": {
                                                "guid": menu_guid,
                                                "name": menu_name
                                            },
                                            "menuGroup": {
                                                "guid": menu_group_guid,
                                                "name": menu_group_name
                                            },
                                            "menuItem": item,
                                            "modifierGroup": mod_group_obj,
                                            "modifierGroupReferenceId": modifier_group_ref_id,
                                            "modifierGroupPosition": modifier_group_position
                                        },
                                        sort_keys=True
                                    )
                                ))

    print("Modifier reference diagnostics:")
    for restaurant_guid, group_count, option_count in diagnostic_counts:
        print(
            f"restaurant_guid={restaurant_guid} | "
            f"modifierGroupReferences={group_count} | "
            f"modifierOptionReferences={option_count}"
        )

    print(f"Staged modifier groups: {len(modifier_group_rows)}")
    print(f"Staged modifier options: {len(modifier_option_rows)}")
    print(f"Staged item→modifier group bridge rows: {len(item_modifier_group_bridge_rows)}")
    print(f"Staged modifier group→option bridge rows: {len(modifier_group_option_bridge_rows)}")

    return (
        modifier_group_rows,
        modifier_option_rows,
        item_modifier_group_bridge_rows,
        modifier_group_option_bridge_rows
    )

modifier_group_cols = [
    "restaurant_guid",
    "modifier_group_guid",
    "modifier_group_reference_id",
    "modifier_group_name",
    "pos_name",
    "min_selections",
    "max_selections",
    "required_flag",
    "multi_select_flag",
    "pricing_mode",
    "modifier_option_count",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

modifier_option_cols = [
    "restaurant_guid",
    "modifier_option_guid",
    "modifier_option_reference_id",
    "modifier_option_name",
    "pos_name",
    "price",
    "pricing_strategy",
    "sales_category_guid",
    "sku",
    "plu",
    "visibility",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

item_modifier_group_bridge_cols = [
    "restaurant_guid",
    "menu_guid",
    "menu_name",
    "menu_group_guid",
    "menu_group_name",
    "menu_item_guid",
    "menu_item_name",
    "modifier_group_guid",
    "modifier_group_reference_id",
    "modifier_group_name",
    "modifier_group_position",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

modifier_group_option_bridge_cols = [
    "restaurant_guid",
    "modifier_group_guid",
    "modifier_group_reference_id",
    "modifier_group_name",
    "modifier_option_guid",
    "modifier_option_reference_id",
    "modifier_option_name",
    "modifier_option_position",
    "source_system",
    "source_endpoint",
    "record_source",
    "source_payload_hash",
    "source_ingested_at_utc",
    "payload_record_json"
]

try:
    (
        modifier_group_rows,
        modifier_option_rows,
        item_modifier_group_bridge_rows,
        modifier_group_option_bridge_rows
    ) = extract_modifier_reference_tables_v2()

    modifier_group_df = spark.createDataFrame(
        modifier_group_rows,
        make_schema(modifier_group_cols)
    )

    modifier_option_df = spark.createDataFrame(
        modifier_option_rows,
        make_schema(modifier_option_cols)
    )

    item_modifier_group_bridge_df = spark.createDataFrame(
        item_modifier_group_bridge_rows,
        make_schema(item_modifier_group_bridge_cols)
    )

    modifier_group_option_bridge_df = spark.createDataFrame(
        modifier_group_option_bridge_rows,
        make_schema(modifier_group_option_bridge_cols)
    )

    merge_table(
        table_name="stg_dim_modifier_group",
        source_entity="brz_toast_menus",
        stage_df=modifier_group_df,
        key_cols=["restaurant_guid", "modifier_group_guid"],
        table_type_note="modifier group dimension"
    )

    merge_table(
        table_name="stg_dim_modifier_option",
        source_entity="brz_toast_menus",
        stage_df=modifier_option_df,
        key_cols=["restaurant_guid", "modifier_option_guid"],
        table_type_note="modifier option dimension"
    )

    merge_table(
        table_name="stg_bridge_menu_item_modifier_group",
        source_entity="brz_toast_menus",
        stage_df=item_modifier_group_bridge_df,
        key_cols=["restaurant_guid", "menu_item_guid", "modifier_group_guid"],
        table_type_note="menu item to modifier group bridge"
    )

    merge_table(
        table_name="stg_bridge_modifier_group_option",
        source_entity="brz_toast_menus",
        stage_df=modifier_group_option_bridge_df,
        key_cols=["restaurant_guid", "modifier_group_guid", "modifier_option_guid"],
        table_type_note="modifier group to modifier option bridge"
    )

    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_modifier_reference_dimension_merge_v2' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'SUCCEEDED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            NULL AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    print(f"✅ Modifier reference dimension V2 build completed successfully. run_id={run_id}")

except Exception as e:
    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())
    safe_error = str(e).replace("'", "''")[:4000]

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_CORE_DIMENSIONS' AS pipeline_name,
            'silver_modifier_reference_dimension_merge_v2' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            'FAILED' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            NULL AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            '{safe_error}' AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

    raise

# ----------------------------
# Validate modifier/reference tables
# ----------------------------

expected_modifier_tables = [
    "stg_dim_modifier_group",
    "stg_dim_modifier_option",
    "stg_bridge_menu_item_modifier_group",
    "stg_bridge_modifier_group_option"
]

existing_tables = [row.tableName for row in spark.sql("SHOW TABLES").collect()]
missing_tables = [t for t in expected_modifier_tables if t not in existing_tables]

if missing_tables:
    print("❌ Missing modifier/reference tables:")
    for t in missing_tables:
        print(t)
else:
    print("✅ All expected modifier reference tables exist.")

for t in expected_modifier_tables:
    if t in existing_tables:
        row_count = spark.table(t).count()
        print(f"{t}: {row_count:,} rows")
        display(spark.table(t).limit(10))
```

```python
expected_tables = [
    "stg_dim_restaurant",
    "stg_dim_revenue_center",
    "stg_dim_sales_category",
    "stg_dim_dining_option",
    "stg_dim_discount",
    "stg_dim_service_charge",
    "stg_dim_tax_rate",
    "stg_dim_alt_payment_type",
    "stg_dim_menu_group",
    "stg_dim_menu_item",
    "stg_dim_modifier_group",
    "stg_dim_modifier_option",
    "stg_bridge_menu_item_modifier_group",
    "stg_bridge_modifier_group_option"
]

existing_tables = [row.tableName for row in spark.sql("SHOW TABLES").collect()]
missing_tables = [t for t in expected_tables if t not in existing_tables]

if missing_tables:
    print("❌ Missing dimension/reference tables:")
    for t in missing_tables:
        print(t)
else:
    print("✅ All expected core dimension/reference tables exist.")

print("\nRow counts:")
for t in expected_tables:
    if t in existing_tables:
        print(f"{t}: {spark.table(t).count():,}")

display(
    spark.table("ctl_pipeline_run")
    .select(
        "pipeline_name",
        "pipeline_stage",
        "status",
        "started_at_utc",
        "ended_at_utc",
        "run_id",
        "error_message"
    )
    .where(F.col("pipeline_name") == "NB_BUILD_CORE_DIMENSIONS")
    .orderBy(F.col("started_at_utc").desc())
)
```

## Validation Notes

- Dimension tables created/merged: ☐
- Bridge tables created/merged: ☐
- `ctl_rowcount_audit` populated: ☐
- `ctl_hash_change_audit` populated: ☐
- `ctl_pipeline_run` populated: ☐