Create a report of OpenSearch-project repository maintainer inactivity and publish it as a public GitHub gist on the user's account.

## Data sources

- **Maintainer inactivity**: `maintainer-inactivity-*` indices on the OpenSearch metrics cluster at `metrics.opensearch.org`. Use the Dashboards console proxy. Fields relevant here: `repository`, `github_login`, `inactive` (bool, stored as 0/1), `event_type` (text; use `event_type.keyword` for aggregations), `current_date` (keyword timestamp).
- **Repo metadata (stars, forks, open issues, archived flag, default branch)**: GitHub REST API, `GET /repos/opensearch-project/{repo}`.
- **Open PR count**: GitHub search API, `GET /search/issues?q=repo:opensearch-project/{repo}+type:pr+state:open`. Do not use `open_issues_count` from the repo endpoint for this — it is issues plus PRs.
- **Latest human commit on default branch**: GitHub REST API, `GET /repos/opensearch-project/{repo}/commits?sha={default_branch}&per_page=100`. Walk the returned commits and pick the most recent one that passes **all four** filters (merge-commit, bot, automated-message, broadcast). Paginate further if every commit on page 1 is filtered out.

  **Filter 0 — merge commits.** Skip any commit with `len(parents) >= 2` (i.e. a merge commit, not a squash). A merge commit is just a pointer that packages other commits into the branch; the actual author work is the commits being merged in, which appear elsewhere in the linear history and are evaluated by Filters 1–3 on their own merits. Squash-merge is the norm in most repos in this org, but some repos (e.g. `perftop`, `ux`, `traffic-comparator`) use `--no-ff` merge commits, and without this filter a merge commit hides the underlying broadcast content (a unique "Merge pull request #N from shreyah963/foo" message escapes Filter 3 even when the PR itself is a broadcast cleanup).

  **Filter 1 — bot logins.** Skip any commit whose `author.login` (falling back to `committer.login` if author is null) is a bot (case-insensitive, also match any login ending in `[bot]`):
  - `dependabot[bot]`, `dependabot-preview[bot]`
  - `mend-for-github-com[bot]`, `renovate[bot]`
  - `github-actions[bot]`
  - `opensearch-trigger-bot[bot]`, `opensearch-ci-bot`, `opensearch-changeset-bot[bot]`
  - any other login with `type == "Bot"` in the commit response

  **Filter 2 — automated commit messages.** Skip any commit whose first message line matches:
  - `Bump <name> from <ver> to <ver>` (dependabot signature)
  - `chore(deps): update <name>` (Mend/Renovate signature)
  - `[AUTO] ...`, `[Backport <branch>] ...` (project automation)

  **Filter 3 — cross-repo broadcast commits.** A "broadcast commit" is a human-authored commit that is part of an org-wide cleanup applied identically to many repos at the same time (e.g. `@shreyah963` landing "Add issues write permission to untriaged label workflow" across 100+ repos in the same week). These commits are technically human but should not be treated as evidence that this particular repo is being maintained. To detect them:
  1. For each non-archived repo, fetch the most recent ~30 commits on the default branch (after Filters 1 and 2).
  2. Build a global index keyed by `(author_login, first_line_of_commit_message_normalized)`. Normalization: lowercase, strip the trailing `(#<PR-number>)` if present, collapse whitespace.
  3. For each `(author, message)` pair, look at the set of repos it appears in. If the same pair appears in **≥ 5 distinct repos** with commit dates spanning **≤ 14 days** (min-to-max), mark every commit with that `(author, message)` pair across those repos as **broadcast** and skip it when picking the latest human commit.
  4. The thresholds (5 repos, 14 days) are tunable; they should be stated explicitly in the report's method-notes section so a reader knows what is being filtered.

  Record the `commit.committer.date` of the most recent commit that passes all three filters, or `n/a` if every commit in the scanned range is filtered out. **Why this matters:** Dependabot/Mend/Renovate/opensearch-trigger-bot/opensearch-ci-bot/github-actions all commit across many repos on a schedule, AND engineers periodically land org-wide cleanups (CI/lint changes, security workflow updates) identically across 50–100+ repos in the same week. Either category would make abandoned repos look freshly active without all three filters.

