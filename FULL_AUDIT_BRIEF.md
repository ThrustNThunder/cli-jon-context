# Full Document + Repo Audit
**Date:** May 11, 2026
**From:** Jon (ThunderBase)
**To:** CLI Jon
**Task:** Full audit of all workspace documents and repos — find duplicates, gaps, stale content, inconsistencies. Produce a clean organized report and fix what you can.
**Mode:** Audit + fix. Write changes directly. No push unless explicitly stated.
**Time:** Take as long as you need. This is important.

---

## What to Audit

### 1. `/home/ubuntu/.openclaw/workspace/project_jon/` — Primary Docs

Read every file. For each one:
- What is it? Is it current?
- Does it duplicate content from another file?
- Is it referenced anywhere or orphaned?
- Should it be merged, archived, or kept as-is?

Known files to check:
- THUNDERGATE_DESIGN_PRINCIPLES.md (23 principles — is it current?)
- THUNDERGATE_RUNTIME_SPEC.md
- THUNDERGATE_OPEN_QUESTIONS.md
- THUNDERCOMMO_ROADMAP.md (is it current through Build 28?)
- THUNDERCOMMO_IOS_DESIGN_BRIEF.md
- THUNDERCOMM_MASTER.md, THUNDERCOMM_ARCHITECTURE.md, THUNDERCOMM_STATE_MODEL.md, THUNDERCOMM_SETUP.md, THUNDERCOMM_FEDERATION.md, THUNDERCOMM_IOS_HANDOFF.md (are these all still relevant or superseded?)
- THUNDERBROWSER.md, THUNDERBROWSER_EXTENSION_DESIGN.md, THUNDERBROWSER_PHASE01_TICKETS.md
- THUNDERAGENT_BUSINESS.md, THUNDEROS_SPEC.md, THUNDEROS_NOTES.md
- BYOAA_INTEGRATION.md, BYOAA_PARTNERSHIP_AGREEMENT_FRAMEWORK.md
- THUNDERGATE_CLAUDE_CODE_VISION.md (just written today)
- THUNDEROS_ARCHITECTURE_MANIFESTO.md (just written today)
- All PATENT_* files
- Any others

### 2. `/home/ubuntu/.openclaw/workspace/` root

Check for orphaned docs, stale briefs, build docs that should be in project_jon/:
- build28-ios-patch.md, build28b-ios-patch.md
- cli-jon-brief-build28.md, cli-jon-brief-build28b.md
- Any others

### 3. `/home/ubuntu/.openclaw/workspace/scripts/`

Check for:
- Duplicate scripts (aa_reserve.py vs aa_reserve_v1.8.py vs aa_jumpseat_reserve.py — what's the canonical one?)
- Scripts that are no longer needed
- Scripts missing documentation

### 4. `/tmp/cli-jon-context/` repo

Check for:
- Duplicate briefs (multiple Build 28 briefs?)
- Stale task files that are done and should be archived
- ACTIVE_TASKS.md — is it current?
- Any briefs that should be committed to the main workspace

### 5. Cross-reference check

Are these pairs consistent with each other?
- THUNDERGATE_DESIGN_PRINCIPLES.md vs THUNDERGATE_RUNTIME_SPEC.md
- THUNDERCOMMO_ROADMAP.md vs build28-ios-patch.md
- BYOAA_INTEGRATION.md vs BYOAA_PARTNERSHIP_AGREEMENT_FRAMEWORK.md
- THUNDERBROWSER.md vs THUNDERBROWSER_EXTENSION_DESIGN.md
- THUNDERGATE_CLAUDE_CODE_VISION.md vs THUNDEROS_ARCHITECTURE_MANIFESTO.md (just written — are they consistent or do they conflict?)

---

## What to Produce

### Report: `/tmp/full-audit-report.md`

For each file/category:
- Status: CURRENT | STALE | DUPLICATE | ORPHANED | NEEDS UPDATE
- Recommendation: KEEP | MERGE INTO X | ARCHIVE | DELETE | UPDATE

### Fixes to make directly:

1. **THUNDERCOMMO_ROADMAP.md** — update to reflect current build state (Build 27 live, Build 28 staged with 8 bugs + APNs + Bug #9)

2. **THUNDERGATE_DESIGN_PRINCIPLES.md** — verify all 23 principles are accurate. Add Claude Code native as a principle if it's missing.

3. **Merge stale ThunderCommo docs** — THUNDERCOMM_MASTER.md, THUNDERCOMM_ARCHITECTURE.md, THUNDERCOMM_STATE_MODEL.md, THUNDERCOMM_SETUP.md — if superseded by the roadmap and IOS design brief, archive them to `project_jon/archive/`

4. **Root-level build docs** — move build28-ios-patch.md, build28b-ios-patch.md, cli-jon-brief-*.md to `project_jon/builds/` directory

5. **Scripts** — document which aa_reserve script is canonical, archive others with clear naming

6. **ACTIVE_TASKS.md in cli-jon-context** — ensure it's fully current with tonight's work

---

## Rules
1. Read everything before making recommendations
2. Don't delete anything — move to archive/ if stale
3. Write changes directly
4. Commit workspace changes: `cd /home/ubuntu/.openclaw/workspace && git add -A && git commit -m "Full doc audit: cleanup, consolidation, archive stale"`
5. Update ACTIVE_TASKS.md
6. Write completion to `/tmp/full-audit-complete.txt`
7. Take the time you need — this is the foundation everything else builds on
