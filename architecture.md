# architecture.md — edu-analytics-fabric

## Problem Statement

A higher education provider runs four operational systems:
- **HubSpot** — lead capture and marketing CRM
- **Dynamics 365** — admissions CRM (applications, offers)
- **Paradigm SIS** — student information system (enrolments, academic records)
- **Canvas LMS** — learning management (attendance, assessments)

Each system is write-optimised and answers only its own questions. No single
system can answer: *"Show me lead-to-enrolment conversion by course and campus,
with revenue, for this intake versus last intake."*

This project builds that single place — a dimensional model on Microsoft Fabric
Lakehouse, queried by Power BI via Direct Lake.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  SOURCE SYSTEMS (CSV files — static demo dataset)                    │
│                                                                      │
│  hubspot_leads.csv   dynamics_applications.csv                       │
│  paradigm_enrolments.csv   canvas_attendance.csv                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Manual upload to Lakehouse Files/
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BRONZE LAYER  (schema: bronze)                                      │
│  Notebook: 01_bronze_ingest.ipynb                                    │
│                                                                      │
│  Raw Delta tables — exact copy of source                             │
│  + _source_file, _source_system, _ingested_at metadata              │
│  No transformations. No business logic.                              │
│                                                                      │
│  bronze.hubspot_leads                                                │
│  bronze.dynamics_applications                                        │
│  bronze.paradigm_enrolments                                          │
│  bronze.canvas_attendance                                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ PySpark read → transform → write
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SILVER LAYER  (schema: silver)                                      │
│  Notebook: 02_silver_transform.ipynb                                 │
│                                                                      │
│  Cleaned, conformed, deduplicated staging tables                     │
│  Wide and flat — NOT 3NF                                             │
│  SCD Type 2 on silver_student (visa_code, is_international)          │
│                                                                      │
│  silver.student        silver.application                            │
│  silver.enrolment      silver.attendance                             │
│  silver.course         silver.campus                                 │
│  silver.intake                                                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ PySpark read → model → write
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GOLD LAYER  (schema: gold)                                          │
│  Notebooks: 03 → 04 → 05 → 06                                        │
│                                                                      │
│  Star schema — Kimball dimensional model                             │
│  Conformed dimensions + three fact tables                            │
│                                                                      │
│  DIMENSIONS              FACTS                                       │
│  gold.dim_date           gold.fact_application_pipeline              │
│  gold.dim_student        gold.fact_enrolment                         │
│  gold.dim_course         gold.fact_attendance_coverage               │
│  gold.dim_campus         gold.fact_attendance_actual                 │
│  gold.dim_intake                                                     │
│  gold.dim_lead_source                                                │
│  gold.dim_status                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ Direct Lake (no data copy, no refresh)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  POWER BI — Direct Lake mode                                         │
│  Reads Gold Delta tables directly from OneLake                      │
│  No import. No scheduled refresh.                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Source CSV Schemas

### hubspot_leads.csv
```
lead_id             VARCHAR   -- natural key from HubSpot
student_first_name  VARCHAR
student_last_name   VARCHAR
student_email       VARCHAR
date_of_birth       DATE
visa_type           VARCHAR   -- 'Domestic', 'International - Student Visa'
lead_source         VARCHAR   -- 'Agent', 'Direct', 'Social Media', 'Web Referral'
course_interest     VARCHAR   -- course_code e.g. BMUS301
campus_preference   VARCHAR   -- campus_code e.g. SYD, MEL
lead_date           DATE      -- when lead was captured
visit_date          DATE      -- campus visit date (nullable)
```

### dynamics_applications.csv
```
application_id      VARCHAR   -- natural key from Dynamics
lead_id             VARCHAR   -- FK back to HubSpot lead
student_email       VARCHAR   -- used to resolve student identity
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR   -- e.g. 2024-S1
application_date    DATE
offer_date          DATE      -- nullable
offer_type          VARCHAR   -- 'Conditional', 'Unconditional', nullable
decision_date       DATE      -- nullable (date student accepted or rejected)
decision_outcome    VARCHAR   -- 'Accepted', 'Rejected', nullable
```

### paradigm_enrolments.csv
```
enrolment_id        VARCHAR   -- natural key from Paradigm
student_email       VARCHAR   -- used to resolve student identity
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR
enrolment_date      DATE
enrolment_status    VARCHAR   -- 'Active', 'Withdrawn', 'Completed'
withdrawal_date     DATE      -- nullable
completion_date     DATE      -- nullable
enrolment_fee       DECIMAL
scholarship_amount  DECIMAL
credit_points       INT
```

