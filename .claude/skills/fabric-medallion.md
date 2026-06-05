---
name: fabric-medallion
description: >
  Use this skill when working on Microsoft Fabric Lakehouse projects using
  medallion architecture (Bronze, Silver, Gold), PySpark notebooks, Delta tables,
  Kimball dimensional modeling, or Power BI Direct Lake. Covers modeling
  conventions, pipeline patterns, SCD Type 2, accumulating snapshot MERGE,
  factless facts, and layer boundary rules.
---

# Fabric Medallion Architecture Skill

## Layer Boundary Rules

These are hard rules. Never cross them.

| Layer | Job | What does NOT belong here |
|---|---|---|
| Bronze | Exact copy of source + metadata columns only | No transformations, no business logic, no joins |
| Silver | Clean, conform, deduplicate, resolve identity | No surrogate keys, no star schema, not 3NF |
| Gold | Star schema — dims + facts only | No raw fields, no source system codes, no cleaning logic |

**Silver is not 3NF.** It is a wide, flat conforming layer optimised for Gold
loading. Do not normalise Silver into entity tables. One wide staging table
per subject area.

**Bronze is an audit layer.** If Silver or Gold breaks, Bronze is the replay
source. Never modify Bronze after ingestion.

---

## Bronze Pattern

Every Bronze table gets exactly three metadata columns added — nothing else:

```python
from pyspark.sql import functions as F

df = spark.read.option("header", True).csv(source_path)

df = df.withColumn("_source_file", F.lit(filename)) \
       .withColumn("_source_system", F.lit(source_system)) \
       .withColumn("_ingested_at", F.current_timestamp())

df.write.format("delta") \
   .mode("overwrite") \
   .partitionBy("_source_system") \
   .saveAsTable(f"bronze.{table_name}")
```

Partition all Bronze tables by `_source_system`. Never partition Bronze by date
unless the source delivers date-partitioned files — let the data drive it.

---

## Silver Pattern

### Identity resolution
When the same person appears across multiple source systems, resolve on
a stable unique key — typically `email`. Never assume source system IDs match.

```python
# Resolve student identity across HubSpot + Dynamics + Paradigm
silver_student = (
    paradigm_students
    .join(hubspot_leads, on="student_email", how="left")
    .join(dynamics_contacts, on="student_email", how="left")
    .select(
        F.coalesce("paradigm_student_id", F.expr("uuid()")).alias("student_id"),
        "hubspot_lead_id",
        "dynamics_contact_id",
        "student_email",
        "student_first_name",
        "student_last_name",
        "visa_code",
        "is_international",
        F.current_timestamp().alias("_loaded_at")
    )
)
```

### Deduplication
Always deduplicate before writing Silver. Use `ROW_NUMBER()` on the natural key,
ordered by the most reliable timestamp descending.

```python
from pyspark.sql.window import Window

w = Window.partitionBy("natural_key").orderBy(F.col("source_timestamp").desc())

df_deduped = df.withColumn("rn", F.row_number().over(w)) \
               .filter(F.col("rn") == 1) \
               .drop("rn")
```

### SCD Type 2
Only implement SCD Type 2 on attributes that affect historical reporting accuracy.
For this project: `visa_code` and `is_international` on `silver.student`.

**SCD Type 2 admin columns — always the same three:**
```
_valid_from     DATE        -- date this version became active
_valid_to       DATE        -- 9999-12-31 for current record
_is_current     BOOLEAN     -- True for the active version only
```

**SCD Type 2 MERGE pattern:**
```python
from delta.tables import DeltaTable

existing = DeltaTable.forName(spark, "silver.student")

# Step 1: expire changed rows
existing.alias("target").merge(
    incoming.alias("source"),
    "target.student_id = source.student_id AND target._is_current = true"
).whenMatchedUpdate(
    condition="""
        target.visa_code != source.visa_code OR
        target.is_international != source.is_international
    """,
    set={
        "_valid_to": F.date_sub(F.current_date(), 1),
        "_is_current": F.lit(False)
    }
).execute()

# Step 2: insert new versions for changed rows + brand new rows
new_versions = incoming.join(
    existing.toDF().filter("_is_current = true"),
    on="student_id", how="left_anti"  # rows not already current
).withColumn("_valid_from", F.current_date()) \
 .withColumn("_valid_to", F.lit("9999-12-31").cast("date")) \
 .withColumn("_is_current", F.lit(True))

new_versions.write.format("delta").mode("append").saveAsTable("silver.student")
```

---

## Gold — Dimension Conventions

### Surrogate keys
Always integers. Generate with `row_number()` over a deterministic order —
never `monotonically_increasing_id()` (non-deterministic across runs).

