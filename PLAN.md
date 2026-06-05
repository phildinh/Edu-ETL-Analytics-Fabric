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

- [ ] Design student scenarios (10 students, each with a specific journey)
- [ ] Create `data/raw/hubspot_leads.csv`
- [ ] Create `data/raw/dynamics_applications.csv`
- [ ] Create `data/raw/paradigm_enrolments.csv`
- [ ] Create `data/raw/canvas_attendance.csv`
- [ ] Validate: every fact table scenario is covered by the data
- [ ] Validate: SCD Type 2 trigger exists (student with visa change)
- [ ] Validate: at-risk student exists (enrolled, zero attendance rows)
- [ ] Upload CSVs to Fabric Lakehouse Files section

---

## Phase 2 — Bronze Layer
> Goal: raw Delta tables in Fabric, partitioned, with metadata columns.

- [ ] Create `notebooks/01_bronze_ingest.ipynb`
- [ ] Ingest hubspot_leads → bronze.hubspot_leads
- [ ] Ingest dynamics_applications → bronze.dynamics_applications
- [ ] Ingest paradigm_enrolments → bronze.paradigm_enrolments
- [ ] Ingest canvas_attendance → bronze.canvas_attendance
- [ ] Verify: all four Delta tables exist in Lakehouse
- [ ] Verify: metadata columns present (_source_file, _source_system, _ingested_at)
- [ ] Verify: partitioned by _source_system

---

## Phase 3 — Silver Layer
> Goal: cleaned, conformed, identity-resolved staging tables with SCD Type 2.

- [ ] Create `notebooks/02_silver_transform.ipynb`
- [ ] Build silver.student with SCD Type 2 logic
- [ ] Build silver.application (join HubSpot lead_id + Dynamics)
- [ ] Build silver.enrolment (net_fee_payable calculated)
- [ ] Build silver.attendance
- [ ] Build silver.course, silver.campus, silver.intake (reference tables)
- [ ] Verify: student identity resolved across all three sources
- [ ] Verify: SCD Type 2 generates two rows for the visa-change student
- [ ] Verify: no duplicate rows on natural keys

---

## Phase 4 — Gold Dimensions
> Goal: all conformed dimension tables loaded, TBD row seeded in dim_date.

- [ ] Create `notebooks/03_gold_dimensions.ipynb`
- [ ] Build gold.dim_date (full date range + academic calendar columns)
- [ ] Seed dim_date TBD row (date_key = 0)
- [ ] Build gold.dim_student (from silver.student, all Type 2 versions)
- [ ] Build gold.dim_course
- [ ] Build gold.dim_campus
- [ ] Build gold.dim_intake
- [ ] Build gold.dim_lead_source
- [ ] Build gold.dim_status
- [ ] Verify: all surrogate keys are unique integers
- [ ] Verify: dim_date TBD row exists (date_key = 0)
- [ ] Verify: dim_student has two rows for the visa-change student

---

## Phase 5 — Gold Fact Tables
> Goal: all three fact types built and correct.

### fact_application_pipeline (Accumulating Snapshot)
- [ ] Create `notebooks/04_gold_fact_pipeline.ipynb`
- [ ] Build initial load (lead milestone only — insert all rows)
- [ ] Implement Delta MERGE for milestone updates
- [ ] Verify: one row per application_id
- [ ] Verify: TBD rows (date_key = 0) for unresolved milestones
- [ ] Verify: lag facts NULL when milestone not reached, integer when reached
- [ ] Verify: milestone flags correct for each student scenario

### fact_enrolment (Transaction)
- [ ] Create `notebooks/05_gold_fact_enrolment.ipynb`
- [ ] Build fact_enrolment from silver.enrolment + dim lookups
- [ ] Verify: net_fee_payable = enrolment_fee - scholarship_amount
- [ ] Verify: is_withdrawal and is_completion flags correct

### fact_attendance (Factless)
- [ ] Create `notebooks/06_gold_fact_attendance.ipynb`
- [ ] Build fact_attendance_coverage (all scheduled slots)
- [ ] Build fact_attendance_actual (attended slots only)
- [ ] Verify: at-risk student has rows in coverage, zero in actual
- [ ] Verify: coverage row count > actual row count

---

## Phase 6 — Power BI
> Goal: Direct Lake semantic model with working reports.

- [ ] Connect Power BI to Fabric Lakehouse Gold schema (Direct Lake)
- [ ] Build semantic model relationships (all fact → all dims)
- [ ] Set inactive relationships for role-playing date FKs
- [ ] Write DAX measures:
  - [ ] Conversion Rate
  - [ ] Avg Pipeline Days
  - [ ] Avg Days Application to Offer
  - [ ] At-Risk Student Count
  - [ ] Total Net Revenue
  - [ ] Withdrawal Rate
- [ ] Build report page 1: Application Pipeline Funnel
- [ ] Build report page 2: Enrolment Revenue
- [ ] Build report page 3: At-Risk Students
- [ ] Take screenshots for portfolio

---

## Phase 7 — Portfolio Wrap-Up
> Goal: repo clean, README complete, ready to share.

- [ ] Write README.md (problem, architecture diagram, key concepts demonstrated)
- [ ] Add architecture diagram image to docs/
- [ ] Add data model ERD to docs/
- [ ] Final GitHub push — all notebooks, CSVs, docs committed
- [ ] Verify repo is public and viewable