### canvas_attendance.csv
```
session_id          VARCHAR   -- unique class session identifier
student_email       VARCHAR
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR
session_date        DATE
session_type        VARCHAR   -- 'Lecture', 'Tutorial', 'Workshop'
attended            INT       -- 1 = attended, 0 = absent (for coverage table)
```

---

## Bronze Layer

**Notebook:** `01_bronze_ingest.ipynb`

**Pattern:** Read CSV from Lakehouse Files section → add metadata → write Delta.

**Metadata columns added to every bronze table:**
```
_source_file      STRING   -- original filename e.g. 'hubspot_leads.csv'
_source_system    STRING   -- 'HUBSPOT', 'DYNAMICS', 'PARADIGM', 'CANVAS'
_ingested_at      TIMESTAMP -- UTC timestamp of ingestion run
```

**Partition strategy:**
- All bronze tables partitioned by `_source_system`
- Ensures downstream reads can prune to one source without scanning others

**Design principle:** Bronze is an audit layer. If something breaks in Silver
or Gold, Bronze is the source of truth to replay from. Never modify Bronze
after ingestion.

---

## Silver Layer

**Notebook:** `02_silver_transform.ipynb`

**Design principle:** Silver is a conforming layer, not an OLTP layer.
Wide, flat staging tables optimised for Gold loading — not 3NF.

### silver.student
Unified student record resolved across HubSpot (lead), Dynamics (applicant),
and Paradigm (enrolled student). Identity resolution key: `student_email`.

```
student_id          VARCHAR   -- natural key (from Paradigm when available, else generated)
hubspot_lead_id     VARCHAR   -- nullable (not all students have HubSpot record)
dynamics_contact_id VARCHAR   -- nullable
student_first_name  VARCHAR
student_last_name   VARCHAR
student_email       VARCHAR   -- identity resolution key, unique per current record
date_of_birth       DATE
visa_code           VARCHAR   -- SCD Type 2 tracked
is_international    BOOLEAN   -- SCD Type 2 tracked
lead_source         VARCHAR
-- SCD Type 2 admin columns
_valid_from         DATE
_valid_to           DATE      -- 9999-12-31 for current record
_is_current         BOOLEAN
_source_system      VARCHAR
_loaded_at          TIMESTAMP
```

**SCD Type 2 trigger attributes:** `visa_code`, `is_international`

When either attribute changes (e.g. student moves from Student Visa to PR):
- Current row: `_valid_to` = change date - 1, `_is_current` = False
- New row: `_valid_from` = change date, `_valid_to` = 9999-12-31, `_is_current` = True

**Why these two attributes?** A student's visa status directly affects:
- Revenue classification (domestic fee vs international fee)
- ESOS compliance reporting
- Historical accuracy: fact rows from 2023 must reflect the 2023 student profile

### silver.application
Conformed application records from Dynamics, enriched with lead data from HubSpot.

```
application_id      VARCHAR
student_id          VARCHAR   -- resolved from student_email
lead_id             VARCHAR   -- nullable (if HubSpot lead exists)
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR
application_date    DATE
offer_date          DATE      -- nullable
offer_type          VARCHAR   -- nullable
decision_date       DATE      -- nullable
decision_outcome    VARCHAR   -- nullable
_loaded_at          TIMESTAMP
```

### silver.enrolment
```
enrolment_id        VARCHAR
student_id          VARCHAR   -- resolved from student_email
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR
enrolment_date      DATE
enrolment_status    VARCHAR
withdrawal_date     DATE      -- nullable
completion_date     DATE      -- nullable
enrolment_fee       DECIMAL
scholarship_amount  DECIMAL
net_fee_payable     DECIMAL   -- calculated: enrolment_fee - scholarship_amount
credit_points       INT
_loaded_at          TIMESTAMP
```

### silver.attendance
```
session_id          VARCHAR
student_id          VARCHAR   -- resolved from student_email
course_code         VARCHAR
campus_code         VARCHAR
intake_code         VARCHAR
session_date        DATE
session_type        VARCHAR
attended            INT       -- 1 or 0
_loaded_at          TIMESTAMP
```

### silver.course / silver.campus / silver.intake
Reference tables built from distinct values across all source systems.
Conformed: one row per course_code, campus_code, intake_code.

---

## Gold Layer — Conformed Dimensions

**Notebook:** `03_gold_dimensions.ipynb`

All dimension surrogate keys are integers generated with `monotonically_increasing_id()`
or a row_number window function. Natural keys retained as alternate keys.

