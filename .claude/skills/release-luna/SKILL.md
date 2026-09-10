---
name: release-luna
description: >
  Prepare a LUNA production release PR in argocd-monorepo. Use when the user says "prepare release
  LUNA-R-XXX", "release for LUNA", "promote to production", or provides services with tag bumps
  that include feature/bugfix PRs (not just automated deps). Handles Jira ticket linking, GitHub
  compare links, unrelated PR approval checklists, and hotfix tag scenarios.
---

# LUNA Production Release

Prepares a production release PR in argocd-monorepo by bumping image tags and creating a
fully-documented PR with Jira links, compare links, and approval checklists.

## Input

Parse the user's request for:
- **Services to promote** — service names with target tags
- **Jira release link or ID** — e.g., `LUNA-R-923` (optional)

### Before doing anything, determine the release type:

1. **If the user provides a Jira release ID or link** → this is a **versioned release** (`LUNA-R-{NNN}`)
2. **If the user says "no release ticket"** or does not provide one → **ask the user**:
   "Do we have a prepared Jira release for this? If yes, provide the release ID or link."
3. Based on the answer:
   - **Yes, there is a Jira release** → proceed as a versioned release (`LUNA-R-{NNN}`)
   - **No prepared release** → this is a **content-based release** — derive the release name from the
     content of the changes (e.g., `LUNA-RELEASE-FIX` for bug fixes, `LUNA-RELEASE-FEATURE` for features).
     Use `LUNA-RELEASE-FIX` as the default when the content is primarily fixes.

**Do NOT proceed to any preparation steps until the release type is determined.**

### After determining the release type, propose services and get confirmation:

**Do NOT assume which services to promote.** Instead:

1. Fetch the Jira tickets for the release (step 2 below)
2. Search for related PRs across LUNA-owned services by checking preprod vs production tag diffs
   and scanning commit messages/PR titles for the release ticket keys.
   **Also check `cm-frontend-server-versions`** — compare preprod vs production `app-values.yaml`
   for any `mf-*` version diffs, then scan the corresponding `mf-*` repo commits for release ticket keys
3. Present a summary to the user **before making any changes**:
   ```
   Release LUNA-R-{NNN} — proposed services:
   - ms-foo: 1.2.3 → 1.2.7 (contains LUNA-XXXX in PRs #42, #45)
   - ms-bar: 3.0.1 → 3.0.4 (contains LUNA-XXXX in PR #88)

   Shall I proceed with these, or add/remove any services?
   ```
4. Only proceed to tag bumps, commits, and PR creation **after the user confirms**.

## Pre-flight

Before making any changes in the argocd-monorepo, **always** switch to the main branch and pull latest:

```bash
git checkout main && git pull origin main
```

## Workflow

### 1. Determine the release ID

**For versioned releases (`LUNA-R-{NNN}`):**

The release ID **must** come from the Jira version name — never guess or increment from merged PRs.

When the user provides a Jira release link containing a version ID (e.g., `.../versions/34649/...`),
extract the version ID and fetch the version name via the Atlassian MCP:

```
JQL: fixVersion = {versionId}
Fields: ["fixVersions"]
MaxResults: 1
```

Read the `name` from the `fixVersions` array on the first returned issue (e.g., `LUNA-R-962`).
Use it exactly as-is.

**For content-based releases:**

Use the content-derived name (e.g., `LUNA-RELEASE-FIX`). No need to check merged PRs for numbering.

### 2. Fetch Jira release info

**Skip this step for content-based releases** (no Jira fixVersion exists).

**For versioned releases only:**

Query the Jira release report for linked tickets:

```
JQL: fixVersion = "LUNA-R-{NNN}"
```

Collect ticket keys, summaries, and links for the PR description.

**For each story ticket, also fetch its subtasks:**

```
JQL: parent IN (LUNA-XXXX, LUNA-YYYY) ORDER BY key ASC
```

Subtasks are listed under their parent in the "Promote to production" section (see body format below).

### 3. Verify and bump tags

```bash
git checkout main && git pull origin main
git checkout -b release/{RELEASE_ID}
# e.g., release/LUNA-R-924 or release/LUNA-RELEASE-FIX
```

For each service, verify current tags match expectations, then update
`environments/production/{service}/app-values.yaml` — change `image.tag` to the preprod value.

