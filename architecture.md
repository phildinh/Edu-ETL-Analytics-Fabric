# architecture.md — edu-analytics-fabric

## Problem Statement

A higher education provider runs four operational systems:
- **HubSpot** — lead capture and marketing CRM
- **Dynamics 365** — admissions CRM (applications, offers)
- **Paradigm SIS** — student information system (enrolments, academic records)
- **Canvas LMS** — learning management (attendance, assessments)

No single system can answer: *"Show me lead-to-enrolment conversion by course
and campus, with revenue, for this intake versus last intake."*

This project builds that single place — a Kimball dimensional model on
Microsoft Fabric Lakehouse, queried by Power BI via Direct Lake.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  SOURCE SYSTEMS (static CSV files)                                │
│  hubspot_leads.csv  dynamics_applications.csv                     │
│  paradigm_enrolments.csv  canvas_attendance.csv                   │
└────────────────────────┬─────────────────────────────────────────┘
                         │ Manual upload → Lakehouse Files/
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  BRONZE  (schema: bronze)   01_bronze_ingest.ipynb               │
│  Raw Delta tables — exact source copy + 3 metadata columns       │
│  bronze.hubspot_leads   bronze.dynamics_applications             │
│  bronze.paradigm_enrolments   bronze.canvas_attendance           │
└────────────────────────┬─────────────────────────────────────────┘
                         │ PySpark read → clean → write
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  SILVER  (schema: silver)   02_silver_transform.ipynb            │
│  Cleaned, conformed, deduplicated — wide flat tables, NOT 3NF   │
│  SCD Type 2 on silver.student (visa_code, is_international)      │
│  silver.student   silver.application   silver.enrolment          │
│  silver.attendance   silver.course   silver.campus  silver.intake│
└────────────────────────┬─────────────────────────────────────────┘
                         │ PySpark read → model → write
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  GOLD  (schema: gold)   Notebooks 03 → 04, 05, 06               │
│  Star schema — Kimball dimensional model                         │
│                                                                   │
│  DIMENSIONS              FACTS                                    │
│  gold.dim_date           gold.fact_application_pipeline          │
│  gold.dim_student        gold.fact_enrolment                     │
│  gold.dim_course         gold.fact_attendance_coverage           │
│  gold.dim_campus         gold.fact_attendance_actual             │
│  gold.dim_intake                                                  │
│  gold.dim_lead_source                                             │
│  gold.dim_status                                                  │
└────────────────────────┬─────────────────────────────────────────┘
                         │ Direct Lake — no copy, no refresh
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  POWER BI — Direct Lake mode                                      │
│  Reads Gold Delta tables directly from OneLake                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Bus Matrix

| Fact Table | Type | dim_date | dim_student | dim_course | dim_campus | dim_intake | dim_lead_source | dim_status |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| fact_application_pipeline | Accumulating snapshot | ✓ ×6 | ✓ | ✓ | ✓ | ✓ | ✓ | |
| fact_enrolment | Transaction | ✓ | ✓ | ✓ | ✓ | ✓ | | ✓ |
| fact_attendance_coverage | Factless (coverage) | ✓ | ✓ | ✓ | ✓ | ✓ | | |
| fact_attendance_actual | Factless (activity) | ✓ | ✓ | ✓ | ✓ | ✓ | | |

---

## Source CSV Schemas

### hubspot_leads.csv
```
lead_id              VARCHAR   -- natural key from HubSpot
student_first_name   VARCHAR
student_last_name    VARCHAR
student_email        VARCHAR   -- identity resolution key
date_of_birth        DATE
visa_type            VARCHAR   -- 'Domestic', 'International - Student Visa'
lead_source          VARCHAR   -- 'Agent', 'Direct', 'Social Media', 'Web Referral'
course_interest      VARCHAR   -- course_code e.g. BMUS301
campus_preference    VARCHAR   -- campus_code e.g. SYD, MEL
lead_date            DATE
visit_date           DATE      -- nullable
```

