Create a report proposing OpenSearch-project maintainers for emeritus status, split into ten weekly batches so that no single repository has more than one maintainer proposed for emeritus in the same batch, and publish it as a public GitHub gist on the user's account.

## Goal

Moving an inactive maintainer to emeritus status is a real, visible action against a real person's record on a repo. Doing it for every inactive maintainer on a given repo all at once concentrates the awkwardness (and the review/approval burden) on that repo's active maintainers in a single week. Spreading proposals across ten weekly batches, with at most one proposal per repo per batch, distributes that workload evenly over time.

## Data sources

- **Maintainer inactivity**: `maintainer-inactivity-*` indices on the OpenSearch metrics cluster at `metrics.opensearch.org`. Use the Dashboards console proxy. Fields relevant here: `repository`, `github_login`, `inactive` (bool, stored as 0/1), `event_type` (text; use `event_type.keyword` for aggregations), `current_date` (a date field — `max` returns epoch-millis; filter snapshots with a numeric `term`, not a string — see `missing-members-report.md` for the gotcha).
- **Repo metadata (archived flag)**: GitHub REST API, `GET /repos/opensearch-project/{repo}`, or batch via GraphQL aliases to avoid one call per repo.
- **Maintainer profile (html_url)**: `gh api /users/{login}` or a batched GraphQL query.

## Method

1. Find the most recent `current_date` snapshot:
   ```
   size: 0, aggs: { max_date: { max: { field: "current_date" } } }
   ```
