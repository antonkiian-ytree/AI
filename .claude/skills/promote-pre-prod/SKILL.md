---
name: promote-pre-prod
description: >
  Promote a service to preprod in argocd-monorepo by triggering the promote-service GitHub Actions
  workflow. Use when the user says "update preprod ms-xxx with <tag>", "promote <tag> to preprod",
  "deploy <snapshot> to preprod", or similar.
---

# Promote to Pre-Prod

Promotes a service to preprod by running the `promote-service.yaml` GitHub Actions workflow
in argocd-monorepo. The workflow auto-creates a PR, and `auto-approve-preprod.yaml` auto-approves it.

## Input

Parse the user's request for:
- **Service name** — e.g., `ms-cpd`, `ms-asset`, `mf-billing`. Both `ms-*` and `mf-*` services use the same workflow.
- **New tag** — the image tag to set (e.g., `5.1.130`, `0.1.65`, `SNAPSHOT-4478cae...`)

If the service name is missing, ask the user.

If the tag is missing, look it up from the **development** environment in argocd-monorepo:

```bash
grep 'tag:' /Users/antonkiian/projects/ytree/argocd-monorepo/environments/development/{service}/app-values.yaml
```

Propose the dev tag to the user for confirmation before proceeding (e.g., "ms-cpd is on `5.1.158` in dev — shall I promote that to preprod?").

## Pre-flight: Check current preprod version

Before triggering the workflow, check the current preprod tag:

- For `ms-*` services: `grep 'tag:' .../environments/preprod/{service}/app-values.yaml`
- For `mf-*` services: `grep '{service}' .../environments/preprod/cm-frontend-server-versions/app-values.yaml`

**If the current preprod version is a SNAPSHOT**, warn the user and ask for confirmation before proceeding:
> "⚠️ ms-cpd is currently on `SNAPSHOT-abc123...` in preprod. Are you sure you want to overwrite it with `{new_tag}`?"

Only proceed after the user confirms. This prevents accidentally replacing an in-progress SNAPSHOT deployment.

## Workflow

### 1. Run the promotion workflow

```bash
gh workflow run promote-service.yaml \
  -R y-tree-limited/argocd-monorepo \
  -f environment=preprod \
  -f services="{service}@{tag}"
```

Multiple services can be promoted at once by space-separating:
```bash
-f services="{service1}@{tag1} {service2}@{tag2}"
```

### 2. Wait for the PR to be created

Poll for the workflow run and its PR. The workflow creates a PR automatically — wait for it to appear:

```bash
# Get the latest run ID for the workflow
gh run list -R y-tree-limited/argocd-monorepo -w promote-service.yaml -L 1 --json databaseId,status --jq '.[0]'
```

Wait for the run to complete, then find the PR it created:

```bash
gh pr list -R y-tree-limited/argocd-monorepo --search "promote {service}" --json number,title,url --jq '.[0]'
```

### 3. Enable automerge and approve the PR

Once the PR is found, enable automerge and approve it:

```bash
gh pr merge {pr_number} -R y-tree-limited/argocd-monorepo --auto --squash
gh pr review {pr_number} -R y-tree-limited/argocd-monorepo --approve
```

### 4. Confirm

Report the PR URL to the user and confirm that automerge and approval have been set.

## Notes

- This skill is for single or small-batch preprod tag updates — for production releases, use `release-luna` or `release-renovation` instead
- The workflow handles branching, committing, pushing, and PR creation automatically
- Automerge and approval are enabled by default — the PR will merge once all checks pass
