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

The user either:
1. Provides a table of services with their preprod and production tags, e.g.:
   ```
   │ ms-asset │ 130.0.33 │ 130.0.31 │
   ```
   Parse each row to extract: **service name**, **preprod tag** (target), **production tag** (current).

2. Asks to auto-discover eligible services from the argocd-monorepo (e.g., "do it from the repo",
   "find them yourself"). In this case, follow the **Auto-Discovery** workflow below.

## Auto-Discovery

When the user does not provide a table, discover eligible services automatically:

### 1. Identify squad services

Find all LUNA services by searching for the ownership label in production app-values:

```bash
grep -rl 'owning_team: "luna"' environments/production/*/app-values.yaml \
  | sed 's|environments/production/||;s|/app-values.yaml||' | sort
```

### 2. Find tag differences

For each LUNA service, compare preprod vs production tags:

```bash
for svc in <luna_services>; do
  preprod_tag=$(grep -E '^\s+tag:' "environments/preprod/$svc/app-values.yaml" | head -1 | awk '{print $2}' | tr -d '"')
  prod_tag=$(grep -E '^\s+tag:' "environments/production/$svc/app-values.yaml" | head -1 | awk '{print $2}' | tr -d '"')
  if [ "$preprod_tag" != "$prod_tag" ]; then echo "$svc | $preprod_tag | $prod_tag"; fi
done
```

### 3. Filter candidates

Exclude services where:
- Preprod tag is a **SNAPSHOT** (contains `SNAPSHOT-`)
- Production tag is a **hotfix** (contains `-hotfix-`)

### 4. Verify ytree-admin-only commits

For each remaining candidate, check commit authors between the two tags (use Agent tool for parallel):

```bash
gh api repos/y-tree-limited/{service}/compare/v{prod_tag}...v{preprod_tag} \
  --jq '.commits | map(.author.login) | unique | .[]'
```

Only include services where **every** commit author is `ytree-admin`. If no services qualify, inform
the user that there are no eligible services for a renovation release and show the exclusion reasons.

## Workflow

### 1. Verify tags

If the user provided a table, confirm current production and preprod tags match:

```bash
for svc in <services>; do
  grep -E '^\s+tag:' "environments/production/$svc/app-values.yaml"
  grep -E '^\s+tag:' "environments/preprod/$svc/app-values.yaml"
done
```

If any tag doesn't match, **stop and report** the discrepancy.

If using auto-discovery, tags are already verified from the repo — skip this step.

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