### gold.dim_date
Pre-populated for the full range of dates in the dataset plus 2 years either side.
One row per calendar day.

```
date_key              INT       -- PK: YYYYMMDD integer e.g. 20240115
full_date             DATE
day_of_week           STRING    -- 'Monday', 'Tuesday', ...
day_number_in_month   INT
month_number          INT
month_name            STRING
calendar_quarter      INT       -- 1, 2, 3, 4
calendar_year         INT
-- Academic calendar (JMC-specific)
intake_code           STRING    -- nullable e.g. '2024-S1'
intake_name           STRING    -- nullable e.g. 'Semester 1 2024'
academic_year         STRING    -- nullable e.g. '2024'
is_teaching_week      BOOLEAN
teaching_week_number  INT       -- nullable: 1–18 within intake
is_census_date        BOOLEAN
is_orientation_week   BOOLEAN
```

**Special row:** `date_key = 0` reserved for TBD (unresolved milestone dates
in the accumulating snapshot). Must be seeded before any fact notebook runs.

### gold.dim_student
Built from `silver.student` where `_is_current = True` for Type 1 attributes.
For Type 2 attributes, all historical rows are included.

```
student_sk            INT       -- surrogate key (PK)
student_id            VARCHAR   -- natural key (alternate key)
student_first_name    VARCHAR   -- SCD Type 1 (overwrite)
student_last_name     VARCHAR   -- SCD Type 1
student_email         VARCHAR   -- SCD Type 1
date_of_birth         DATE      -- SCD Type 1
visa_code             VARCHAR   -- SCD Type 2 (history tracked)
visa_description      VARCHAR   -- denormalised from ref
is_international      BOOLEAN   -- SCD Type 2 (history tracked)
student_type          STRING    -- 'Domestic' or 'International' (derived)
lead_source           VARCHAR   -- SCD Type 1
-- SCD Type 2 admin
_valid_from           DATE
_valid_to             DATE
_is_current           BOOLEAN
```

### gold.dim_course
```
course_sk             INT       -- surrogate key (PK)
course_id             VARCHAR   -- natural key (course_code)
course_name           VARCHAR
qualification_level   VARCHAR   -- 'Bachelor', 'Master', 'Diploma', etc.
aqf_level             INT       -- Australian Qualifications Framework level
credit_points         INT
duration_semesters    INT
is_cricos_registered  BOOLEAN   -- international student eligibility
is_active             BOOLEAN
```

### gold.dim_campus
```
campus_sk             INT
campus_id             VARCHAR   -- natural key (campus_code)
campus_name           VARCHAR
city                  VARCHAR
state_code            VARCHAR   -- 'NSW', 'VIC', etc.
country               VARCHAR   -- default 'Australia'
delivery_mode         VARCHAR   -- 'On Campus', 'Online', 'Blended'
is_active             BOOLEAN
```

### gold.dim_intake
```
intake_sk             INT
intake_id             VARCHAR   -- natural key (intake_code e.g. '2024-S1')
intake_name           VARCHAR   -- e.g. 'Semester 1 2024'
academic_year         VARCHAR
semester_number       INT       -- 1 or 2
start_date_key        INT       -- FK → dim_date
end_date_key          INT       -- FK → dim_date
census_date_key       INT       -- FK → dim_date
```

### gold.dim_lead_source
```
lead_source_sk        INT
lead_source_id        VARCHAR   -- natural key
source_name           VARCHAR   -- 'Agent', 'Direct', 'Social Media', 'Web Referral'
source_category       VARCHAR   -- 'Paid', 'Organic', 'Referral'
```

### gold.dim_status
Reusable across all fact tables. Status category field distinguishes context.

```
status_sk             INT
status_id             VARCHAR   -- natural key (status_code + category)
status_code           VARCHAR   -- 'Active', 'Withdrawn', 'Offered', etc.
status_description    VARCHAR
status_category       VARCHAR   -- 'Application', 'Enrolment', 'Offer'
is_terminal           BOOLEAN   -- no further state changes expected
```

---

## Gold Layer — Fact Tables

### fact_application_pipeline (Accumulating Snapshot)
**Notebook:** `04_gold_fact_pipeline.ipynb`

**Grain:** One row per student application. Inserted at lead capture.
Updated via Delta MERGE as each milestone completes.

