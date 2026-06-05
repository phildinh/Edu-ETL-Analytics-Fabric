# PROGRESS.md — edu-analytics-fabric

Dynamic session log. Update this file at the start and end of every
Claude Code session. Never let progress live only in conversation history.

---

## Current Status

**Phase:** 0 — Project Foundation
**Last updated:** 2025-06-05
**Next action:** Write README.md, create folder structure, push to GitHub

---

## What Is Done

### Phase 0 — Project Foundation
- [x] Scope and architecture agreed — medallion architecture, 3 fact tables
- [x] CLAUDE.md written (101 lines — within limit)
- [x] architecture.md written — full schemas, bus matrix, design decisions
- [x] .claude/skills/fabric-medallion.md written
- [x] PLAN.md written — all 7 phases with task-level detail
- [x] PROGRESS.md created

---

## What Is Next

1. Write `README.md` — portfolio-facing summary of the project
2. Run PowerShell commands to create folder structure
3. Copy all docs into repo
4. First GitHub push

Then move to **Phase 1 — Source Data**: design and generate the four CSV files.

---

## Key Decisions Made (Session Log)

| Date | Decision | Reason |
|---|---|---|
| 2025-06-05 | No dbt — PySpark notebooks only | Keeps focus on data modeling, reduces complexity |
| 2025-06-05 | Direct Lake over Import mode | Fabric-native, no refresh pipeline, demonstrates modern Fabric architecture |
| 2025-06-05 | 3 fact tables: accumulating snapshot + transaction + factless | Covers all major Kimball fact types in one demo |
| 2025-06-05 | Drop periodic snapshot | Requires scheduled job — not demonstrable with static CSV data |
| 2025-06-05 | SCD Type 2 on visa_code + is_international only | Minimal scope, maximum historical significance for reporting |
| 2025-06-05 | Two attendance tables (coverage + actual) | Required for at-risk detection — one table cannot answer absence queries |
| 2025-06-05 | Lag facts stored as integers at ETL time | Performance — Power BI reads pre-computed integer, not DAX subtraction |
| 2025-06-05 | date_key = 0 for TBD milestones | FK columns must never be NULL — Kimball standard |

---

## Blockers / Open Questions

None currently.

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