```python
from pyspark.sql.window import Window

w = Window.orderBy("natural_key")
df = df.withColumn("table_sk", F.row_number().over(w))
```

### Natural keys
Always retain as alternate key column alongside the surrogate key.
Naming: `_sk` suffix for surrogate, `_id` suffix for natural key.

```python
# correct
student_sk   INT    -- surrogate (PK)
student_id   STRING -- natural key (alternate key)

# wrong — never use just one or the other
```

### dim_date
- PK is integer `YYYYMMDD` — e.g. `20240115`
- **Always seed surrogate key 0 as the TBD row before any fact notebook runs**
- TBD row represents "milestone not yet reached" in accumulating snapshot facts

```python
tbd_row = spark.createDataFrame([{
    "date_key": 0,
    "full_date": None,
    "calendar_year": None,
    "month_name": "TBD",
    "semester_code": None,
    "is_teaching_week": False
}])
tbd_row.write.format("delta").mode("append").saveAsTable("gold.dim_date")
```

### SCD Type 2 dimensions in Gold
When joining fact tables to SCD Type 2 dimensions, always filter to the
version active at the time of the fact event — not just `_is_current = true`.

```python
# correct: point-in-time join
fact.join(
    dim_student,
    (fact.student_id == dim_student.student_id) &
    (fact.event_date >= dim_student._valid_from) &
    (fact.event_date < dim_student._valid_to)
)

# wrong: current-only join loses historical accuracy
fact.join(dim_student, on="student_id")
    .filter(dim_student._is_current == True)
```

---

## Gold — Fact Table Conventions

### Three fact types — know which one you are building

| Type | Grain | Insert/Update | Null measures? |
|---|---|---|---|
| Transaction | One row per event | Insert only, never update | No |
| Accumulating Snapshot | One row per pipeline instance | Insert + UPDATE via MERGE | Yes — NULL until milestone reached |
| Factless | One row per scheduled slot or activity | Insert only | No measures at all |

### Degenerate dimensions
Single-attribute values with no hierarchy (e.g. source system IDs, document numbers)
stay in the fact table as VARCHAR columns — do not create a dimension table for them.

### Null FK handling
Fact table foreign keys must **never be NULL**.
- Unknown dimension member → surrogate key `-1` (Unknown row in dim table)
- Unresolved date milestone → surrogate key `0` (TBD row in dim_date)

Always seed these special rows in dimension tables before loading facts.

---

## Accumulating Snapshot — Full Pattern

This is the most complex fact type. Follow this pattern exactly.

**The row lifecycle:**
1. Row inserted when first milestone fires (e.g. lead captured)
2. All future milestone date keys default to `0` (TBD)
3. All lag facts default to `NULL`
4. Each subsequent milestone triggers a MERGE — updates that milestone's columns only
5. Lag fact calculated and stored permanently when both endpoint dates are non-zero
6. Row is never deleted — terminal state is the final milestone flag = 1

**MERGE pattern:**
```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "gold.fact_application_pipeline")

# incoming_df contains only rows where a new milestone has fired
# it carries the updated date key and newly calculable lag

target.alias("t").merge(
    incoming_df.alias("s"),
    "t.application_id = s.application_id"
).whenMatchedUpdate(set={
    # only update columns that have changed — leave others untouched
    "offer_date_key":             "CASE WHEN s.offer_date_key != 0 THEN s.offer_date_key ELSE t.offer_date_key END",
    "application_to_offer_days":  "CASE WHEN s.offer_date_key != 0 AND t.offer_date_key = 0 THEN s.application_to_offer_days ELSE t.application_to_offer_days END",
    "is_offered":                 "CASE WHEN s.offer_date_key != 0 THEN 1 ELSE t.is_offered END",
    "_last_updated_at":           "current_timestamp()"
}).whenNotMatchedInsert(values={
    "pipeline_sk":                "s.pipeline_sk",
    "application_id":             "s.application_id",
    "student_sk":                 "s.student_sk",
    "lead_date_key":              "s.lead_date_key",
    "visit_date_key":             "0",
    "application_date_key":       "0",
    "offer_date_key":             "0",
    "decision_date_key":          "0",
    "enrolment_date_key":         "0",
    "lead_to_application_days":   "null",
    "application_to_offer_days":  "null",
    "offer_to_decision_days":     "null",
    "decision_to_enrolment_days": "null",
    "lead_to_enrolment_days":     "null",
    "is_lead":                    "1",
    "is_visited":                 "0",
    "is_applied":                 "0",
    "is_offered":                 "0",
    "is_accepted":                "0",
    "is_enrolled":                "0",
    "_loaded_at":                 "current_timestamp()",
    "_last_updated_at":           "current_timestamp()"
}).execute()
```

