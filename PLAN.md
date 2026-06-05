# PLAN.md — edu-analytics-fabric

Authoritative execution plan. Phases and tasks are fixed unless scope changes.
Update PROGRESS.md each session — not this file.

---

## Phase 0 — Project Foundation
> Goal: repo ready, all reference docs in place, structure agreed.

- [x] Define project scope and architecture
- [x] Write CLAUDE.md
- [x] Write architecture.md
- [x] Write .claude/skills/fabric-medallion.md
- [x] Write PLAN.md and PROGRESS.md
- [ ] Write README.md (portfolio-facing summary)
- [ ] Create folder structure in repo
- [ ] Push initial commit to GitHub

---

## Phase 1 — Source Data
> Goal: four CSV files that deliberately exercise every modeling scenario.

- [x] Design student scenarios (10 students, each with a specific journey)
- [x] Create `data/raw/hubspot_leads.csv`
- [x] Create `data/raw/dynamics_applications.csv`
- [x] Create `data/raw/paradigm_enrolments.csv`
- [x] Create `data/raw/canvas_attendance.csv`
- [x] Validate: every fact table scenario covered by the data
- [x] Validate: SCD Type 2 trigger exists (Amir Hassan — domestic → international)
- [x] Validate: at-risk student exists (Linh Nguyen — all attended = 0)
- [ ] Upload CSVs to Fabric Lakehouse Files/raw/ section

---

## Phase 2 — Bronze Layer
> Goal: raw Delta tables in Fabric, append + date partitioned, with metadata columns.

- [x] Create `notebooks/01_bronze_ingest.ipynb`
- [ ] Import notebook into Fabric and attach Lakehouse
- [ ] Upload CSVs to Lakehouse Files/raw/
- [ ] Run notebook — verify four Bronze Delta tables created
- [ ] Verify: metadata columns present (_source_file, _source_system, _ingested_at, _ingestion_date)
- [ ] Verify: partitioned by (_source_system, _ingestion_date)
- [ ] Verify: row counts match source CSVs (10, 8, 6, 34)

---

## Phase 2.5 — Fabric Pipeline
> Goal: one pipeline that runs the full medallion stack end to end.
> Accepts pipeline_run_date as a parameter — drives Bronze partition and Silver MERGE.

- [ ] Create pipeline `pl_edu_analytics_full_load` in Fabric
- [ ] Add Copy Data activity — source: local/GitHub CSV, dest: Lakehouse Files/raw/
- [ ] Add Notebook activity: 01_bronze_ingest (pass pipeline_run_date parameter)
- [ ] Add Notebook activity: 02_silver_transform (pass pipeline_run_date parameter)
- [ ] Add Notebook activity: 03_gold_dimensions (pass pipeline_run_date parameter)
- [ ] Add Notebook activity: 04_gold_fact_pipeline (pass pipeline_run_date parameter)
- [ ] Add Notebook activity: 05_gold_fact_enrolment (pass pipeline_run_date parameter)
- [ ] Add Notebook activity: 06_gold_fact_attendance (pass pipeline_run_date parameter)
- [ ] Chain all activities with success dependencies
- [ ] Test: run pipeline end to end with initial dataset
- [ ] Test: add 2 new students to CSVs, re-run pipeline — verify MERGE correct

---

## Phase 3 — Silver Layer
> Goal: cleaned, conformed, MERGE-based staging tables with SCD Type 2.
> Reads only the Bronze partition matching pipeline_run_date — no duplicates on reruns.

- [ ] Create `notebooks/02_silver_transform.ipynb`
- [ ] Accept pipeline_run_date as notebook parameter
- [ ] Read Bronze filtered by _ingestion_date = pipeline_run_date
- [ ] Build silver.student — MERGE with SCD Type 2 on visa_code, is_international
- [ ] Build silver.application — MERGE on application_id
- [ ] Build silver.enrolment — MERGE on enrolment_id (net_fee_payable calculated)
- [ ] Build silver.attendance — MERGE on session_id
- [ ] Build silver.course, silver.campus, silver.intake — MERGE on natural keys
- [ ] Verify: student identity resolved across all three sources via student_email
- [ ] Verify: SCD Type 2 generates two rows for Amir Hassan after visa change run
- [ ] Verify: re-running same date produces no duplicates