Verify with `git diff --stat`.

### 4. Gather compare links and PRs

For each service, get the PRs between the two tags:

```bash
gh api repos/y-tree-limited/{service}/compare/v{prod_tag}...v{preprod_tag} \
  --jq '.commits | map(.message | split("\n")[0]) | .[]'
```

Extract PR numbers from commit messages (`(#NNN)`), then get details:

```bash
gh pr view {number} --repo y-tree-limited/{service} \
  --json title,number,author,mergedAt,headRefName \
  --jq '"\(.number) | \(.title) | \(.author.login) | \(.mergedAt) | \(.headRefName)"'
```

**Order PRs by the date they were merged to the main branch (oldest first).** This is the canonical
ordering for Compare Changes — it reflects the sequence in which changes landed on main, regardless
of cherry-pick or hotfix timing.

Run compare gathering in parallel (use Agent tool) for speed.

### 5. Choose the minimum tag and classify PRs

**Promote to the minimum tag that includes all release-related PRs.** If unrelated PRs appear
after the last release PR in the tag sequence, use the earlier tag to exclude them. Only include
unrelated PRs when they are sandwiched between release PRs (i.e., they cannot be skipped).

Example — skip when possible:
```
v0.2.25 = #355 (LUNA-6498)  ← release
v0.2.26 = #356 (LUNA-6498)  ← release
v0.2.27 = #358 (LUNA-6498)  ← release ← promote to this tag
v0.2.28 = #359 (VG-6485)    ← unrelated, skip by using v0.2.27
```

Example — cannot skip:
```
v0.2.26 = #356 (LUNA-6498)  ← release
v0.2.27 = #359 (VG-6485)    ← unrelated, but sandwiched
v0.2.28 = #358 (LUNA-6498)  ← release ← must promote to v0.2.28, #359 included
```

For each PR in the compare range, classify and prefix with a marker:
- **🆕** — PR is part of this release (title or branch contains a Jira ticket key from the release)
- **🆕⚠️** — PR is unrelated but included in the release (sandwiched between release PRs, cannot be skipped)
- **✅** — PR is already released (only in hotfix → main tag scenarios, see below)

These markers apply to **all** releases (hotfix and non-hotfix alike).

Group 🆕⚠️ unrelated PRs by author for the approval checklist. 🆕 PRs never go under "Unrelated PRs approval".

### 6. Commit and push

```bash
git add <changed files>
git commit -m "$(cat <<'EOF'
{RELEASE_ID}

Co-Authored-By: Atlas <atlas@users.noreply.github.com>
EOF
)"
git push -u origin release/{RELEASE_ID}
```

### 7. Create PR

**Branch:** `release/{RELEASE_ID}` (no extra info — just the release ID)

**Title:** `{RELEASE_ID}`

**Body format for versioned releases (`LUNA-R-{NNN}`):**

```markdown
[LUNA-R-{NNN}](https://yofiworld.atlassian.net/projects/LUNA/versions/{versionId}/tab/release-report-all-issues)

# Release Description:

## Promote to production:
- [LUNA-XXXX](https://yofiworld.atlassian.net/browse/LUNA-XXXX) Ticket summary
  -- [LUNA-AAAA](https://yofiworld.atlassian.net/browse/LUNA-AAAA) Subtask summary
  -- [LUNA-BBBB](https://yofiworld.atlassian.net/browse/LUNA-BBBB) Subtask summary
- [LUNA-YYYY](https://yofiworld.atlassian.net/browse/LUNA-YYYY) Ticket summary
  -- [LUNA-CCCC](https://yofiworld.atlassian.net/browse/LUNA-CCCC) Subtask summary

## Compare Changes
- [{service}](https://github.com/y-tree-limited/{service}/compare/v{prod}...v{preprod})
  🆕 [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) `branch-name`
  PR title (release PR)
  🆕⚠️ [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) `branch-name`
  PR title (unrelated, sandwiched)

- [{service2}](...)
  ...

## Unrelated PRs approval:
- [ ] @author1 approved
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr}) `branch-name`
  PR title

- [ ] @author2 approved
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr}) `branch-name`
  PR title
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr}) `branch-name`
  PR title
```

**Body format for content-based releases (e.g., `LUNA-RELEASE-FIX`):**