### dynamics_applications.csv
```
application_id       VARCHAR   -- natural key from Dynamics
lead_id              VARCHAR   -- FK to HubSpot lead
student_email        VARCHAR   -- identity resolution key
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR   -- e.g. 2024-S1
application_date     DATE
offer_date           DATE      -- nullable
offer_type           VARCHAR   -- 'Conditional', 'Unconditional', nullable
decision_date        DATE      -- nullable
decision_outcome     VARCHAR   -- 'Accepted', 'Rejected', nullable
```

### paradigm_enrolments.csv
```
enrolment_id         VARCHAR   -- natural key from Paradigm
student_email        VARCHAR   -- identity resolution key
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR
enrolment_date       DATE
enrolment_status     VARCHAR   -- 'Active', 'Withdrawn', 'Completed'
withdrawal_date      DATE      -- nullable
completion_date      DATE      -- nullable
enrolment_fee        DECIMAL
scholarship_amount   DECIMAL
credit_points        INT
```

### canvas_attendance.csv
```
session_id           VARCHAR   -- unique class session identifier
student_email        VARCHAR   -- identity resolution key
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR
session_date         DATE
session_type         VARCHAR   -- 'Lecture', 'Tutorial', 'Workshop'
attended             INT       -- 1 = attended, 0 = scheduled but absent
```

---

## Bronze Layer

**Notebook:** `01_bronze_ingest.ipynb`

Metadata columns added to every bronze table (nothing else):
```
_source_file      STRING     -- e.g. 'hubspot_leads.csv'
_source_system    STRING     -- 'HUBSPOT', 'DYNAMICS', 'PARADIGM', 'CANVAS'
_ingested_at      TIMESTAMP  -- UTC ingestion timestamp
```

Metadata column `_ingestion_date` = DATE portion of `_ingested_at` — used as partition key.
Partition all bronze tables by `(_source_system, _ingestion_date)`. Mode: append not overwrite.
Each pipeline run creates a new date partition — full history preserved for audit and replay.
Silver reads only the partition matching `pipeline_run_date` — prevents duplicates on reruns.

---

## Silver Layer

**Notebook:** `02_silver_transform.ipynb`

Wide, flat staging tables — one per subject area. Not 3NF.
Identity resolution across all sources uses `student_email` as the common key.

### silver.student
```
student_id           VARCHAR   -- from Paradigm when available, else generated
hubspot_lead_id      VARCHAR   -- nullable
dynamics_contact_id  VARCHAR   -- nullable
student_first_name   VARCHAR
student_last_name    VARCHAR
student_email        VARCHAR
date_of_birth        DATE
visa_code            VARCHAR   -- SCD Type 2
is_international     BOOLEAN   -- SCD Type 2
lead_source          VARCHAR
_valid_from          DATE
_valid_to            DATE      -- 9999-12-31 for current record
_is_current          BOOLEAN
_source_system       VARCHAR
_loaded_at           TIMESTAMP
```

### silver.application
```
application_id       VARCHAR
student_id           VARCHAR   -- resolved from student_email
lead_id              VARCHAR   -- nullable
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR
application_date     DATE
offer_date           DATE      -- nullable
offer_type           VARCHAR   -- nullable
decision_date        DATE      -- nullable
decision_outcome     VARCHAR   -- nullable
_loaded_at           TIMESTAMP
```

### silver.enrolment
```
enrolment_id         VARCHAR
student_id           VARCHAR   -- resolved from student_email
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR
enrolment_date       DATE
enrolment_status     VARCHAR
withdrawal_date      DATE      -- nullable
completion_date      DATE      -- nullable
enrolment_fee        DECIMAL
scholarship_amount   DECIMAL
net_fee_payable      DECIMAL   -- calculated: enrolment_fee - scholarship_amount
credit_points        INT
_loaded_at           TIMESTAMP
```

### silver.attendance
```
session_id           VARCHAR
student_id           VARCHAR   -- resolved from student_email
course_code          VARCHAR
campus_code          VARCHAR
intake_code          VARCHAR
session_date         DATE
session_type         VARCHAR
attended             INT       -- 1 or 0
_loaded_at           TIMESTAMP
```

### silver.course / silver.campus / silver.intake
Distinct reference values conformed across all source systems.
One row per `course_code`, `campus_code`, `intake_code`.

---

## Gold Layer — Dimensions

