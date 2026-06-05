# CLAUDE.md — edu-analytics-fabric

## What this project is

A portfolio data engineering project demonstrating medallion architecture
(Bronze → Silver → Gold) on Microsoft Fabric Lakehouse, with a Kimball
dimensional model in the Gold layer queried by Power BI via Direct Lake.

Domain: fictional higher education provider tracking the full student lifecycle
from lead inquiry to enrolment across four source systems (HubSpot, Dynamics 365,
Paradigm SIS, Canvas LMS).

---

## How to start every session

1. Read `PROGRESS.md` — understand what is done and what is next
2. Read `PLAN.md` — confirm which phase and tasks are active
3. Read `architecture.md` — check the relevant schema before touching any table
4. Read `.claude/skills/fabric-medallion.md` before writing any code
5. At end of session — update `PROGRESS.md` with what was completed and what is next

---

## Project tracking files

| File | Purpose | Updated |
|---|---|---|
| `PLAN.md` | Fixed execution plan — 7 phases, all tasks with checkboxes | Only if scope changes |
| `PROGRESS.md` | Dynamic session log — status, decisions, blockers | Every session |
| `architecture.md` | Full schemas, bus matrix, design decisions | Only if design changes |
| `.claude/skills/fabric-medallion.md` | Code patterns and conventions for this stack | Only if patterns change |

---

## Stack

| Component | Tool |
|---|---|
| Lakehouse | Microsoft Fabric (trial workspace) |
| Storage | OneLake — Delta tables |
| Transformation | PySpark notebooks (no dbt) |
| Reporting | Power BI — Direct Lake mode |
| Source control | GitHub (manual copy-paste to Fabric) |
| Source data | Static CSV files → Lakehouse Files section |

---

## Repo structure

```
edu-analytics-fabric/
├── CLAUDE.md
├── PLAN.md                    ← execution plan — 7 phases
├── PROGRESS.md                ← session log — update every session
├── architecture.md            ← full schemas and design decisions
├── README.md
├── data/raw/
│   ├── hubspot_leads.csv
│   ├── dynamics_applications.csv
│   ├── paradigm_enrolments.csv
│   └── canvas_attendance.csv
├── notebooks/
│   ├── 01_bronze_ingest.ipynb
│   ├── 02_silver_transform.ipynb
│   ├── 03_gold_dimensions.ipynb
│   ├── 04_gold_fact_pipeline.ipynb
│   ├── 05_gold_fact_enrolment.ipynb
│   └── 06_gold_fact_attendance.ipynb
├── docs/
│   ├── data_model.md
│   └── business_questions.md
├── screenshots/powerbi/
└── .claude/skills/
    └── fabric-medallion.md
```

---

## Fact tables

| Notebook | Fact table | Kimball type | Pattern |
|---|---|---|---|
| 04 | fact_application_pipeline | Accumulating snapshot | Delta MERGE — insert + selective column update |
| 05 | fact_enrolment | Transaction | Insert only, never updated |
| 06 | fact_attendance_coverage + fact_attendance_actual | Factless | Insert only — two separate tables |

---

## Conformed dimensions (all built in notebook 03)

| Table | SCD type | Tracked attributes |
|---|---|---|
| dim_date | Static | date_key = 0 reserved for TBD milestones |
| dim_student | Type 2 | visa_code, is_international |
| dim_course | Type 1 | — |
| dim_campus | Type 1 | — |
| dim_intake | Type 1 | — |
| dim_lead_source | Type 1 | — |
| dim_status | Type 1 | — |

---

## Execution order

```
01_bronze → 02_silver → 03_gold_dimensions → 04, 05, 06 (any order after 03)
```

Never run a fact notebook before 03 completes.
Always seed dim_date TBD row (date_key = 0) at end of 03.

---

## Key rules

- Silver is wide and flat — not 3NF. See skill for why.
- Fact FK columns are never NULL — use 0 for TBD dates, -1 for unknown dims.
- Lag facts are integers stored at ETL time — never calculated in DAX.
- MERGE for accumulating snapshot updates milestone columns only — never full reload.
- Two attendance tables required — coverage + actual. Never collapse into one with a flag.
- Always check architecture.md for full field specs before generating any DDL or notebook code.
- Include markdown explanation cells in every notebook — this is a portfolio project.