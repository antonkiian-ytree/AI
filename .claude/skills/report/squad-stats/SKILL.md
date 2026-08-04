---
name: squad-stats
description: >
  Prepare monthly Tech Delivery and Tech Operations stats for a squad. Use when the user says
  "squad stats for LUNA July", "prepare monthly stats", "tech delivery report", "squad predictability",
  or asks for any monthly engineering metrics report. Collects: Squad Predictability, Squad Releases,
  Cycle Time, Tech OKRs completion, Operational Standards, Tech Debt, Production Issues, and
  Critical/High Vulnerabilities.
---

# Squad Stats — Monthly Tech Report

Produces a monthly stats report matching this template:

```
Tech Delivery Stats (Application Development)
Squad Predictability - X%
Squad Releases - N
Cycle time - N days
Tech OKRs completion - X%

Tech Operations Stats
Operational Standards - % of domain scorecard deployed - X% (N out of M)
App Dev. Technical Debt (total number of tickets open and closed in month)
Total number of open technical debt tickets - N
Closed tech debts - N

App Dev: Production issues total - N
% of production issues caused by software bugs and total number of issues - X% (N)
% of production issues raised as questions and product issues - X% (N)

Critical and high vulnerabilities:
Critical - N
High - N
```

## Input

Parse the user's request for:
- **Squad** — Jira project key (e.g., LUNA, VEGA)
- **Month/Year** — reporting period (e.g., July 2026)

If missing, ask.

## Metric Collection

Work through each metric **one at a time**, confirming values with the user before moving on.

### 1. Squad Predictability

**Source:** Jira Velocity Report (user provides screenshot or data).

**Calculation — weighted by calendar overlap with the reporting month:**

1. Identify all sprints that overlap the reporting month
2. For each sprint, calculate:
   - **Days in month** — how many days of that sprint fall within the reporting month
   - **Weight** — `days_in_month / total_days_in_month` (e.g., for July: `days / 31`)
   - **Sprint predictability** — `completed_points / committed_points × 100%`
3. **Weighted predictability** — `Σ (weight × sprint_predictability)` across all overlapping sprints

**Example (July = 31 days):**

| Sprint      | Sprint dates    | Days in July | Weight | Committed | Completed | Predictability |
|-------------|-----------------|--------------|--------|-----------|-----------|----------------|
| Q2 Sprint 7 | Jun 22 – Jul 6  | 6            | 6/31   | 154.5     | 152       | 98.4%          |
| Q3 Sprint 1 | Jul 6 – Jul 20  | 14           | 14/31  | 149       | 111       | 74.5%          |
| Q3 Sprint 2 | Jul 20 – Jul 31 | 11           | 11/31  | 157       | 143.5     | 91.4%          |

Weighted = `(6/31 × 98.4) + (14/31 × 74.5) + (11/31 × 91.4)` = **85%**

Round to nearest whole percent.

### 2. Squad Releases

**Source:** Jira Releases page for the project.

**JQL to count unique fixVersions released in the month:**
```
project = {PROJECT} AND fixVersion in releasedVersions() AND resolved >= "{YYYY-MM-01}" AND resolved <= "{YYYY-MM-DD}"
```

Count the **unique fixVersion names** from the results.

### 3. Cycle Time

**Source:** Jira issue changelogs + GitHub PR creation dates, via direct REST API.

**Auth:** Use the Jira API token stored in env var `JIRA_API_TOKEN` with the user's Atlassian email. Use `curl` with basic auth (`email:token`) against `https://yofiworld.atlassian.net/rest/api/3`.

**Important:** The old `/rest/api/3/search` endpoint is deprecated. Use `POST /rest/api/3/search/jql` with JSON body `{"jql":"...","fields":[...],"maxResults":N}`.

**Definition:** Time from **first PR created** for a ticket to **Done status transition** (calendar days). If no PR exists, fall back to **In Progress → Done** status transitions. If neither exists, skip the issue.

**Steps:**

1. **Get issues resolved in the month:**
   ```
   JQL: project = {PROJECT} AND statusCategory = Done AND resolved >= "{YYYY-MM-01}" AND resolved <= "{YYYY-MM-DD}" AND issuetype in (Story, Task, Bug)
   Fields: key, customfield_10211 (Story point estimate)
   ```

2. **Discover the squad's "Done" statuses:** Fetch transitions for one resolved issue (`GET /rest/api/3/issue/{key}/changelog`) and identify statuses in the `Done` status category. Common per-squad mappings:
   - LUNA: Completed, RELEASED, Closed
   - Each squad may differ — always check first

