# NB_BUILD_RECON_MART

Artifact Type: Notebook
Build Method: Notebook
Dependencies: Silver transactional/support tables; control/audit tables
Grain: One row per restaurant_guid + business_date
Last Updated: April 27, 2026
Layer: Gold
Notes: Already built mart_reconciliation_daily successfully based on uploaded notebook output. Use discount_amount, not discount_value.
Primary / Merge Key: restaurant_guid + business_date
Purpose: Build the first Gold reconciliation mart by combining sales-side, discount, tax, service charge, and payment-side daily metrics.
Source Objects: stg_toast_check; stg_toast_selection; stg_toast_payment; stg_toast_check_service_charge; stg_toast_check_discount; stg_toast_selection_discount; stg_toast_selection_tax; ctl_rowcount_audit; ctl_pipeline_run
Status: Documented
Target Object: mart_reconciliation_daily

## Notebook Purpose

Builds `mart_reconciliation_daily` by aggregating check totals, selection totals, service charges, check/selection discounts, selection taxes, and payments by `restaurant_guid + business_date`.

## Notebook Code Source

Uploaded file: `NB_BUILD_RECON_MART.md.txt`

## Important Logic Notes

- Sales side uses `stg_toast_check`, `stg_toast_selection`, discounts, taxes, and service charges.
- Tender side uses `stg_toast_payment` and `paid_business_date`.
- Discounts use `discount_amount`, not `discount_value`.
- Tips are separate from service charges.
- The target merge key is `restaurant_guid + business_date`.

## Paste Notebook Code Here

