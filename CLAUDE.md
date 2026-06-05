# CLAUDE.md — edu-analytics-fabric

## What this project is

A portfolio data engineering project demonstrating a full medallion architecture
(Bronze → Silver → Gold) on Microsoft Fabric Lakehouse, with a dimensional model
in the Gold layer queried by Power BI via Direct Lake mode.

The domain is a fictional higher education provider tracking the complete student
lifecycle — from first lead inquiry through to enrolment — across four source systems.

This is a demo project. Dataset is small (~10 students, static CSVs) by design.
The goal is to demonstrate correct data modeling logic and pipeline execution,
not scale.

---

## Stack

| Component | Tool |
|---|---|
| Lakehouse | Microsoft Fabric Lakehouse (trial workspace) |
| Storage | OneLake (Delta tables) |
| Transformation | PySpark notebooks in Fabric |
| Orchestration | Manual notebook execution (demo scope) |
| Reporting | Power BI via Direct Lake mode |
| Source control | GitHub (manual copy-paste to Fabric) |
| Source data | Static CSV files uploaded to Lakehouse Files section |

---

## Repository structure

```
edu-analytics-fabric/
├── CLAUDE.md                          ← you are here
├── architecture.md                    ← full architecture and design decisions
├── README.md                          ← portfolio-facing project summary
├── data/
│   └── raw/
│       ├── hubspot_leads.csv          ← source 1: leads and campus visits
│       ├── dynamics_applications.csv  ← source 2: applications and offers
│       ├── paradigm_enrolments.csv    ← source 3: confirmed enrolments
│       └── canvas_attendance.csv      ← source 4: class session attendance
├── notebooks/
│   ├── 01_bronze_ingest.ipynb         ← CSV → raw Delta (add metadata columns)
│   ├── 02_silver_transform.ipynb      ← clean, conform, deduplicate, SCD Type 2
│   ├── 03_gold_dimensions.ipynb       ← build all conformed dimension tables
│   ├── 04_gold_fact_pipeline.ipynb    ← accumulating snapshot + Delta MERGE
│   ├── 05_gold_fact_enrolment.ipynb   ← transaction fact (revenue, credit points)
│   └── 06_gold_fact_attendance.ipynb  ← factless fact (at-risk student detection)
├── docs/
│   ├── data_model.md                  ← table-by-table field specs
│   └── business_questions.md         ← questions each fact table answers
└── screenshots/
    └── powerbi/                       ← Power BI report screenshots for portfolio
```

---

## Medallion layers

### Bronze
- Raw copy of each CSV file
- No transformations, no business logic
- Adds three metadata columns only: `_source_file`, `_source_system`, `_ingested_at`
- Stored as Delta tables in Lakehouse Tables section under `bronze` schema
- Partition strategy: partition by `_source_system` for query pruning

### Silver
- Cleaned and conformed staging tables — one per subject area
- Responsibilities: trim whitespace, cast types, handle nulls, deduplicate,
  standardise codes (e.g. "JMC Academy" vs "JMCA"), resolve student identity
  across source systems
- SCD Type 2 implemented on `silver_student` for: `visa_code`, `is_international`
  (these attributes affect revenue and compliance reporting historically)
- NOT 3NF — silver is wide and flat, optimised for downstream Gold loading
- Stored as Delta tables under `silver` schema

### Gold
- Star schema dimensional model (Kimball)
- Conformed dimensions shared across all fact tables
- Three fact tables covering different Kimball fact types (see below)
- Stored as Delta tables under `gold` schema
- Queried by Power BI via Direct Lake — no import, no refresh pipeline

---

## Fact tables

### fact_application_pipeline (Accumulating Snapshot)
The centrepiece of the model. One row per student application. Row is inserted
at lead capture and UPDATED as each milestone completes via Delta MERGE.

Milestones tracked:
1. Lead captured (HubSpot)
2. Campus visit (HubSpot)
3. Application submitted (Dynamics)
4. Offer made (Dynamics)
5. Offer accepted or rejected (Dynamics)
6. Enrolled or withdrawn (Paradigm)

Each milestone has: a date FK (points to dim_date, surrogate 0 = TBD),
a lag integer (days since previous milestone, NULL until calculable),
and a milestone flag (0/1 additive count).

MERGE pattern: whenMatchedUpdate (update milestone columns only) +
whenNotMatchedInsert (new application row).

### fact_enrolment (Transaction)
One row per confirmed enrolment event. Inserted once, never updated.
Measures: enrolment_fee, scholarship_amount, net_fee_payable, credit_points,
is_withdrawal (flag), is_completion (flag).