3. **For each issue, determine start and end dates:**
   - **End date:** Last changelog entry transitioning to a Done status (`GET /rest/api/3/issue/{key}/changelog`)
   - **Start date (priority order):**
     1. **GitHub PR:** `gh search prs "{TICKET_KEY}" --owner y-tree-limited --json createdAt,title` → filter results where title contains the ticket key → use earliest `createdAt`
     2. **Status fallback:** First changelog entry transitioning to "In Progress"
     3. **Skip** if neither found

4. **Calculate cycle time:** `end_date - start_date` in calendar days

5. **Report the median** as the headline metric (robust against outliers).

**Output:** `Cycle time - N days` (median, calendar days)

### 4. Tech OKRs Completion

**Source:** Jira — all tickets attached to current quarter sprints.

**Steps:**

1. Query all issues in the current quarter's sprints:
   ```
   JQL: project = {PROJECT} AND sprint in ("{PROJECT} {YYYY}-Q{N} Sprint 1", "{PROJECT} {YYYY}-Q{N} Sprint 2", ...) AND issuetype in (Story, Task, Bug)
   Fields: key, status, customfield_10211 (Story point estimate)
   ```

2. Calculate completion **both by tickets and by story points**:
   - By tickets: `done_count / total_count × 100%`
   - By SP: `done_sp / total_sp × 100%`

3. List the not-done tickets for context.

**Output:** `Tech OKRs completion - X% (by tickets) / Y% (by story points)`

### 5. Operational Standards (Domain Scorecard)

**Source:** Ask the user — domain scorecard deployment percentage. This value changes slowly.

**Important:** Check memory for the previous month's value and ask: "Last time Operational Standards was X% — should I use the same value or has it changed?"

Format: `X%`

### 6. Tech Debt

**Source:** Jira — tickets with label `tech-debt` in the squad's project.

**Steps:**

1. **Open tech debt tickets** (not completed):
   ```
   JQL: project = {PROJECT} AND labels = tech-debt AND statusCategory != Done
   ```

2. **Closed tech debts in the month:**
   ```
   JQL: project = {PROJECT} AND labels = tech-debt AND statusCategory = Done AND resolved >= "{YYYY-MM-01}" AND resolved <= "{YYYY-MM-DD}"
   ```

**Output:**
```
App Dev. Technical Debt (total number of tickets open and closed in month)
Total number of open technical debt tickets - N
Closed tech debts - N
```

### 7. Production Issues

**Source:** Slack `#production_issues` channel + Jira ticket types.

**Steps:**

1. **Find the squad's Slack user group ID** from memory or by searching Slack channels. Known IDs:
   - squad-luna: `S01D3S40B4L`

2. **Read `#production_issues` channel** (channel ID: `CA0P0BJQ7`) for the reporting month. Count all structured "Y TREE:" messages where `<!subteam^{SQUAD_ID}>` is tagged in the `*Squads:*` field.

3. **For each issue, search for linked Jira tickets** in the thread (look for `{PROJECT}-XXXX` links). Also search Jira for tickets with `[Production issue]` or `[Production-issue]` in summary created during the month:
   ```
   JQL: project = {PROJECT} AND (summary ~ "Production issue" OR summary ~ "production issue" OR issuetype = Bug) AND created >= "{YYYY-MM-01}" AND created <= "{YYYY-MM-DD}"
   ```

4. **Classify each issue:**
   - **Has Jira ticket → check issue type AND status:**
     - Status = Completed or RELEASED → **software bug** (regardless of type — Bug, Task, Investigation)
     - Status = Closed → **investigation** (exclude from total — investigated and dismissed)
   - **No Jira ticket → question/product issue**

5. **Calculate:**
   - Total = all issues minus investigations (Closed tickets)
   - Software bugs = count and percentage
   - Questions/product issues = count and percentage

**Output:**
```
App Dev: Production issues total - N
% of production issues caused by software bugs and total number of issues - X% (N)
% of production issues raised as questions and product issues - X% (N)
```

### 8. Critical and High Vulnerabilities

**Source:** Grafana Application Security dashboard.

**Dashboard:** `{GRAFANA_BASE}/d/application-security/application-security`

**Filters:**
- `var-metrics_datasource` = metrics datasource UID
- `var-namespace` = `business-application`
- `var-squad` = squad name (e.g., `luna`)
- `var-service` = `$__all`

Use the Grafana skill to query metrics, or ask the user to open the dashboard with the squad filter and share the counts.

**Output:**
```
Critical and high vulnerabilities:
Critical - N
High - N (note if fixed on PreProd but not yet delivered)
```

## Output

Once all metrics are collected, produce the final report in the template format above.