```
-- Surrogate key
pipeline_sk               INT         -- PK

-- Dimension foreign keys
student_sk                INT         -- FK → dim_student (current record at insert time)
course_sk                 INT         -- FK → dim_course
campus_sk                 INT         -- FK → dim_campus
intake_sk                 INT         -- FK → dim_intake
lead_source_sk            INT         -- FK → dim_lead_source

-- Six milestone date foreign keys (0 = TBD, not null)
lead_date_key             INT         -- FK → dim_date: when lead captured
visit_date_key            INT         -- FK → dim_date: when campus visited (0 if skipped)
application_date_key      INT         -- FK → dim_date: when application submitted
offer_date_key            INT         -- FK → dim_date: when offer made
decision_date_key         INT         -- FK → dim_date: when decision recorded
enrolment_date_key        INT         -- FK → dim_date: when enrolment confirmed

-- Degenerate dimensions (natural keys for traceability)
lead_id                   VARCHAR
application_id            VARCHAR     -- nullable until application stage reached

-- Lag facts (integer days, NULL until both milestone dates are known)
lead_to_visit_days        INT         -- visit_date - lead_date
lead_to_application_days  INT         -- application_date - lead_date
application_to_offer_days INT         -- offer_date - application_date
offer_to_decision_days    INT         -- decision_date - offer_date
decision_to_enrolment_days INT        -- enrolment_date - decision_date
lead_to_enrolment_days    INT         -- enrolment_date - lead_date (full pipeline lag)

-- Milestone flags (additive: SUM = count of students who reached this stage)
is_lead                   INT         -- always 1
is_visited                INT         -- 1 when visit_date known
is_applied                INT         -- 1 when application_date known
is_offered                INT         -- 1 when offer_date known
is_accepted               INT         -- 1 when decision = 'Accepted'
is_enrolled               INT         -- 1 when enrolment_date known
is_withdrawn_pre_enrol    INT         -- 1 when dropped before enrolment

-- Audit
_loaded_at                TIMESTAMP
_last_updated_at          TIMESTAMP
```

**MERGE logic (executed each notebook run):**
```
Source = silver.application joined with silver.student, dim_* lookups
Target = gold.fact_application_pipeline

WHEN MATCHED AND new milestone data available:
  UPDATE offer_date_key, application_to_offer_days, is_offered, ...
  UPDATE _last_updated_at

WHEN NOT MATCHED:
  INSERT new row with lead milestone populated, all others = 0 or NULL
```

**Lag calculation rule:** A lag column is calculated and written permanently
when BOTH endpoint dates first become non-zero. It is never recalculated.
NULL means "not yet calculable — milestone has not occurred."

---

### fact_enrolment (Transaction Fact)
**Notebook:** `05_gold_fact_enrolment.ipynb`

**Grain:** One row per confirmed enrolment event. Inserted once, never updated.

```
enrolment_sk              INT         -- PK surrogate
student_sk                INT         -- FK → dim_student
course_sk                 INT         -- FK → dim_course
campus_sk                 INT         -- FK → dim_campus
intake_sk                 INT         -- FK → dim_intake
enrolment_date_key        INT         -- FK → dim_date
status_sk                 INT         -- FK → dim_status

enrolment_id              VARCHAR     -- degenerate dimension

-- Measures (all additive unless noted)
enrolment_fee             DECIMAL     -- gross fee
scholarship_amount        DECIMAL     -- discount applied
net_fee_payable           DECIMAL     -- enrolment_fee - scholarship_amount
credit_points             INT
is_withdrawal             INT         -- 1 if status = Withdrawn
is_completion             INT         -- 1 if status = Completed

_loaded_at                TIMESTAMP
```

---

### fact_attendance_coverage (Factless — Coverage)
**Notebook:** `06_gold_fact_attendance.ipynb`

**Grain:** One row per enrolled student per scheduled class session.
Represents what SHOULD have happened. No measures — presence of the row is the fact.

```
coverage_sk               INT         -- PK surrogate
student_sk                INT         -- FK → dim_student
course_sk                 INT         -- FK → dim_course
campus_sk                 INT         -- FK → dim_campus
intake_sk                 INT         -- FK → dim_intake
session_date_key          INT         -- FK → dim_date
session_id                VARCHAR     -- degenerate dimension
session_type              VARCHAR     -- degenerate: 'Lecture', 'Tutorial', 'Workshop'
coverage_count            INT         -- always 1 (additive — count scheduled slots)
_loaded_at                TIMESTAMP
```

### fact_attendance_actual (Factless — Activity)
**Grain:** One row per student per session actually attended.

Same schema as `fact_attendance_coverage` with `attendance_count = 1`.

