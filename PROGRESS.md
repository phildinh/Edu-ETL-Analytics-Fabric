# PROGRESS.md — edu-analytics-fabric

Dynamic session log. Update this file at the start and end of every
Claude Code session. Never let progress live only in conversation history.

---

## Current Status

**Phase:** 2 — Bronze Layer (complete) / 2.5 — Pipeline (next)
**Last updated:** 2025-06-05
**Next action:** Update 01_bronze_ingest to append + date partition, then write pipeline + all remaining notebooks

---

## What Is Done

### Phase 0 — Project Foundation
- [x] Scope and architecture agreed — medallion architecture, 3 fact tables
- [x] CLAUDE.md written and trimmed (123 lines)
- [x] architecture.md written — full schemas, bus matrix, design decisions
- [x] .claude/skills/fabric-medallion.md written
- [x] PLAN.md written — all 7 phases with task-level detail
- [x] PROGRESS.md created

### Phase 1 — Source Data
- [x] 10 student scenarios designed and verified
- [x] hubspot_leads.csv created (10 rows)
- [x] dynamics_applications.csv created (8 rows — STU001/002 drop off)
- [x] paradigm_enrolments.csv created (6 rows — STU001-004 never enrol)
- [x] canvas_attendance.csv created (34 rows)
- [x] Validated: all scenarios covered (drop-off, rejection, withdrawal, at-risk, SCD Type 2, completion)

### Phase 2 — Bronze Layer
- [x] 01_bronze_ingest.ipynb created (overwrite mode — needs update to append + date partition)

---

## What Is Next

1. **Update `01_bronze_ingest.ipynb`** — change to append mode + `_ingestion_date` partition column
2. **Write `02_silver_transform.ipynb`** — MERGE pattern, reads Bronze by `pipeline_run_date`
3. **Write `03_gold_dimensions.ipynb`**
4. **Write `04_gold_fact_pipeline.ipynb`** — accumulating snapshot MERGE
5. **Write `05_gold_fact_enrolment.ipynb`**
6. **Write `06_gold_fact_attendance.ipynb`**
7. **Build Fabric pipeline `pl_edu_analytics_full_load`** — passes `pipeline_run_date` to all notebooks

---

## Key Decisions Made (Session Log)

| Date | Decision | Reason |
|---|---|---|
| 2025-06-05 | No dbt — PySpark notebooks only | Keeps focus on data modeling, reduces complexity |
| 2025-06-05 | Direct Lake over Import mode | Fabric-native, no refresh pipeline |
| 2025-06-05 | 3 fact tables: accumulating snapshot + transaction + factless | Covers all major Kimball fact types |
| 2025-06-05 | Drop periodic snapshot | Not demonstrable with static CSV data |
| 2025-06-05 | SCD Type 2 on visa_code + is_international only | Affects revenue and compliance historically |
| 2025-06-05 | Two attendance tables (coverage + actual) | Required for at-risk detection |
| 2025-06-05 | Lag facts stored as integers at ETL time | Power BI reads pre-computed integer, not DAX subtraction |
| 2025-06-05 | date_key = 0 for TBD milestones | FK columns must never be NULL — Kimball standard |
| 2025-06-05 | Bronze — append + _ingestion_date partition (not overwrite) | Full audit history, replay any date partition |
| 2025-06-05 | Silver reads Bronze by pipeline_run_date | Prevents duplicates on reruns, enables historical replay |
| 2025-06-05 | Full Fabric pipeline with pipeline_run_date parameter | One button runs full stack, parameter drives all notebooks |
| 2025-06-05 | Silver uses MERGE not overwrite | Cumulative — new students inserted, changed students updated, history preserved |

---

## Blockers / Open Questions

- 01_bronze_ingest.ipynb needs update: overwrite → append + _ingestion_date partition
- SCD Type 2 trigger for Amir Hassan: currently only one version in CSV.
  Plan: Silver notebook will simulate the visa change by checking Paradigm vs HubSpot visa mismatch.
  Need to confirm approach before writing Silver notebook.

---

## Session Template

Copy this block at the start of each new Claude Code session:

```
## Session — [DATE]

### Starting point
[What was done last session — copy from "What Is Next" above]

### Completed this session
- [ ] task 1
- [ ] task 2

### What is next
[Update the "What Is Next" section above after this session]

### Issues encountered
[Any errors, blockers, or decisions made]
```
