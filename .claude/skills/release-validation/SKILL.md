---
name: release-validation
description: >
  Validate an argocd-monorepo release PR — verify tags exist, compare ranges match the PR description,
  and all commits are accounted for. Use when the user says "validate release", "validate changes",
  "check release PR", or provides an argocd-monorepo PR URL/number to validate.
---

# Release Validation

Validates an argocd-monorepo release PR by checking that all tag bumps are correct, all target tags
exist on GitHub, and every commit in the compare ranges is accounted for in the PR description.

## Input

Parse the user's request for:
- **PR number or URL** — e.g., `42871` or `https://github.com/y-tree-limited/argocd-monorepo/pull/42871`

## Workflow

### 1. Fetch PR details

```bash
gh pr view {number} --repo y-tree-limited/argocd-monorepo \
  --json title,body,headRefName,baseRefName,state,files
```

Extract from the PR:
- Services being bumped (from changed files)
- The PR body (tickets, compare links, intermediate releases)

### 2. Get the diff

```bash
gh pr diff {number} --repo y-tree-limited/argocd-monorepo
```

For each changed `app-values.yaml`, extract:
- **Service name** — from the file path (`environments/production/{service}/app-values.yaml`)
- **Old tag** — the `-` line value
- **New tag** — the `+` line value

### 3. Verify tags exist

For each service, confirm the target tag exists on GitHub:

```bash
gh api repos/y-tree-limited/{service}/git/refs/tags/v{new_tag} --jq '.ref'
```

Remember: source repo tags use the `v` prefix, but `app-values.yaml` stores tags without the `v` prefix.

### 4. Compare ranges and collect commits

For each service, fetch all commits between old and new tags:

```bash
gh api repos/y-tree-limited/{service}/compare/v{old_tag}...v{new_tag} \
  --jq '[.commits[] | {sha: .sha[:7], msg: .commit.message | split("\n")[0]}]'
```

Run these in parallel for speed.

### 5. Cross-reference commits against PR description

For each commit in every compare range, verify it is accounted for in the PR body:
- **Release tickets** — commit message contains a Jira ticket key listed in the PR's tickets section
- **Intermediate/unrelated** — commit is mentioned in the PR's intermediate releases or unrelated sections
- **Unaccounted** — flag any commit not mentioned anywhere in the PR description

### 6. Output validation report

Present results as a structured report:

```
## Validation Results

### Tags
| Service | From | To | Status |
|---------|------|----|--------|
| {service} | {old_tag} | {new_tag} | Tag exists / MISSING |

### {service} ({compare range}) — OK / ISSUE
{N} commits — summary of what's included and whether all are accounted for.

### Verdict
All tags exist, all commits accounted for, no undocumented changes. LGTM.
— or —
Issues found: {list of problems}
```

### 7. Post comment on PR (if requested)

If the user asks to add a comment, post the validation report as a PR comment:

```bash
gh pr comment {number} --repo y-tree-limited/argocd-monorepo --body "{report}"
```

## Checks performed

1. **Tag existence** — every target tag must exist in the source repo
2. **Commit coverage** — every commit in the compare range must be accounted for in the PR description
3. **Tag format** — `app-values.yaml` must use tags without `v` prefix
4. **Hotfix consistency** — for hotfix-to-hotfix bumps, verify only expected cherry-picks are included
5. **No extra files** — only `app-values.yaml` files for the listed services should be changed

## Notes

- This skill is read-only — it does not modify any repository or create any branches
- Tags in source repos use the `v` prefix; tags in `app-values.yaml` do not
- For hotfix tags, validate the cherry-pick scope matches the PR description's rationale