**Notebook:** `03_gold_dimensions.ipynb`

Surrogate keys: integers generated with `row_number()` — deterministic.
Natural keys retained as alternate keys alongside surrogate keys.
Seed `dim_date` TBD row (`date_key = 0`) before exiting this notebook.

### gold.dim_date
```
date_key             INT       -- PK: YYYYMMDD integer (0 = TBD)
full_date            DATE
day_of_week          STRING
day_number_in_month  INT
month_number         INT
month_name           STRING
calendar_quarter     INT
calendar_year        INT
intake_code          STRING    -- nullable
intake_name          STRING    -- nullable
academic_year        STRING    -- nullable
is_teaching_week     BOOLEAN
teaching_week_number INT       -- nullable: 1–18 within intake
is_census_date       BOOLEAN
```

### gold.dim_student  (SCD Type 2)
```
student_sk           INT       -- surrogate key (PK)
student_id           VARCHAR   -- natural key
student_first_name   VARCHAR   -- Type 1
student_last_name    VARCHAR   -- Type 1
student_email        VARCHAR   -- Type 1
date_of_birth        DATE      -- Type 1
visa_code            VARCHAR   -- Type 2 (history tracked)
visa_description     VARCHAR   -- Type 2 (denormalised)
is_international     BOOLEAN   -- Type 2 (history tracked)
student_type         STRING    -- 'Domestic' or 'International' (derived)
lead_source          VARCHAR   -- Type 1
_valid_from          DATE
_valid_to            DATE
_is_current          BOOLEAN
```

### gold.dim_course  (SCD Type 1)
```
course_sk            INT
course_id            VARCHAR   -- natural key (course_code)
course_name          VARCHAR
qualification_level  VARCHAR
aqf_level            INT
credit_points        INT
duration_semesters   INT
is_cricos_registered BOOLEAN
is_active            BOOLEAN
```

### gold.dim_campus  (SCD Type 1)
```
campus_sk            INT
campus_id            VARCHAR   -- natural key (campus_code)
campus_name          VARCHAR
city                 VARCHAR
state_code           VARCHAR
country              VARCHAR   -- default 'Australia'
delivery_mode        VARCHAR   -- 'On Campus', 'Online', 'Blended'
is_active            BOOLEAN
```

### gold.dim_intake  (SCD Type 1)
```
intake_sk            INT
intake_id            VARCHAR   -- natural key (intake_code)
intake_name          VARCHAR   -- e.g. 'Semester 1 2024'
academic_year        VARCHAR
semester_number      INT
start_date_key       INT       -- FK → dim_date
end_date_key         INT       -- FK → dim_date
census_date_key      INT       -- FK → dim_date
```

### gold.dim_lead_source  (SCD Type 1)
```
lead_source_sk       INT
lead_source_id       VARCHAR
source_name          VARCHAR   -- 'Agent', 'Direct', 'Social Media', 'Web Referral'
source_category      VARCHAR   -- 'Paid', 'Organic', 'Referral'
```

### gold.dim_status  (SCD Type 1)
Reused across all fact tables. `status_category` distinguishes context.
```
status_sk            INT
status_id            VARCHAR   -- natural key (code + category)
status_code          VARCHAR
status_description   VARCHAR
status_category      VARCHAR   -- 'Application', 'Enrolment', 'Offer'
is_terminal          BOOLEAN
```

---

## Gold Layer — Fact Tables

### gold.fact_application_pipeline  (Accumulating Snapshot)
**Notebook:** `04_gold_fact_pipeline.ipynb`
**Grain:** One row per student application. Inserted at lead capture. Updated via Delta MERGE as milestones complete.

