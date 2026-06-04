Create a report of OpenSearch-project repository active maintainers who are not members of the OpenSearch-project organization and publish it as a public GitHub gist on the user's account.

## Data sources

- **Maintainer inactivity**: `maintainer-inactivity-*` indices on the OpenSearch metrics cluster at `metrics.opensearch.org`. Use the Dashboards console proxy. Fields relevant here: `repository`, `github_login`, `inactive` (bool, stored as 0/1), `event_type` (text; use `event_type.keyword` for aggregations), `current_date` (a date field — `max` returns epoch-millis, e.g. `1780531226685`).
- **List of members**: Use the GH CLI to request all members of Opensearch-project

## Method

1. Find the most recent `current_date` snapshot in `maintainer-inactivity-*`:
   ```
   size: 0, aggs: { max_date: { max: { field: "current_date" } } }
   ```
   `current_date` indexes as a date, so the max comes back as epoch-millis. Filter the snapshot with a numeric `term` on that value, not a string.
2. Within that snapshot, restrict to `event_type.keyword = "All"` (the rollup row — one document per maintainer per repo). The other `event_type` values partition the same maintainers across event categories and would double-count.
3. Keep only rows where `inactive = false` (active maintainerships).
4. Reduce to distinct `github_login`. A maintainer counts as active if they have at least one active row from step 3.
5. Get a list of all members of the OpenSearch-Project using GH CLI. Drop any maintainer whose `github_login` matches a member `login`. Compare case-insensitively — GitHub logins are case-insensitive.
6. For each remaining (active, non-member) maintainer, fetch GitHub user metadata (`html_url`, `type`) via `gh api /users/{login}` or one batched GraphQL call. Flag logins that return 404 — renamed or deleted accounts, which are necessarily non-members.

## Report format

Markdown. One table, sorted by maintainer login ascending (case-insensitive). Columns:

| Column | Notes |
|---|---|
| Rank | 1-indexed |
| Maintainer | Linked to `{html_url}` |
| Maintainer Type | `{type}` |

Include a summary section above the table stating:
- Snapshot `current_date` used
- Total members identified in the project
- Total maintainers identified from the snapshot
- Total active maintainers in the snapshot (`inactive = false`)
- Count remaining in the report (active, non-member maintainers)


## Publishing

Create a **public** gist on the user's GitHub account using `gh gist create --public`. The `gh` CLI is authenticated with `gist` scope. Filename: `opensearch-maintainers-not-members.md`. Return the gist URL.

If a previous run created a gist and it should be updated rather than replaced, use `gh gist edit <gist-id> <file>` instead.

## Caveats to keep in the report

- "Active" here is the complement of the `inactive` flag set by the ingestion pipeline that populates `maintainer-inactivity-*`. The inactivity threshold is on the order of months; do not claim a specific number unless it has been verified against the pipeline source.
- The `inactive` flag is per maintainer-per-repo. A maintainer who is active on one repo and inactive on another is treated as active (they have at least one active row).
- Not being an org member does not necessarily mean a maintainer lacks repo access — OpenSearch also grants access via teams and outside-collaborator roles. This report measures org membership only.