## Method

1. Find the most recent `current_date` snapshot in `maintainer-inactivity-*`:
   ```
   size: 0, aggs: { max_date: { max: { field: "current_date" } } }
   ```
2. Within that snapshot, restrict to `event_type.keyword = "All"` (the rollup row — one document per maintainer per repo). The other `event_type` values partition the same maintainers across event categories and would double-count.
3. Aggregate by `repository.keyword`, computing total maintainers (`doc_count`) and inactive count (`sum` of the `inactive` field) per repo.
4. For each repo, fetch its GitHub metadata, PR count, and **latest human commit** timestamp (bot-filtered — see Data sources).
5. Exclude repos where `archived = true`. Do not include them at all.
6. Sort by `% inactive` descending. Tie-break by inactive count desc, then maintainer count desc, then repo name ascending.
7. For each repo, assess if there are a large number of user issues or PR's that aren't getting dealt with? Provide a three sentence summary for the repo summarizing it's status.

## Report format

Markdown. One table, sorted by % inactive descending. Columns:

| Column | Notes |
|---|---|
| Rank | 1-indexed |
| Repo | Linked to `https://github.com/opensearch-project/{repo}` |
| Maintainers | Total maintainers from the inactivity snapshot |
| Inactive | Count where `inactive = true` |
| % Inactive | `inactive / maintainers * 100`, two decimal places, trailing `%` |
| Stars | `stargazers_count` |
| Forks | `forks_count` |
| Open Issues | `open_issues_count - open_prs` (PRs subtracted) |
| Open PRs | From search API `total_count` |
| Latest human commit (default branch) | `YYYY-MM-DD` from committer date of the most recent commit that passes all four filters (merge-commit, bot, automated-message, broadcast). `n/a` if every commit in the scanned range is filtered. **Do not** report a date if the latest commit is a merge commit (`parents >= 2`), is from a bot login (Dependabot/Mend/Renovate/opensearch-trigger-bot/opensearch-ci-bot/github-actions or any `*[bot]`), matches an automated-bump message, or is part of an org-wide broadcast cleanup (`(author, first-line-of-message)` pair appearing in ≥ 5 distinct repos within a 14-day span) — those land across nearly every repo and make stale repos look freshly active. |
| Summary | Per repo summary |

Include a summary section above the table stating:
- Snapshot `current_date` used
- Total repos in the snapshot
- Count and names of archived repos that were excluded
- Count remaining in the report

Note in the summary that "Latest human commit" is the most recent commit on the default branch that passes four filters: (a) not a merge commit (`parents >= 2`), (b) not a bot login, (c) not an automated-bump message, and (d) not a cross-repo broadcast cleanup (`(author, first-line-of-message)` pair appearing in ≥ 5 distinct repos within a 14-day span). Explain that without all four filters, Dependabot/Mend/Renovate/opensearch-trigger-bot/opensearch-ci-bot/github-actions activity plus org-wide engineer-driven cleanups (CI/lint sweeps, security workflow updates) would mask abandoned repos as freshly active, and that merge commits on repos using `--no-ff` would hide broadcast content behind a unique merge-pull-request message. State the broadcast thresholds (5 repos, 14 days) so a reader knows what is being filtered.

## Publishing

Create a **public** gist on the user's GitHub account using `gh gist create --public`. The `gh` CLI is authenticated with `gist` scope. Filename: `opensearch-maintainer-inactivity.md`. Return the gist URL.

If a previous run created a gist and it should be updated rather than replaced, use `gh gist edit <gist-id> <file>` instead.

## Caveats to keep in the report

- "Inactive" is defined by the ingestion pipeline that populates `maintainer-inactivity-*`. The threshold is on the order of months; do not claim a specific number unless it has been verified against the pipeline source.
- A repo at 100% inactive is not necessarily abandoned — it may simply have a stale MAINTAINERS.md. Flag but do not draw conclusions.