```
from pyspark.sql import functions as F
from pyspark.sql import types as T
from delta.tables import DeltaTable
from datetime import datetime
import uuid

run_id = str(uuid.uuid4())
started_at_utc = datetime.utcnow()

target_table = "mart_reconciliation_daily"

required_tables = [
    "stg_toast_check",
    "stg_toast_selection",
    "stg_toast_payment",
    "stg_toast_check_service_charge",
    "stg_toast_check_discount",
    "stg_toast_selection_discount",
    "stg_toast_selection_tax",
    "stg_dim_alt_payment_type"
]

# ------------------------------------------------------------
# Helpers
# ------------------------------------------------------------

def table_exists(table_name):
    return spark.catalog.tableExists(table_name)

def require_tables(table_names):
    missing = [t for t in table_names if not table_exists(t)]
    if missing:
        raise Exception("Missing required tables: " + ", ".join(missing))

def require_columns(table_name, required_columns):
    existing = set(spark.table(table_name).columns)
    missing = [c for c in required_columns if c not in existing]
    if missing:
        raise Exception(f"{table_name} is missing required columns: {', '.join(missing)}")

def money_expr(col_name):
    return F.coalesce(F.col(col_name).cast("decimal(18,4)"), F.lit(0).cast("decimal(18,4)"))

def number_expr(col_name):
    return F.coalesce(F.col(col_name).cast("decimal(18,4)"), F.lit(0).cast("decimal(18,4)"))

def business_date_expr(col_name):
    return F.coalesce(
        F.to_date(F.col(col_name).cast("string"), "yyyy-MM-dd"),
        F.to_date(F.col(col_name).cast("string"), "yyyyMMdd"),
        F.to_date(F.col(col_name))
    )

def normalize_business_date(df, source_col="business_date"):
    return df.withColumn("business_date", business_date_expr(source_col))

def append_rowcount_audit(table_name, source_entity, row_count, audit_status, notes):
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
        "gold",
        table_name,
        "toast",
        source_entity,
        None,
        None,
        row_count,
        None,
        None,
        None,
        audit_status,
        datetime.utcnow(),
        notes
    )]

    spark.createDataFrame(audit_row, audit_schema).write.format("delta").mode("append").saveAsTable("ctl_rowcount_audit")

def log_pipeline_run(status, rows_written=None, error_message=None):
    ended_at_utc = datetime.utcnow()
    duration_seconds = int((ended_at_utc - started_at_utc).total_seconds())
    safe_error = error_message.replace("'", "''")[:4000] if error_message else None

    spark.sql(f"""
        INSERT INTO ctl_pipeline_run
        SELECT
            '{run_id}' AS run_id,
            'NB_BUILD_RECON_MART' AS pipeline_name,
            'gold_reconciliation_mart_build' AS pipeline_stage,
            'manual_build' AS run_type,
            'manual' AS trigger_type,
            'toast' AS source_system,
            NULL AS restaurant_guid,
            '{status}' AS status,
            TIMESTAMP('{started_at_utc}') AS started_at_utc,
            TIMESTAMP('{ended_at_utc}') AS ended_at_utc,
            {duration_seconds} AS duration_seconds,
            NULL AS rows_read,
            {rows_written if rows_written is not None else "NULL"} AS rows_written,
            NULL AS rows_inserted,
            NULL AS rows_updated,
            NULL AS rows_deleted,
            {f"'{safe_error}'" if safe_error else "NULL"} AS error_message,
            current_user() AS created_by,
            current_timestamp() AS created_at_utc,
            current_timestamp() AS updated_at_utc
    """)

# ------------------------------------------------------------
# Build mart
# ------------------------------------------------------------

try:
    require_tables(required_tables)

    require_columns("stg_toast_check", [
        "restaurant_guid",
        "business_date",
        "check_guid",
        "order_guid",
        "amount_subtotal",
        "amount_tax",
        "amount_total"
    ])

    require_columns("stg_toast_selection", [
        "restaurant_guid",
        "business_date",
        "selection_guid",
        "quantity",
        "pre_discount_price",
        "price",
        "tax_amount"
    ])

    require_columns("stg_toast_payment", [
        "restaurant_guid",
        "payment_guid",
        "paid_business_date",
        "amount",
        "tip_amount",
        "payment_type",
        "other_payment_reference"
    ])

    require_columns("stg_dim_alt_payment_type", [
        "alt_payment_type_guid",
        "alt_payment_type_name"
    ])

    require_columns("stg_toast_check_service_charge", [
        "restaurant_guid",
        "business_date",
        "amount"
    ])

    require_columns("stg_toast_check_discount", [
        "restaurant_guid",
        "business_date",
        "discount_amount"
    ])

    require_columns("stg_toast_selection_discount", [
        "restaurant_guid",
        "business_date",
        "discount_amount"
    ])

    require_columns("stg_toast_selection_tax", [
        "restaurant_guid",
        "business_date",
        "tax_amount"
    ])

    # ----------------------------
    # Sales side — checks
    # ----------------------------

    check_daily = (
        normalize_business_date(spark.table("stg_toast_check"))
        .where(
            (F.col("business_date").isNotNull()) &
            (F.col("payment_status") == "CLOSED")
        )
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.countDistinct("check_guid").alias("check_count"),
            F.countDistinct("order_guid").alias("order_count_from_checks"),
            F.sum(money_expr("amount_subtotal")).alias("check_subtotal_amount"),
            F.sum(money_expr("amount_tax")).alias("check_tax_amount"),
            F.sum(money_expr("amount_total")).alias("check_total_amount")
        )
    )

    # ----------------------------
    # Sales side — selections/items
    # ----------------------------

    selection_daily = (
        normalize_business_date(spark.table("stg_toast_selection"))
        .where(F.col("business_date").isNotNull())
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.countDistinct("selection_guid").alias("selection_count"),
            F.sum(number_expr("quantity")).alias("selection_quantity"),
            F.sum(money_expr("pre_discount_price")).alias("selection_pre_discount_amount"),
            F.sum(money_expr("price")).alias("selection_price_amount"),
            F.sum(money_expr("tax_amount")).alias("selection_tax_amount_from_selection")
        )
    )

    # ----------------------------
    # Service charges / auto gratuity
    # ----------------------------

    closed_check_keys = (
        normalize_business_date(spark.table("stg_toast_check"))
        .where(
            (F.col("business_date").isNotNull()) &
            (F.col("payment_status") == "CLOSED")
        )
        .select("restaurant_guid", "business_date", "check_guid")
        .distinct()
    )

    service_charge_daily = (
        normalize_business_date(spark.table("stg_toast_check_service_charge"))
        .join(
            closed_check_keys,
            ["restaurant_guid", "business_date", "check_guid"],
            "inner"
        )
        .where(F.col("business_date").isNotNull())
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.count("*").alias("service_charge_line_count"),
            F.sum(money_expr("amount")).alias("service_charge_amount")
        )
    )

    # ----------------------------
    # Check-level discounts
    # Use discount_amount, not discount_value.
    # discount_value is the configured percent/value; discount_amount is the actual applied dollars.
    # ----------------------------

    check_discount_daily = (
        normalize_business_date(spark.table("stg_toast_check_discount"))
        .join(
            closed_check_keys,
            ["restaurant_guid", "business_date", "check_guid"],
            "inner"
        )
        .where(F.col("business_date").isNotNull())
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.count("*").alias("check_discount_line_count"),
            F.sum(money_expr("discount_amount")).alias("check_discount_amount")
        )
    )

    # ----------------------------
    # Selection-level discounts
    # Use discount_amount, not discount_value.
    # ----------------------------

    selection_discount_daily = (
        normalize_business_date(spark.table("stg_toast_selection_discount"))
        .where(F.col("business_date").isNotNull())
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.count("*").alias("selection_discount_line_count"),
            F.sum(money_expr("discount_amount")).alias("selection_discount_amount")
        )
    )

    # ----------------------------
    # Selection tax bridge
    # ----------------------------

    selection_tax_daily = (
        normalize_business_date(spark.table("stg_toast_selection_tax"))
        .where(F.col("business_date").isNotNull())
        .groupBy("restaurant_guid", "business_date")
        .agg(
            F.count("*").alias("selection_tax_line_count"),
            F.sum(money_expr("tax_amount")).alias("selection_tax_bridge_amount")
        )
    )

    # ----------------------------
    # Tender side — payments
    # Use paid_business_date as tender business date.
    # ----------------------------

    # -------------------------------------------------------------------
    # payment_daily
    # Purpose:
    #   Aggregate Toast tender-side payment metrics by restaurant_guid + business_date.
    #   Classify OTHER payment references before summing payment_amount.
    #
    # Confirmed OTHER classifications from stg_dim_alt_payment_type:
    #   3de10cc8-96d7-4eb6-8ac1-d411736d54ed = Deposit Redeem
    #   d45433c4-bef2-476c-bd30-23f7300616e5 = PV House Account
    #
    # Reconciliation treatment:
    #   - PV House Account is a real tender and belongs in payment_amount.
    #   - Deposit Redeem is not normal payment subtotal and must not inflate payment_amount.
    #   - Unclassified OTHER is excluded from payment_amount and kept visible for QA.
    # -------------------------------------------------------------------

    DEPOSIT_REDEEM_ALT_PAYMENT_GUIDS = [
        "3de10cc8-96d7-4eb6-8ac1-d411736d54ed"
    ]

    PAYMENT_SUBTOTAL_ALT_PAYMENT_GUIDS = [
        "d45433c4-bef2-476c-bd30-23f7300616e5"
    ]

    payment_enriched = (
        spark.table("stg_toast_payment").alias("p")
        .join(
            spark.table("stg_dim_alt_payment_type").alias("apt"),
            F.col("p.other_payment_reference") == F.col("apt.alt_payment_type_guid"),
            "left"
        )
        .withColumn(
            "payment_recon_bucket",
            F.when(F.col("p.payment_type") != "OTHER", F.lit("PAYMENT_SUBTOTAL"))
            .when(F.col("p.other_payment_reference").isin(PAYMENT_SUBTOTAL_ALT_PAYMENT_GUIDS), F.lit("PAYMENT_SUBTOTAL"))
            .when(F.col("p.other_payment_reference").isin(DEPOSIT_REDEEM_ALT_PAYMENT_GUIDS), F.lit("DEPOSIT_REDEEM"))
            .otherwise(F.lit("UNCLASSIFIED_OTHER"))
        )
    )

    payment_daily = (
        payment_enriched
        .groupBy(
            F.col("p.restaurant_guid").alias("restaurant_guid"),
            F.col("p.paid_business_date").alias("business_date")
        )
        .agg(
            F.countDistinct(
                F.when(
                    F.col("payment_recon_bucket") == "PAYMENT_SUBTOTAL",
                    F.col("p.payment_guid")
                )
            ).alias("payment_count"),

            F.round(
                F.sum(
                    F.when(
                        F.col("payment_recon_bucket") == "PAYMENT_SUBTOTAL",
                        F.coalesce(F.col("p.amount"), F.lit(0))
                    ).otherwise(F.lit(0))
                ),
                2
            ).alias("payment_amount"),

            F.round(
                F.sum(
                    F.when(
                        F.col("payment_recon_bucket") == "PAYMENT_SUBTOTAL",
                        F.coalesce(F.col("p.tip_amount"), F.lit(0))
                    ).otherwise(F.lit(0))
                ),
                2
            ).alias("payment_tip_amount"),

            F.round(
                F.sum(
                    F.when(
                        F.col("payment_recon_bucket") == "DEPOSIT_REDEEM",
                        F.coalesce(F.col("p.amount"), F.lit(0))
                    ).otherwise(F.lit(0))
                ),
                2
            ).alias("deposit_redeem_amount"),

            F.round(
                F.sum(
                    F.when(
                        F.col("payment_recon_bucket") == "UNCLASSIFIED_OTHER",
                        F.coalesce(F.col("p.amount"), F.lit(0))
                    ).otherwise(F.lit(0))
                ),
                2
            ).alias("unclassified_other_payment_amount")
        )
    )

    # ----------------------------
    # Build daily key set
    # ----------------------------

    base_keys = (
        check_daily.select("restaurant_guid", "business_date")
        .unionByName(selection_daily.select("restaurant_guid", "business_date"))
        .unionByName(service_charge_daily.select("restaurant_guid", "business_date"))
        .unionByName(check_discount_daily.select("restaurant_guid", "business_date"))
        .unionByName(selection_discount_daily.select("restaurant_guid", "business_date"))
        .unionByName(selection_tax_daily.select("restaurant_guid", "business_date"))
        .unionByName(payment_daily.select("restaurant_guid", "business_date"))
        .distinct()
    )

    mart_df = (
        base_keys
        .join(check_daily, ["restaurant_guid", "business_date"], "left")
        .join(selection_daily, ["restaurant_guid", "business_date"], "left")
        .join(service_charge_daily, ["restaurant_guid", "business_date"], "left")
        .join(check_discount_daily, ["restaurant_guid", "business_date"], "left")
        .join(selection_discount_daily, ["restaurant_guid", "business_date"], "left")
        .join(selection_tax_daily, ["restaurant_guid", "business_date"], "left")
        .join(payment_daily, ["restaurant_guid", "business_date"], "left")
    )

    numeric_columns = [c for c in mart_df.columns if c not in ["restaurant_guid", "business_date"]]

    for c in numeric_columns:
        mart_df = mart_df.withColumn(c, F.coalesce(F.col(c), F.lit(0)))

    mart_df = (
        mart_df
        .withColumn(
            "total_discount_amount",
            F.col("check_discount_amount") + F.col("selection_discount_amount")
        )
        .withColumn(
            "sales_including_service_amount",
            F.col("check_total_amount") + F.col("service_charge_amount")
        )
        .withColumn(
            "payments_excluding_tips_amount",
            F.col("payment_amount") - F.col("payment_tip_amount")
        )
        .withColumn(
            "sales_including_service_vs_payments_ex_tips_variance",
            F.col("sales_including_service_amount") - F.col("payments_excluding_tips_amount")
        )
        .withColumn(
            "sales_vs_tender_variance",
            F.col("check_total_amount") - F.col("payment_amount")
        )
        .withColumn(
            "sales_vs_tender_variance_pct",
            F.when(
                F.col("check_total_amount") != 0,
                (F.col("sales_vs_tender_variance") / F.col("check_total_amount")).cast("decimal(18,6)")
            ).otherwise(F.lit(None).cast("decimal(18,6)"))
        )
        .withColumn(
            "row_hash",
            F.sha2(
                F.concat_ws(
                    "||",
                    *[
                        F.coalesce(F.col(c).cast("string"), F.lit(""))
                        for c in [
                            "restaurant_guid",
                            "business_date",
                            "check_count",
                            "order_count_from_checks",
                            "selection_count",
                            "payment_count",
                            "check_subtotal_amount",
                            "check_tax_amount",
                            "check_total_amount",
                            "selection_price_amount",
                            "service_charge_amount",
                            "total_discount_amount",
                            "sales_including_service_amount",
                            "payments_excluding_tips_amount",
                            "sales_including_service_vs_payments_ex_tips_variance",
                            "selection_tax_bridge_amount",
                            "payment_amount",
                            "payment_tip_amount",
                            "sales_vs_tender_variance"
                        ]
                    ]
                ),
                256
            )
        )
        .withColumn("mart_grain", F.lit("restaurant_guid_business_date"))
        .withColumn("source_system", F.lit("toast"))
        .withColumn("run_id", F.lit(run_id))
        .withColumn("created_at_utc", F.current_timestamp())
        .withColumn("updated_at_utc", F.current_timestamp())
    )

    # ----------------------------
    # Merge into target Delta table
    # ----------------------------

    if not table_exists(target_table):
        (
            mart_df
            .write
            .format("delta")
            .mode("overwrite")
            .saveAsTable(target_table)
        )
    else:
        delta_table = DeltaTable.forName(spark, target_table)

        (
            delta_table.alias("t")
            .merge(
                mart_df.alias("s"),
                "t.restaurant_guid = s.restaurant_guid AND t.business_date = s.business_date"
            )
            .whenMatchedUpdateAll()
            .whenNotMatchedInsertAll()
            .execute()
        )

    mart_row_count = mart_df.count()

    append_rowcount_audit(
        table_name=target_table,
        source_entity="toast_silver_reconciliation_inputs",
        row_count=mart_row_count,
        audit_status="PASS",
        notes="mart_reconciliation_daily merge/upsert completed."
    )

    log_pipeline_run(
        status="SUCCEEDED",
        rows_written=mart_row_count
    )

    print(f"✅ {target_table} built successfully. Rows written/staged: {mart_row_count}. run_id={run_id}")

except Exception as e:
    error_message = str(e)

    try:
        log_pipeline_run(
            status="FAILED",
            rows_written=None,
            error_message=error_message
        )
    except Exception as logging_error:
        print(f"⚠️ Failed to log pipeline error: {logging_error}")

    raise This page should no longer be treated as an empty code stub.
```

## Validation Notes

- Mart table exists: ☐
- Row count checked: ☐
- Sales-side totals reviewed: ☐
- Tender-side totals reviewed: ☐
- Variance logic reviewed: ☐