---

## Phase 4 — Gold Dimensions
> Goal: all conformed dimension tables loaded, TBD row seeded in dim_date.

- [ ] Create `notebooks/03_gold_dimensions.ipynb`
- [ ] Accept pipeline_run_date as notebook parameter
- [ ] Build gold.dim_date (full date range + academic calendar columns)
- [ ] Seed dim_date TBD row (date_key = 0) and Unknown row (date_key = -1)
- [ ] Build gold.dim_student — MERGE from silver.student (all Type 2 versions)
- [ ] Build gold.dim_course — MERGE from silver.course
- [ ] Build gold.dim_campus — MERGE from silver.campus
- [ ] Build gold.dim_intake — MERGE from silver.intake
- [ ] Build gold.dim_lead_source — MERGE from silver distinct lead sources
- [ ] Build gold.dim_status — MERGE from reference values
- [ ] Verify: all surrogate keys are unique integers
- [ ] Verify: dim_date TBD row exists (date_key = 0)
- [ ] Verify: dim_student has two rows for Amir Hassan after visa change run

---

## Phase 5 — Gold Fact Tables
> Goal: all three fact types built and correct.

### fact_application_pipeline (Accumulating Snapshot)
- [ ] Create `notebooks/04_gold_fact_pipeline.ipynb`
- [ ] Accept pipeline_run_date as notebook parameter
- [ ] Build initial load — insert all lead rows, all milestones = 0 or NULL
- [ ] Implement Delta MERGE — update milestone columns as data arrives
- [ ] Calculate and store lag facts permanently when both dates known
- [ ] Verify: one row per application_id
- [ ] Verify: TBD date keys = 0 for unresolved milestones
- [ ] Verify: lag facts NULL when not yet calculable, integer when reached
- [ ] Verify: milestone flags correct for all 10 student scenarios

### fact_enrolment (Transaction)
- [ ] Create `notebooks/05_gold_fact_enrolment.ipynb`
- [ ] Accept pipeline_run_date as notebook parameter
- [ ] Build fact_enrolment — INSERT new rows only (transaction fact, never update)
- [ ] Verify: net_fee_payable = enrolment_fee - scholarship_amount
- [ ] Verify: is_withdrawal and is_completion flags correct

### fact_attendance (Factless)
- [ ] Create `notebooks/06_gold_fact_attendance.ipynb`
- [ ] Accept pipeline_run_date as notebook parameter
- [ ] Build fact_attendance_coverage — all scheduled slots (attended = 0 or 1)
- [ ] Build fact_attendance_actual — attended sessions only (attended = 1)
- [ ] Verify: Linh Nguyen has rows in coverage, zero rows in actual
- [ ] Verify: coverage row count > actual row count

---

## Phase 6 — Power BI
> Goal: Direct Lake semantic model with working reports.

- [ ] Connect Power BI to Fabric Lakehouse Gold schema (Direct Lake)
- [ ] Build semantic model — all fact → all dim relationships
- [ ] Set inactive relationships for role-playing date FKs in fact_application_pipeline
- [ ] Write DAX measures:
  - [ ] Conversion Rate = SUM(is_enrolled) / SUM(is_lead)
  - [ ] Avg Pipeline Days = AVERAGE(lead_to_enrolment_days)
  - [ ] Avg Days Application to Offer = AVERAGE(application_to_offer_days)
  - [ ] At-Risk Student Count
  - [ ] Total Net Revenue = SUM(net_fee_payable)
  - [ ] Withdrawal Rate = SUM(is_withdrawal) / COUNT(enrolment_sk)
- [ ] Build report page 1: Application Pipeline Funnel
- [ ] Build report page 2: Enrolment Revenue
- [ ] Build report page 3: At-Risk Students
- [ ] Take screenshots for portfolio

---

## Phase 7 — Portfolio Wrap-Up
> Goal: repo clean, README complete, ready to share.

- [ ] Write README.md (problem, architecture overview, key concepts demonstrated)
- [ ] Add architecture diagram image to docs/
- [ ] Add data model ERD to docs/
- [ ] Final GitHub push — all notebooks, CSVs, docs committed
- [ ] Verify repo is public and viewable