2. Within that snapshot, restrict to `event_type.keyword = "All"` (the rollup row — one document per maintainer per repo).
3. Aggregate by `repository.keyword` to get total maintainers and inactive count per repo (`sum` of `inactive`), same as `maintainer-report.md`. Derive active = total − inactive.
4. Pull the individual rows where `inactive = true` — these are the emeritus candidates (`repository`, `github_login` pairs).
5. Exclude candidates whose repo has `archived = true`.
6. **Batch assignment** — the core of this report:
   a. Group remaining candidates by repo.
   b. Sort repos by descending candidate count (ties broken by repo name ascending), so the repos hardest to fit are placed first.
   c. Within each repo, sort its candidates by `github_login` ascending (there's no per-candidate "how long inactive" signal in this index, so this is just a deterministic tie-break).
   d. Maintain a running count per batch (10 batches, initially empty), plus a per-batch count of how many times each maintainer login has already been assigned. For each repo's candidates in order, restrict to batches that **do not yet contain a candidate from this repo** (hard constraint — fall back to all 10 batches only if every batch already has one, per 6e). Among that restricted set, pick the batch with (1) the fewest existing entries for this maintainer login, then (2) the lowest total count, then (3) lowest batch number. The maintainer tie-break is a soft preference, not a hard rule — the same person is often inactive on several repos, and this just keeps them from landing in the same week twice when an alternative week is available.
   e. Exception: if a single repo has more than 10 inactive maintainers, it cannot avoid a repeat within some batch. In that case, once every batch has one of its candidates, allow repeats but space them out as evenly as possible (largest gap between repeat batch numbers), and call this repo out explicitly in the summary as violating the "one per repo per batch" rule.
   f. Within each batch, sort entries by repo name ascending.
   g. Label batch *N* as starting the week of `{run date} + 7*(N-1) days`, so batch 1 is the current week and batch 10 is nine weeks out.

## Report format

Markdown. Above everything, a **Summary** section with:
- Snapshot `current_date` used
- Total candidates (inactive maintainer × repo pairs) found
- **Unique maintainers represented** — distinct `github_login` count, since the same person can be inactive on several repos and so show up more than once across the ten batches. Call out how many maintainers repeat across repos and name the most-repeated few.
- Count and names of archived repos excluded
- Number of distinct repos represented across all batches
- Batch sizes (should be roughly equal — total candidates / 10)
- Any repo that triggered the >10-candidates exception in step 6e, and how its repeats were spaced

Then ten subsections, one per batch:

`## Batch N — week of YYYY-MM-DD`

Each with a table:

| Column | Notes |
|---|---|
| Repo | Linked to `https://github.com/opensearch-project/{repo}` |
| Maintainer | Linked to `{html_url}` |
| Repo total maintainers | From the snapshot aggregation |
| Repo active maintainers | `total − inactive` |
| Repo % inactive | Two decimal places, trailing `%` — context for how urgent this repo's cleanup is overall |

## Publishing

Create a **public** gist on the user's GitHub account using `gh gist create --public`. Filename: `opensearch-emeritus-batches.md`. Return the gist URL.

If a previous run created a gist and it should be updated rather than replaced, use `gh gist edit <gist-id> <file>` instead.

## Filing emeritus PRs (per candidate, on request)

The report only identifies and batches candidates — it does not open PRs by itself. When asked to actually propose a specific candidate (e.g. "open a PR for this person"), do the following, modeled on `opensearch-project/opensearch-api-specification#1177`:

1. Fork the target repo to the user's account if a fork doesn't already exist: `gh repo fork opensearch-project/{repo} --clone`.
2. Clone (or relocate the clone) to `/Users/epugh/Documents/projects/opensearch/{repo}-epugh`, matching the user's existing local naming convention for forks in that directory.
3. Branch name: `move-{login}-emeritus`.
4. Edit `MAINTAINERS.md`: remove the candidate's row from the "Current Maintainers" table; append (or add to an existing) `## Emeritus` section with a `| Maintainer | GitHub ID |` table row for them.
5. Edit `.github/CODEOWNERS` (if the login appears there): remove `@{login}`.
6. Commit with DCO sign-off (`git commit -s`) using message `Update maintainer status for {login}`.
7. Push the branch to the user's fork and open the PR against `opensearch-project/{repo}:main` (or the repo's actual default branch) with `gh pr create`:
   - **Title**: `Propose moving @{login} to Emeritus status`
   - **Dashboard link**: Substitute the target repository name for `{repo}` in the URL below. The encoded query filters the dashboard to `repository.keyword: "{repo}"`; keep the existing tenant and time range.
   - **Body**:
     ```
   Hey @{login}, we noticed you haven't used your maintainer privileges in this repository over the past year. Per the OpenSearch Project [inactivity policy](https://github.com/opensearch-project/technical-steering/blob/main/policies/RESPONSIBILITIES.md#inactivity), maintainers inactive for 12 months or more are moved to emeritus status.
   
   **If you plan to continue contributing as a maintainer**, please respond here and close this PR and no further action is needed.
   
   Otherwise, this PR will move you to the emeritus list. Emeritus status is not permanent: you can return to active maintainer status at any time by
   expressing interest to the current maintainers.
   
   **Existing maintainers**: Please merge this PR once @{GITHUB_HANDLE} confirms they do not plan to use their maintainer privileges, or after 7 days with no response. If neither maintainers nor @{GITHUB_HANDLE} take any action within 7 days, a member of the [admin team](https://github.com/orgs/opensearch-project/teams/admin) will merge it.
   
   ### Activity data
   
   This determination is based on the [OpenSearch Maintainer Dashboard](https://metrics.opensearch.org/_dashboards/app/dashboards#/view/30fedc30-9ae2-11ef-a168-f19b1bbc360c) (filter by repository: `{repo}`). If you believe the activity data is mistaken, please say so here so we can investigate before merging.
     ```
8. This is a visible, real action against a real person's status — always confirm the planned diff and PR text with the user before pushing/opening the PR, even when working through a batch.
9. After opening the PR, please update the gist to track the PR being created.

## Caveats to keep in the report

- "Inactive" is defined by the ingestion pipeline that populates `maintainer-inactivity-*`. The threshold is on the order of months; do not claim a specific number unless it has been verified against the pipeline source.
- Moving a maintainer to emeritus only touches maintainers already flagged inactive — it does not change any repo's **active** maintainer count or quorum standing. A repo already short of the 3-active-maintainer quorum (see `orphan-repos-report.md`) still needs new maintainer recruitment; emeritus cleanup doesn't fix that, and the report should not imply otherwise.
- A maintainer flagged inactive is not necessarily gone for good, and a repo at high % inactive may simply have a stale `MAINTAINERS.md`. Verify with the repo's active maintainers before actually filing the emeritus PR — this report proposes an order of operations, it doesn't authorize the change.
- The batching only *guarantees* "no repo repeated within a batch" (except the >10-candidate exception in 6e) and roughly even batch sizes. Keeping the same maintainer out of a given week twice is a best-effort secondary preference (step 6d), not a hard rule — a maintainer inactive on many repos can still land in the same batch more than once if no alternative week is free.
- The batching does not account for a maintainer already mid-conversation about emeritus status outside this process — cross-check recent issues/PRs on the repo before sending a given week's batch.
