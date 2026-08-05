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
   - Approve the PR: `gh pr review --approve --repo <repo> <pr_number>`
   - Enable auto-merge: `gh pr merge --auto --squash --repo <repo> <pr_number>`
   - Report each one as processed.

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