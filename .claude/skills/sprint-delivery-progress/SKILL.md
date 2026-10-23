---
name: sprint-delivery-progress
description: >
  Generate sprint delivery progress report for retro/standup. Use when the user says
  "sprint progress", "sprint delivery", "retro summary", "last sprint / next sprint",
  or asks for sprint-level delivery overview. Pulls ticket data from Jira sprints and
  produces a concise summary of completed work and upcoming focus areas.
---

# Sprint Delivery Progress Report

Produces a sprint delivery progress report with two sections: **Previous Sprint** and **Next Sprint**.

## Input

Parse the user's request for:
- **Squad** — Jira project key (default: LUNA)
- **Previous sprint name** — e.g., "LUNA 2026-Q3 Sprint 2" (if not given, detect from Jira)
- **Next sprint name** — e.g., "LUNA 2026-Q3 Sprint 3" (if not given, detect from Jira — the current open sprint)

If sprint names are not provided, detect them:
1. Query open sprints: `project = {PROJECT} AND sprint in openSprints()` — the current sprint
2. Derive the previous sprint name by decrementing the sprint number (e.g., Sprint 3 → Sprint 2). If Sprint 1, go to previous quarter's last sprint.

### Sprint naming convention

`{PROJECT} {YYYY}-Q{N} Sprint {M}` — e.g., `LUNA 2026-Q3 Sprint 2`

## Data Collection

### Step 1: Fetch tickets for both sprints

Run two JQL queries in parallel:

**Previous sprint:**
```
project = {PROJECT} AND sprint = "{LAST_SPRINT_NAME}" AND issuetype not in subtaskIssueTypes() ORDER BY status DESC
```

**Next sprint:**
```
project = {PROJECT} AND sprint = "{CURRENT_SPRINT_NAME}" AND issuetype not in subtaskIssueTypes() ORDER BY status DESC
```

Fields: `summary`, `status`, `assignee`, `issuetype`

### Step 2: Count live releases for the last sprint

Count released Jira fixVersions with `releaseDate` within the last sprint's date range.

Sprint date range: `[startDate, startDate + 13 days]` (sprints are 14 calendar days).

Query:
```
project = {PROJECT} AND fixVersion in releasedVersions() AND issuetype not in subtaskIssueTypes() ORDER BY fixVersion DESC
```
Fields: `fixVersions`

Extract unique fixVersion names where `releaseDate` falls within the sprint range.

### Step 3: Count live bugs (production issues) for the last sprint

Identify production bugs in the last sprint by matching tickets whose summary contains `[Production issue]` or `[production issue]` prefix, or whose parent epic is related to production issues.
- **Only count tickets with Jira issue type = Bug** — exclude Investigations, Tasks, etc.
- **Only count tickets created during the sprint period** — exclude carry-overs from earlier sprints
- **Found** = total such Bug tickets created in the sprint period
- **Solved** = those in a done status (Completed, Released, Closed)
- Collect Jira links for each bug to include in the report

### Step 4: Count support escalations from Slack #production_issues

Search Slack `#production_issues` channel (`CA0P0BJQ7`) for messages from LUNA squad members during the sprint period.

Search for each squad member individually:
- `from:<member> in:<#CA0P0BJQ7> after:{START_DATE} before:{END_DATE+1}`

Count distinct threads where LUNA squad members responded. Summarise each thread briefly (topic + who responded).

### Step 5: Summarize

**Previous Sprint — focus on what was delivered:**
- Count total tickets and done/released tickets
- Count live releases (from Step 2)
- Count live bugs found/solved (from Step 3)
- Group completed work by theme/area (e.g., DMH, Performance, Billing, Platform, QA)
- Keep each theme to one line with key deliverables
- Include prod issues investigated/resolved

**Next Sprint — focus on what's planned:**
- Count total tickets
- List focus areas by theme, no per-ticket status
- Mention blocked items if any

## Output Format

```
**Previous Sprint ({SPRINT_NAME}): {TOTAL} tickets — {DONE_COUNT} done/released — {RELEASE_COUNT} live releases**


- {Theme}: {key deliverables}
- {Theme}: {key deliverables}
- ...

**Bugs (Live) ({SPRINT_NAME}): {FOUND} found / {SOLVED} solved**

- [{TICKET_KEY}](link) {status}
- ... (or "none")

**Support escalations (#production_issues) ({SPRINT_NAME}): {COUNT} threads**

- {brief topic (who responded)}
- ...

**Next Sprint ({SPRINT_NAME}): {TOTAL} tickets**

- {Theme}: {planned work}
- {Theme}: {planned work}
- ...

**Key learnings and action items**

- {concise learning — one sentence}
- {concise learning — one sentence}
- Action: {improvement item}
```

### Rules

- Keep it concise — one line per theme, no individual ticket keys
- Group by domain/feature area, not by person
- Use plain text, no tables
- Do NOT include per-ticket status in the current sprint section
- Mention blocked items separately at the end if any exist
- If retro notes are provided by the user (what went well / what to improve), incorporate them into a **Key learnings and action items** section at the end. Rewrite the raw input as very short bullet points — max 6–8 words each, must fit in a presentation slide. Merge related items. Prefix improvement items with "Action:". Do not split into "went well" / "to improve" sub-sections