### fact_attendance (Factless — Activity + Coverage)
Two tables working together:
- `fact_attendance_coverage`: one row per enrolled student per scheduled session
  (what SHOULD have happened)
- `fact_attendance_actual`: one row per student per session actually attended
  (what DID happen)

At-risk query: students in coverage who have NO matching rows in actual.

---

## Conformed dimensions

| Dimension | SCD Type | Key attributes |
|---|---|---|
| dim_student | Type 2 | visa_code, is_international (history tracked) |
| dim_course | Type 1 | course_name, qualification_level, credit_points |
| dim_campus | Type 1 | campus_name, state, delivery_mode |
| dim_date | Static | Full calendar + academic calendar (semester, teaching week, census date) |
| dim_intake | Type 1 | intake_name, start_date, end_date |
| dim_lead_source | Type 1 | source_name, source_category |
| dim_status | Type 1 | status_description, status_category (reused across fact tables) |

All surrogate keys are integers. Natural keys retained as alternate keys.
dim_date uses integer YYYYMMDD as PK. Surrogate key 0 reserved for TBD
(unresolved milestone dates in accumulating snapshot).

---

## Dataset design (10 students)

Deliberately engineered to exercise every scenario:

| Student | Journey outcome |
|---|---|
| STU001–STU002 | Lead captured, never applied (drop-off at stage 1) |
| STU003–STU004 | Applied, offer made, never accepted (drop-off at stage 3) |
| STU005 | Accepted offer, enrolled, then withdrew |
| STU006 | Enrolled, never attended a single class (at-risk scenario) |
| STU007 | Enrolled, changed visa status mid-pipeline (SCD Type 2 trigger) |
| STU008–STU010 | Full journey: lead → enrolled → attending → completed |

---

## Execution order

Always run notebooks in this order:
1. `01_bronze_ingest` — must run first, all subsequent notebooks depend on bronze Delta tables
2. `02_silver_transform` — depends on bronze
3. `03_gold_dimensions` — depends on silver; must complete before any fact notebook
4. `04_gold_fact_pipeline` — depends on gold dimensions
5. `05_gold_fact_enrolment` — depends on gold dimensions
6. `06_gold_fact_attendance` — depends on gold dimensions

Notebooks 4, 5, 6 are independent of each other and can run in any order
after notebook 3 completes.

---

## Key design decisions

### Why Silver is not 3NF
Silver is a conforming layer, not an OLTP layer. Its job is to clean and
standardise data for Gold loading — not to enforce write integrity. Wide,
flat staging tables are intentional.

### Why lag facts are stored as integers, not calculated in DAX
Pre-computing lag at ETL time (pipeline) means Power BI reads a simple integer
column. Calculating in DAX means every visual render re-executes the subtraction
across the full dataset. For large fact tables this is a significant performance
difference. NULL when milestone not yet reached — no defensive DAX needed.

### Why Direct Lake over Import mode
Direct Lake reads Delta files directly from OneLake — no data copy, no refresh
pipeline. Gold Delta tables updated by notebooks are immediately visible in
Power BI. This is the Fabric-native pattern and demonstrates understanding of
the full Fabric architecture.

### Why the accumulating snapshot uses MERGE not full reload
A full reload would lose the history of which columns were updated when.
The MERGE pattern (Delta `whenMatchedUpdate` + `whenNotMatchedInsert`) updates
only the milestone columns that have changed — preserving all previously
calculated lag values and leaving TBD date keys (0) for future milestones.
This is the correct Kimball implementation.

### Why two attendance tables instead of one
A single attendance table can only record what happened. To detect students
who were scheduled but never showed up, you need a coverage table (what should
have happened) and subtract. This is the Kimball factless fact coverage pattern.

---

## Notes for Claude Code sessions

- Always check `architecture.md` for the full table schemas before generating
  any DDL or notebook code
- All Delta table paths follow the pattern:
  `abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.Lakehouse/Tables/<schema>/<table>`
- In Fabric notebooks, the Lakehouse is mounted and tables are accessible via
  `spark.read.format("delta").load()` or via the `lakehouse.schema.table` shorthand
- Surrogate key 0 in dim_date must be seeded before any fact table notebook runs
- SCD Type 2 logic in silver: use `_valid_from`, `_valid_to`, `_is_current` columns
- Notebook cells should include markdown cells explaining the logic — this is
  a portfolio project, readability matters as much as correctness