```markdown
# Release Description:

## Promote to production:
- [LUNA-XXXX](https://yofiworld.atlassian.net/browse/LUNA-XXXX) Ticket summary
- [LUNA-YYYY](https://yofiworld.atlassian.net/browse/LUNA-YYYY) Ticket summary

## Compare Changes
- [{service}](https://github.com/y-tree-limited/{service}/compare/v{prod}...v{preprod})
  🆕 [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) `branch-name`
  PR title

## Unrelated PRs approval:
...
```

Note: content-based releases have no Jira release link at the top. The "Promote to production"
section lists tickets found in the PR titles/branches. No subtask fetching is needed.

Also add unrelated PR authors as **GitHub reviewers**:

```bash
gh pr edit {pr_number} --add-reviewer author1,author2
```

## Sections to omit when not applicable

- **Unrelated PRs approval** — only include if there are PRs not linked to a release Jira ticket
- **Hotfix section** — only include when hotfix tags are involved (see below)
- **"Changes include" checklist** — never include, it adds no value

## Hotfix tag handling

### Hotfix → main tag (e.g. `117.1.0-hotfix-...` → `117.3.0`)

When production runs a hotfix tag and the new tag is a main-branch tag:

1. Compare hotfix base to hotfix tag (`v117.1.0...v117.1.0-hotfix-...`) → PRs already released
2. Compare hotfix base to new tag (`v117.1.0...v117.3.0`) → PRs on main
3. Cross-reference and prefix each PR in Compare Changes:
   - PRs in both = ✅ already released
   - PRs only on main AND part of this release = 🆕 new in this release
   - PRs only on main AND NOT part of this release = 🆕⚠️ unrelated (sandwiched)
4. ✅ PRs do NOT go under "Unrelated PRs approval" — only 🆕⚠️ unrelated PRs need approval
5. 🆕 PRs do NOT go under "Unrelated PRs approval" — they are the reason for the release
6. Check for hotfix-only PRs by comparing preprod tag vs production hotfix tag directly
   (`v{preprod_tag}...v{hotfix_tag}`). Flag lost PRs with ⚠️ note including each PR's title
7. When a PR and its revert are both in range, list as separate lines (both 🆕⚠️).
   Revert line: `[repo@short-sha](commit-url)` with title `Revert <original title>`
8. Use full compare link `hotfix-tag...new-tag` for the service compare URL

### Hotfix → hotfix (e.g. `4.27.10-hotfix-...T09` → `4.27.10-hotfix-...T15`)

When promoting from one hotfix to another on the same base:

1. Compare old hotfix tag to new hotfix tag directly
2. Do NOT add ✅/🆕/🆕⚠️ marks — all PRs are new
3. Do NOT add "Hotfix-only PRs not in preprod" warning
4. List PRs plainly in the Compare Changes sub-list

### 8. Prepare Slack draft

After the PR is created, prepare a draft message in **#squad_luna-release** (`C0679L7F59A`):

```
@luna :large_green_circle: RELEASE
<{jira_release_url}|{RELEASE_ID}>
please review {pr_url}
```

- For **versioned releases**: `{RELEASE_ID}` links to the Jira release page
- For **content-based releases**: `{RELEASE_ID}` is plain text (no Jira link)
- Use Slack's native link format: `<url|text>`
- Use `slack_send_message_draft` tool to create the draft (not send)

## Notes

- **Working directory:** use `repos/argocd-monorepo` (relative to project root) for all argocd-monorepo operations. If it doesn't exist, fall back to `/Users/antonkiian/projects/ytree/argocd-monorepo`
- **Tag formats differ by context:**
  - **Source repos** (GitHub tags, compare links): use `v` prefix (e.g., `v130.0.33`)
  - **Production `app-values.yaml`**: use **no** `v` prefix (e.g., `tag: 130.0.33`). When bumping the production image tag, strip the `v` prefix from the preprod/source tag.
- PR title is just the release ID (e.g., `LUNA-R-923` or `LUNA-RELEASE-FIX`) — not a conventional commit message
- Jira tickets listed under "Promote to production" have no "fixes" prefix — just ticket link and summary
- Jira cloud ID: `3e0004f3-0133-4a66-85f9-a2ce7b5ba86d` (yofiworld.atlassian.net)
