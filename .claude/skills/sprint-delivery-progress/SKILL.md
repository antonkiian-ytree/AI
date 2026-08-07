---
name: sprint-delivery-progress
description: >
  Generate sprint delivery progress report for retro/standup. Use when the user says
  "sprint progress", "sprint delivery", "retro summary", "last sprint / next sprint",
  or asks for sprint-level delivery overview. Pulls ticket data from Jira sprints and
  produces a concise summary of completed work and upcoming focus areas.
---

# Sprint Delivery Progress Report

Produces a sprint delivery progress report with two sections: **Last Sprint** and **Current Sprint**.

## Input

Parse the user's request for:
- **Squad** — Jira project key (default: LUNA)
- **Last sprint name** — e.g., "LUNA 2026-Q3 Sprint 2" (if not given, detect from Jira)
- **Current sprint name** — e.g., "LUNA 2026-Q3 Sprint 3" (if not given, detect from Jira)

If sprint names are not provided, detect them:
1. Query open sprints: `project = {PROJECT} AND sprint in openSprints()` — the current sprint
2. Derive the previous sprint name by decrementing the sprint number (e.g., Sprint 3 → Sprint 2). If Sprint 1, go to previous quarter's last sprint.

### Sprint naming convention

`{PROJECT} {YYYY}-Q{N} Sprint {M}` — e.g., `LUNA 2026-Q3 Sprint 2`

## Data Collection

### Step 1: Fetch tickets for both sprints

Run two JQL queries in parallel:

**Last sprint:**
```
project = {PROJECT} AND sprint = "{LAST_SPRINT_NAME}" AND issuetype not in subtaskIssueTypes() ORDER BY status DESC
```

**Current sprint:**
```
project = {PROJECT} AND sprint = "{CURRENT_SPRINT_NAME}" AND issuetype not in subtaskIssueTypes() ORDER BY status DESC
```

Fields: `summary`, `status`, `assignee`, `issuetype`

### Step 2: Summarize

**Last Sprint — focus on what was delivered:**
- Count total tickets and done/released tickets
- Group completed work by theme/area (e.g., DMH, Performance, Billing, Platform, QA)
- Keep each theme to one line with key deliverables
- Include prod issues investigated/resolved

**Current Sprint — focus on what's planned:**
- Count total tickets
- List focus areas by theme, no per-ticket status
- Mention blocked items if any

## Output Format

```
**Last Sprint ({SPRINT_NAME}): {TOTAL} tickets — {DONE_COUNT} done/released**
- {Theme}: {key deliverables}
- {Theme}: {key deliverables}
- ...

**Current Sprint ({SPRINT_NAME}): {TOTAL} tickets**
- {Theme}: {planned work}
- {Theme}: {planned work}
- ...
```

### Rules

- Keep it concise — one line per theme, no individual ticket keys
- Group by domain/feature area, not by person
- Use plain text, no tables
- Do NOT include per-ticket status in the current sprint section
- Mention blocked items separately at the end if any exist
- If retro notes are provided by the user (what went well / what to improve), incorporate them into a **Key learnings and action items** section at the end
