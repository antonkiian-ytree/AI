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
- **Release ID** — e.g., `LUNA-R-923` (from Jira release)
- **Services to promote** — service names with preprod and production tags

If missing, check recent Jira releases or ask the user.

## Workflow

### 1. Determine the release ID

Find the next release number by checking recent merged release PRs:

```bash
gh pr list --state merged --limit 5 --search "LUNA-R" --json number,title
```

If the user provides a release ID, use it. Otherwise, increment from the latest.

### 2. Fetch Jira release info

Query the Jira release report for linked tickets:

```
JQL: fixVersion = "LUNA-R-{NNN}"
```

Collect ticket keys, summaries, and links for the PR description.

### 3. Verify and bump tags

```bash
git checkout main && git pull origin main
git checkout -b release/LUNA-R-{NNN}
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
  --json title,number,author,mergedAt \
  --jq '"\(.number) | \(.title) | \(.author.login) | \(.mergedAt)"'
```

**Order PRs by merged timestamp (oldest first).**

Run compare gathering in parallel (use Agent tool) for speed.

### 5. Classify PRs

For each PR in the compare range:
- **Related to release** — PR title contains a Jira ticket key from the release
- **Unrelated** — everything else (deps updates, refactors, etc.)

Group unrelated PRs by author for the approval checklist.

### 6. Commit and push

```bash
git add <changed files>
git commit -m "LUNA-R-{NNN}"
git push -u origin release/LUNA-R-{NNN}
```

### 7. Create PR

**Branch:** `release/LUNA-R-{NNN}` (no extra info — just the release ID)

**Title:** `LUNA-R-{NNN}`

**Body format:**

```markdown
[LUNA-R-{NNN}](https://yofiworld.atlassian.net/projects/LUNA/versions/{versionId}/tab/release-report-all-issues)

# Release Description:

## Promote to production:
- [LUNA-XXXX](https://yofiworld.atlassian.net/browse/LUNA-XXXX) Ticket summary
- [LUNA-YYYY](https://yofiworld.atlassian.net/browse/LUNA-YYYY) Ticket summary

## Compare Changes
- [{service}](https://github.com/y-tree-limited/{service}/compare/v{prod}...v{preprod})
  [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) PR title
  [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) PR title

- [{service2}](...)
  ...

## Unrelated PRs approval:
- [ ] @author1 approved
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr})

- [ ] @author2 approved
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr})
  [{repo}#{pr}](https://github.com/y-tree-limited/{repo}/pull/{pr})
```

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
3. Cross-reference:
   - PRs in both = ✅ already released
   - PRs only on main = ❌ not released
4. Prefix each PR in Compare Changes with ✅ or ❌
5. ✅ PRs do NOT go under "Unrelated PRs approval" — only ❌ unrelated PRs need approval
6. Check for hotfix-only PRs by comparing preprod tag vs production hotfix tag directly
   (`v{preprod_tag}...v{hotfix_tag}`). Flag lost PRs with ⚠️ note including each PR's title
7. When a PR and its revert are both in range, list as separate lines (both ❌).
   Revert line: `[repo@short-sha](commit-url)` with title `Revert <original title>`
8. Use full compare link `hotfix-tag...new-tag` for the service compare URL

### Hotfix → hotfix (e.g. `4.27.10-hotfix-...T09` → `4.27.10-hotfix-...T15`)

When promoting from one hotfix to another on the same base:

1. Compare old hotfix tag to new hotfix tag directly
2. Do NOT add ✅/❌ marks — all PRs are new
3. Do NOT add "Hotfix-only PRs not in preprod" warning
4. List PRs plainly in the Compare Changes sub-list

## Notes

- Tags in source repos use `v` prefix (e.g., `v130.0.33`)
- PR title is just the release ID (e.g., `LUNA-R-923`) — not a conventional commit message
- Jira tickets listed under "Promote to production" have no "fixes" prefix — just ticket link and summary
- Jira cloud ID: `3e0004f3-0133-4a66-85f9-a2ce7b5ba86d` (yofiworld.atlassian.net)