**Lag calculation rule:**
Calculate and store a lag column permanently the first time both endpoint
dates are non-zero. Never recalculate a lag that is already stored.
NULL = not yet calculable (milestone has not occurred). This is intentional.

```python
# Calculate lag only when both dates are now known
df = df.withColumn(
    "application_to_offer_days",
    F.when(
        (F.col("application_date_key") != 0) & (F.col("offer_date_key") != 0),
        F.datediff(
            F.to_date(F.col("offer_date_key").cast("string"), "yyyyMMdd"),
            F.to_date(F.col("application_date_key").cast("string"), "yyyyMMdd")
        )
    ).otherwise(F.lit(None))
)
```

---

## Factless Fact — Coverage + Activity Pattern

Two tables are required to detect absence. One table alone is not enough.

```
fact_attendance_coverage  — what SHOULD happen (every scheduled slot)
fact_attendance_actual    — what DID happen (every attended session)
```

At-risk query: students in coverage with no matching rows in actual.

```python
# PySpark version
coverage = spark.table("gold.fact_attendance_coverage")
actual = spark.table("gold.fact_attendance_actual")

at_risk = coverage.join(
    actual,
    on=["student_sk", "course_sk", "intake_sk"],
    how="left_anti"  # rows in coverage with NO match in actual
).select("student_sk", "course_sk", "intake_sk").distinct()
```

Never collapse coverage and actual into one table with an `attended` flag.
That design cannot answer "enrolled but never attended" without a self-join.

---

## Notebook Execution Order

Always run in this sequence. Never skip steps.

```
01_bronze_ingest        → writes bronze.* Delta tables
02_silver_transform     → reads bronze.*, writes silver.* Delta tables
03_gold_dimensions      → reads silver.*, writes gold.dim_* Delta tables
                          seeds dim_date TBD row (key = 0) before exiting
04_gold_fact_pipeline   → reads silver.* + gold.dim_*, writes gold.fact_application_pipeline
05_gold_fact_enrolment  → reads silver.* + gold.dim_*, writes gold.fact_enrolment
06_gold_fact_attendance → reads silver.* + gold.dim_*, writes gold.fact_attendance_*
```

Notebooks 04, 05, 06 are independent of each other but all depend on 03.
Never run a fact notebook before 03 completes — surrogate keys will not exist.

---

## Naming Conventions

| Pattern | Meaning | Example |
|---|---|---|
| `_sk` suffix | Surrogate key (integer PK or FK) | `student_sk` |
| `_id` suffix | Natural business key | `student_id` |
| `_key` suffix | Date dimension FK (integer YYYYMMDD) | `offer_date_key` |
| `_days` suffix | Lag fact (integer) | `application_to_offer_days` |
| `is_` prefix | Milestone flag or boolean (0/1 integer) | `is_enrolled` |
| `bronze.` prefix | Bronze schema table | `bronze.hubspot_leads` |
| `silver.` prefix | Silver schema table | `silver.student` |
| `gold.` prefix | Gold schema table | `gold.dim_student` |
| `dim_` prefix | Dimension table | `dim_course` |
| `fact_` prefix | Fact table | `fact_enrolment` |
| `_valid_from/to` | SCD Type 2 admin columns | `_valid_from` |
| `_is_current` | SCD Type 2 current record flag | `_is_current` |
| `_loaded_at` | Pipeline audit timestamp | `_loaded_at` |
| `_last_updated_at` | Last MERGE update timestamp | `_last_updated_at` |

---

## Power BI Direct Lake

- Connect Power BI to the Fabric Lakehouse Gold schema directly
- No import, no scheduled refresh — Gold Delta tables are immediately visible
- For role-playing date dimensions (multiple date FKs in one fact table):
  create one inactive relationship per date FK in the semantic model
- Activate inactive relationships in DAX with `USERELATIONSHIP()`

```dax
-- Example: avg days using the offer date relationship
Avg App to Offer Days =
CALCULATE(
    AVERAGE(fact_application_pipeline[application_to_offer_days]),
    USERELATIONSHIP(fact_application_pipeline[offer_date_key], dim_date[date_key])
)
```

- Semi-additive measures (e.g. headcount) use `AVERAGEX` over time, not `SUM`
- Additive milestone flags use `SUM` — `SUM(is_enrolled)` = count of enrolled students
- Lag facts use `AVERAGE` — `AVERAGE(lead_to_enrolment_days)` = avg pipeline duration
- Never `SUM` a lag fact — it is meaningless as a total
