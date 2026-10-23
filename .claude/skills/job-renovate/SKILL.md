---
name: job-renovate
description: >
  Process open Renovate PRs for the y-tree-limited organization. Use when the user says
  "process renovate PRs", "handle renovate", "review renovate PRs", or similar.
---

Process open Renovate PRs for the y-tree-limited organization.

## Steps

1. **Find all matching PRs** by running three separate GitHub API searches and deduplicating results by PR URL:
   ```
   is:open is:pr user:y-tree-limited archived:false team-review-requested:y-tree-limited/squad-luna head:renovate sort:updated-desc
   ```
   ```
   is:open is:pr user:y-tree-limited archived:false review-requested:@me head:renovate sort:updated-desc
   ```
   ```
   is:open is:pr user:y-tree-limited archived:false reviewed-by:@me head:renovate sort:updated-desc
   ```
   Use `gh api "search/issues?q=..."` for each query (handle pagination if needed), then merge and deduplicate by PR URL.

2. **Check CI status** for each PR:
   - For each PR, get the latest workflow run on the PR's head branch using `gh run list --repo <repo> --branch <branch> --limit 1`.

3. **For PRs with successful builds** — present the list and ask the user for confirmation before proceeding. Only after the user confirms:

   **Split PRs into two groups based on age** (use `createdAt` from the PR):
   - **Recent PRs** (created less than 7 days ago): approve and auto-merge directly, no branch update needed.
   - **Old PRs** (created 7+ days ago): these are likely out-of-date with the base branch and need
     the sequential branch-update flow described below.

   **Recent PRs (< 7 days old):**
   - If only **one PR per repo**: approve and auto-merge directly.
   - If **multiple PRs in the same repo**: process sequentially — approve and auto-merge the first,
     wait for it to merge, then update the next PR's branch before approving it. Same sequential
     flow as old PRs (the branch goes out-of-date after each merge).

   **Old PRs (≥ 7 days old) — only process if CI is already passing.** Skip old PRs with failed or
   in-progress builds. For passing old PRs, **process sequentially within each repo** (one at a time):

   a. **First old PR in the repo:**
      - Update the branch (merge base into PR branch):
        `gh api repos/y-tree-limited/<repo>/pulls/<pr_number>/update-branch -X PUT`
      - Wait for CI to pass on the updated branch (poll with `gh pr checks`).
      - Approve: `gh pr review --approve --repo <repo> <pr_number>`
      - Enable auto-merge: `gh pr merge --auto --squash --repo <repo> <pr_number>`
      - Wait for the PR to actually merge before proceeding to the next PR.
        Poll with `gh pr view --repo <repo> <pr_number> --json state --jq '.state'` until it returns `MERGED`.

   b. **Subsequent old PRs in the same repo:**
      - Wait for the default branch CI to finish (the merge of the previous PR triggers a build).
        Poll with `gh run list --repo <repo> --branch <default_branch> --limit 1 --json status,conclusion`
        until the latest run completes.
      - Update the branch: `gh api repos/y-tree-limited/<repo>/pulls/<pr_number>/update-branch -X PUT`
      - Wait for CI to pass on the updated PR branch.
      - Approve and enable auto-merge.
      - Wait for merge before moving to the next PR.

   c. **PRs across different repos** can be processed in parallel (each repo is independent).

   Report each one as processed.

3a. **For Gradle wrapper PRs** (branch starts with `renovate/gradle`):
   These PRs require regenerating the wrapper locally because Renovate only bumps the version in `gradle/wrapper/gradle-wrapper.properties` but does not update the wrapper JAR and scripts.

   Steps:
   1. Extract the target Gradle version from the PR title or from `gradle-wrapper.properties` on the PR branch.
   2. Clone the repo into `/Users/antonkiian/projects/kiiananton/AI/repos/<repo-name>` (if already cloned, fetch and reset):
      ```bash
      git clone git@github.com:y-tree-limited/<repo-name>.git /Users/antonkiian/projects/kiiananton/AI/repos/<repo-name>
      ```
   3. Check out the Renovate PR branch:
      ```bash
      cd /Users/antonkiian/projects/kiiananton/AI/repos/<repo-name>
      git checkout <branch>
      ```
   4. Run the Gradle wrapper task to regenerate wrapper files:
      ```bash
      gradle :wrapper --gradle-version <version>
      ```
   5. Commit and push the updated wrapper files:
      ```bash
      git add gradle/wrapper/ gradlew gradlew.bat
      git commit -m "Regenerate Gradle wrapper for <version>"
      git push
      ```
   6. Enable auto-merge and approve:
      ```bash
      gh pr merge --auto --squash --repo y-tree-limited/<repo-name> <pr_number>
      gh pr review --approve --repo y-tree-limited/<repo-name> <pr_number>
      ```

   Process all Gradle wrapper PRs in a batch — clone each repo, regenerate, push, approve.

4. **For PRs with failed builds** — present a summary table to the user with:
   - Repository name
   - PR number and title
   - Link to the failed run

5. **For PRs with in-progress builds** — list them separately so the user knows they are still running.

## Output format

After processing, display:
- Count of PRs approved and auto-merged
- List of failed PRs (repo, PR link, failure link)
- List of in-progress PRs (if any)