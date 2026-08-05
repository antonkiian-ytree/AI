---
name: release-renovation
description: >
  Prepare a "renovation" production release in argocd-monorepo — batch-promote services whose only
  preprod-vs-production diff is automated dependency updates (ytree-admin). Use when the user says
  "prepare renovation release", "release renovation", "promote automated deps to prod", or provides
  a table of services with preprod ≠ production tags that are ytree-admin only.
---

# Renovation Release — Automated Dependency Promotion to Production

Promotes services to production when the only changes between preprod and production are automated
dependency updates from ytree-admin (Renovate-style batch release).

## Pre-flight

Before making any changes in the argocd-monorepo, **always** switch to the main branch and pull latest:

```bash
git checkout main && git pull origin main
```

## Input

The user provides a table of services with their preprod and production tags, e.g.:

```
│ ms-asset │ 130.0.33 │ 130.0.31 │
```

Parse each row to extract: **service name**, **preprod tag** (target), **production tag** (current).

## Workflow

### 1. Verify tags

For each service, confirm current production and preprod tags match the user's table:

```bash
for svc in <services>; do
  grep -E '^\s+tag:' "environments/production/$svc/app-values.yaml"
  grep -E '^\s+tag:' "environments/preprod/$svc/app-values.yaml"
done
```

If any tag doesn't match, **stop and report** the discrepancy.

### 2. Create branch and bump tags

```bash
git checkout main && git pull origin main
git checkout -b release/luna-renovation
```

For each service, update `environments/production/{service}/app-values.yaml` — change the `image.tag`
value from the current production tag to the preprod tag.

Verify with `git diff --stat` — the number of changed files must match the number of services.

### 3. Gather compare links and PRs

For each service, fetch the PRs between the two tags:

```bash
gh api repos/y-tree-limited/{service}/compare/v{prod_tag}...v{preprod_tag} \
  --jq '.commits | map(.message | split("\n")[0]) | .[]'
```

Extract PR numbers from commit messages (pattern `(#NNN)`), then get PR details:

```bash
gh pr view {number} --repo y-tree-limited/{service} --json title,number,author \
  --jq '"\(.number) | \(.title) | \(.author.login)"'
```

Order PRs by merge timestamp (oldest first).

### 4. Commit and push

```bash
git add environments/production/*/app-values.yaml  # only changed files
git commit -m "Promote N services to production (automated dependency updates)"
git push -u origin release/luna-renovation
```

### 5. Create PR

**Title:** `Renovation: promote N services to production (automated dependency updates)`

**Body format:**

```markdown
Services where PRE-PROD ≠ PRODUCTION and all changes between them are only from ytree-admin (automated dependency updates):

| # | Service | PRE-PROD | PRODUCTION |
|---|---------|----------|------------|
| 1 | {service} | {preprod_tag} | {prod_tag} |
...

---

# Release Description:

Promote N services to production — all changes are automated dependency updates from ytree-admin only.

## Promote to production:
No Jira tickets — automated dependency updates only.

## Compare Changes
- [{service}](https://github.com/y-tree-limited/{service}/compare/v{prod_tag}...v{preprod_tag})
  [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) {pr_title}
  [{service}#{pr}](https://github.com/y-tree-limited/{service}/pull/{pr}) {pr_title}

- [{service2}](...)
  ...
```

**No "Unrelated PRs approval" section** — all PRs are from ytree-admin by definition.

## Notes

- Tags in source repos use `v` prefix (e.g., `v130.0.33`)
- Branch name is always `release/luna-renovation` (not a LUNA-R-NNN release)
- If a `release/luna-renovation` branch already exists on the remote, ask the user before proceeding
- Run compare link gathering in parallel (use Agent tool) for speed
- The PR description has two parts: the overview table first, then the standard release description with compare links
- **Working directory:** use `repos/argocd-monorepo` (relative to project root) for all argocd-monorepo operations. If it doesn't exist, fall back to `/Users/antonkiian/projects/ytree/argocd-monorepo`