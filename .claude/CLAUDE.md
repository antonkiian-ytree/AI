# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an **ops automation repo** — a collection of Claude Code skills and configuration for managing deployments, releases, and operational workflows for the **LUNA squad** at **Y-Tree**. It contains no application code; its value is entirely in the `.claude/` directory (skills, settings, memory).

## Architecture

- **`.claude/skills/`** — Slash-command skills that automate recurring DevOps tasks:
  - `release-luna` — Prepare LUNA production release PRs in argocd-monorepo (tag bumps, Jira linking, compare links, hotfix handling)
  - `release-renovation` — Batch-promote services with only automated dependency updates to production
  - `promote-pre-prod` — Trigger the `promote-service.yaml` GitHub Actions workflow to update preprod tags
  - `job-renovate` — Find, review, approve, and auto-merge open Renovate PRs across y-tree-limited org
  - `report-squad-stats` — Collect monthly Tech Delivery and Tech Operations metrics from Jira, Slack, GitHub, and Grafana
  - `sprint-delivery-progress` — Generate sprint delivery progress report (last sprint delivered + current sprint focus areas) from Jira
- **`repos/argocd-monorepo/`** — Git-ignored local clone of `y-tree-limited/argocd-monorepo`, used by release skills for tag bumps. If missing, skills fall back to `/Users/antonkiian/projects/ytree/argocd-monorepo`.
- **`.claude/settings.local.json`** — Pre-approved permissions for common `gh` commands, Jira search, and Slack reads.

## Atlas Workspace

The primary development workspace is at `/Users/antonkiian/projects/ytree/atlas-workspace/.claude/worktrees/atlas-team-squad-luna/`. It contains:
- **`workspace.yaml`** — squad-luna's owned repos (with roles: project/reference/knowledge) and cross-repo relationships
- **`bin/atlas`** — CLI tooling for PR creation, snapshot deploys, worktree management, workflow runs
- **`knowledge/`** — backend/frontend standards repos (read-only context)

When skills in this repo need to create branches or PRs in squad-luna service repos, use the atlas workspace tooling and conventions (e.g., `bin/atlas pr create`, `bin/atlas wf run`). See `snapshot-deploy.md` and other references under `.claude/skills/deploy/references/` for deploy flows.

**Maintenance:** Refresh the atlas workspace every weekend by running `bin/atlas workspace refresh` from the worktree directory. This keeps repo clones, knowledge bases, and the repo index up to date.

## Key External Systems

- **GitHub org:** `y-tree-limited` — all service repos and argocd-monorepo live here
- **Jira:** `yofiworld.atlassian.net` (cloud ID: `3e0004f3-0133-4a66-85f9-a2ce7b5ba86d`)
- **Slack:** `#production_issues` (`CA0P0BJQ7`), `#tech-leadership` (`C03NQGVEQFK`)
- **Grafana:** Application Security dashboard at `d/application-security/application-security`

## Tag Convention

- **Source repos** (GitHub tags, compare links): use `v` prefix (e.g., `v130.0.33`)
- **argocd-monorepo `app-values.yaml`**: no `v` prefix (e.g., `tag: 130.0.33`)

## Working With Skills

Skills are self-contained — each `SKILL.md` has the full workflow, input format, and output format. When modifying a skill, preserve its existing structure (frontmatter, sections, examples). Skills are invoked via slash commands (e.g., `/release-luna`, `/job-renovate`).

## Memory

Project-scoped memory is stored in `.claude/memory/`. Use it to persist context about ongoing work, decisions, and ticket background that will be useful across conversations. See `MEMORY.md` for the index.