**At-risk query pattern:**
Students who are in coverage but have zero matching rows in actual
= enrolled but never attended = at-risk students for early intervention.

```sql
SELECT s.student_id, s.student_first_name, c.course_name
FROM gold.fact_attendance_coverage cov
JOIN gold.dim_student s ON s.student_sk = cov.student_sk AND s._is_current = true
JOIN gold.dim_course c ON c.course_sk = cov.course_sk
WHERE NOT EXISTS (
    SELECT 1 FROM gold.fact_attendance_actual act
    WHERE act.student_sk = cov.student_sk
    AND act.course_sk = cov.course_sk
    AND act.intake_sk = cov.intake_sk
)
GROUP BY s.student_id, s.student_first_name, c.course_name
```

---

## Dataset Scenarios (10 Students)

| student_id | Name | Journey | Key scenario demonstrated |
|---|---|---|---|
| STU001 | Alex Chen | Lead only | Funnel drop-off at stage 1 (never applied) |
| STU002 | Maria Santos | Lead + visit | Drop-off after campus visit |
| STU003 | James Kim | Lead → applied → no offer | Application rejected |
| STU004 | Priya Patel | Lead → applied → offered → rejected offer | Offer not accepted |
| STU005 | Tom Wilson | Lead → enrolled → withdrew | Post-enrolment withdrawal |
| STU006 | Linh Nguyen | Lead → enrolled → never attended | At-risk (factless fact scenario) |
| STU007 | Amir Hassan | Lead → enrolled (visa change) | SCD Type 2 trigger: domestic → international |
| STU008 | Sophie Brown | Full journey | Clean completion baseline |
| STU009 | Daniel Park | Full journey | Clean completion baseline |
| STU010 | Emma Davis | Full journey | Clean completion baseline |

---

## Business Questions Each Fact Table Answers

### fact_application_pipeline
- What is the lead-to-enrolment conversion rate by course?
- Which stage of the pipeline has the highest drop-off?
- How many days on average does it take from application to offer?
- How many applications are still open (offer_date_key = 0)?
- Which lead source produces the highest conversion rate?

### fact_enrolment
- What is total enrolment fee revenue by course and intake?
- What is the withdrawal rate by campus?
- How many credit points are enrolled students carrying this semester?
- What is the net revenue after scholarships by course?

### fact_attendance
- Which students are enrolled but have never attended a session?
- What is the attendance rate by course and session type?
- Which courses have attendance below 70% in the first four weeks?

---

## Power BI Setup

**Connection type:** Direct Lake
**Semantic model:** Created from Fabric Lakehouse Gold schema
**Relationships:**
- All fact tables → dim_date (multiple role-playing relationships via inactive relationships)
- All fact tables → dim_student, dim_course, dim_campus, dim_intake
- Inactive relationships activated in DAX measures with `USERELATIONSHIP()`

**Key measures:**
```
Conversion Rate =
DIVIDE(
    SUM(fact_application_pipeline[is_enrolled]),
    SUM(fact_application_pipeline[is_lead])
)

Avg Pipeline Days =
AVERAGEX(
    FILTER(fact_application_pipeline, fact_application_pipeline[lead_to_enrolment_days] <> BLANK()),
    fact_application_pipeline[lead_to_enrolment_days]
)

At Risk Students =
CALCULATE(
    DISTINCTCOUNT(fact_attendance_coverage[student_sk]),
    NOT fact_attendance_coverage[student_sk] IN
        VALUES(fact_attendance_actual[student_sk])
)
```

---

## Key Design Decisions

| Decision | What | Why |
|---|---|---|
| Silver is not 3NF | Wide flat tables, not normalised entities | Silver is a conforming layer for Gold loading — not an OLTP system |
| Lag facts stored as integers | Pre-computed at ETL time | Power BI reads one integer column — no expensive DAX subtraction at render time |
| MERGE not full reload | Delta `whenMatchedUpdate` + `whenNotMatchedInsert` | Accumulating snapshot semantics require per-column updates, not row replacement |
| date_key = 0 for TBD | Special row in dim_date | Fact table FKs must never be NULL — unresolved milestones point to the TBD row |
| Two attendance tables | Coverage + Actual | Cannot detect absence with a single table — need the set difference pattern |
| SCD Type 2 on visa/international only | Minimal Type 2 scope | These two attributes affect revenue and compliance classification historically |
| Direct Lake over Import | Fabric-native connection | No data copy, no refresh pipeline — Gold Delta tables immediately visible in Power BI |