```
pipeline_sk                INT       -- PK surrogate

-- Dimension FKs
student_sk                 INT       -- FK → dim_student
course_sk                  INT       -- FK → dim_course
campus_sk                  INT       -- FK → dim_campus
intake_sk                  INT       -- FK → dim_intake
lead_source_sk             INT       -- FK → dim_lead_source

-- Six milestone date FKs (0 = TBD, never NULL)
lead_date_key              INT       -- FK → dim_date
visit_date_key             INT       -- FK → dim_date
application_date_key       INT       -- FK → dim_date
offer_date_key             INT       -- FK → dim_date
decision_date_key          INT       -- FK → dim_date
enrolment_date_key         INT       -- FK → dim_date

-- Degenerate dimensions
lead_id                    VARCHAR
application_id             VARCHAR   -- nullable until application stage

-- Lag facts (INT, NULL until both endpoint dates known)
lead_to_visit_days         INT
lead_to_application_days   INT
application_to_offer_days  INT
offer_to_decision_days     INT
decision_to_enrolment_days INT
lead_to_enrolment_days     INT       -- full pipeline lag

-- Milestone flags (additive INT: SUM = count reached this stage)
is_lead                    INT       -- always 1
is_visited                 INT
is_applied                 INT
is_offered                 INT
is_accepted                INT
is_enrolled                INT
is_withdrawn_pre_enrol     INT

_loaded_at                 TIMESTAMP
_last_updated_at           TIMESTAMP
```

---

### gold.fact_enrolment  (Transaction)
**Notebook:** `05_gold_fact_enrolment.ipynb`
**Grain:** One row per confirmed enrolment. Inserted once, never updated.

```
enrolment_sk               INT       -- PK surrogate
student_sk                 INT       -- FK → dim_student
course_sk                  INT       -- FK → dim_course
campus_sk                  INT       -- FK → dim_campus
intake_sk                  INT       -- FK → dim_intake
enrolment_date_key         INT       -- FK → dim_date
status_sk                  INT       -- FK → dim_status
enrolment_id               VARCHAR   -- degenerate dimension
enrolment_fee              DECIMAL
scholarship_amount         DECIMAL
net_fee_payable            DECIMAL
credit_points              INT
is_withdrawal              INT       -- 1 if Withdrawn
is_completion              INT       -- 1 if Completed
_loaded_at                 TIMESTAMP
```

---

### gold.fact_attendance_coverage  (Factless — Coverage)
### gold.fact_attendance_actual  (Factless — Activity)
**Notebook:** `06_gold_fact_attendance.ipynb`
**Grain:** One row per enrolled student per scheduled session (coverage) /
per session actually attended (actual).

```
-- Shared schema for both tables
attendance_sk              INT       -- PK surrogate
student_sk                 INT       -- FK → dim_student
course_sk                  INT       -- FK → dim_course
campus_sk                  INT       -- FK → dim_campus
intake_sk                  INT       -- FK → dim_intake
session_date_key           INT       -- FK → dim_date
session_id                 VARCHAR   -- degenerate dimension
session_type               VARCHAR   -- degenerate: 'Lecture', 'Tutorial', 'Workshop'
record_count               INT       -- always 1 (additive)
_loaded_at                 TIMESTAMP
```

At-risk detection: students in coverage with no matching rows in actual.

---

## Business Questions by Fact Table

### fact_application_pipeline
- Lead-to-enrolment conversion rate by course and intake
- Which pipeline stage has the highest drop-off?
- Average days from application to offer by campus
- How many applications are still open? (`offer_date_key = 0`)
- Which lead source produces the highest conversion rate?

### fact_enrolment
- Total and net revenue by course and intake
- Withdrawal rate by campus
- Credit point load by course and intake
- Revenue impact of scholarships

### fact_attendance
- Students enrolled but never attended (at-risk)
- Attendance rate by course, session type, and teaching week
- Courses with attendance below threshold in first four weeks

---

## Key Design Decisions

| Decision | Why |
|---|---|
| Silver is wide and flat — not 3NF | Silver is a conforming layer for Gold loading, not an OLTP system |
| Lag facts stored as integers at ETL time | Power BI reads one pre-computed integer — no DAX subtraction on every render |
| Delta MERGE not full reload | Accumulating snapshot requires per-column updates — full reload loses milestone history |
| `date_key = 0` for TBD milestones | Fact FKs must never be NULL — unresolved dates point to the TBD row |
| Two attendance tables | Cannot detect absence with one table — requires coverage minus actual set difference |
| SCD Type 2 on visa + is_international only | These two attributes affect revenue classification and compliance historically |
| Direct Lake over Import | No data copy, no refresh pipeline — Gold Delta tables immediately visible in Power BI |
