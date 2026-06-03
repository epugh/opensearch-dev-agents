Create a report of OpenSearch-project repositories that are potentially orphaned or abandoned, and publish it as a public GitHub gist on the user's account.

## Goal

Identify repositories at risk of being orphaned. The OpenSearch project requires a quorum of **three active maintainers** to nominate a new maintainer (see the project's `MAINTAINERS.md` governance). A repo is at risk when it cannot form (or barely forms) that quorum, or when its maintainer roster has not refreshed in a long time. Two patterns matter most:

1. **Small + inactive**: few maintainers listed, most of them inactive, so the repo cannot muster three active maintainers to bring in new help.
2. **Large but inactive**: many maintainers listed but most flagged inactive — the roster looks healthy on paper but isn't in practice.

Stagnation is a third signal: a long time since a new maintainer was last added suggests the repo isn't evolving or growing its bench.

## Data sources

- **Maintainer inactivity**: `maintainer-inactivity-*` indices on the OpenSearch metrics cluster at `metrics.opensearch.org`. Use the Dashboards console proxy. Fields: `repository`, `github_login`, `inactive` (bool, 0/1), `event_type` (use `event_type.keyword` for aggregations), `current_date` (keyword timestamp).
- **Repo metadata (stars, forks, archived flag, default branch)**: GitHub REST API, `GET /repos/opensearch-project/{repo}`.
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

  Record the `commit.committer.date` of the most recent commit that passes all three filters, or `n/a` if every commit in the scanned range is filtered out.
- **Last new maintainer added**: derived from the commit history of `MAINTAINERS.md` on the default branch. See the method section.

## Method

1. Find the most recent `current_date` snapshot in `maintainer-inactivity-*`:
   ```
   size: 0, aggs: { max_date: { max: { field: "current_date" } } }
   ```
2. Within that snapshot, restrict to `event_type.keyword = "All"` (the rollup row — one document per maintainer per repo).
3. Aggregate by `repository.keyword`, computing total maintainers (`doc_count`) and inactive count (`sum` of `inactive`) per repo. From these derive **active = total − inactive**.
4. Fetch GitHub metadata and **latest human commit** timestamp for each repo (see Data sources for the bot-filtering rules — this is important: bot-only activity must not make a repo look active).
5. Exclude repos where `archived = true`.
6. For each remaining repo, determine **last new maintainer added date**:
   - List commits that touched `MAINTAINERS.md` on the default branch: `GET /repos/opensearch-project/{repo}/commits?path=MAINTAINERS.md&sha={default_branch}&per_page=100` (paginate if needed).
   - Walk the commits from oldest to newest. For each commit, fetch the patch via `GET /repos/opensearch-project/{repo}/commits/{sha}` and inspect the diff for `MAINTAINERS.md`.
   - A commit "adds a new maintainer" if a line was added (`+` in the patch, ignoring header/whitespace lines) that introduces a GitHub login not present in any earlier version. The OpenSearch standard format uses a markdown table with a `GitHub ID` column; match `@username` references or table cells containing a recognizable handle.
   - Record the **committer date** of the most recent such commit. If the file does not exist or no addition can be found (e.g. only renames/format changes), record `n/a`.
   - Performance note: walking every diff is expensive. Cache responses; you can stop walking once you have the most recent addition that newly introduces a login currently present in the file.
7. Classify each repo into an **orphan risk tier**:
   - 🔴 **Critical** — fewer than 3 active maintainers (cannot form quorum to nominate a new maintainer).
   - 🟠 **Fragile** — exactly 3 active maintainers (one departure breaks quorum), OR active ≥ 3 but the last new maintainer was added more than 3 years ago.
   - 🟡 **At risk** — active < 5 AND % inactive ≥ 50%, OR last new maintainer added more than 2 years ago.
   - 🟢 **Healthy** — none of the above.
   Apply tiers in order; a repo gets the highest-severity tier it matches.
8. Sort by tier severity (Critical → Fragile → At risk → Healthy). Within a tier, sort by **active maintainer count ascending**, then by **days-since-last-new-maintainer descending**, then by repo name ascending.

## Report format

Markdown. Above the table include a **Summary** section with:
- Snapshot `current_date` used
- Total repos in the snapshot
- Count and names of archived repos excluded
- Count remaining in the report
- Count of repos per risk tier (🔴 / 🟠 / 🟡 / 🟢)
- A short note explaining "Latest human commit" = the most recent commit on the default branch that passes four filters: not a merge commit (`parents >= 2`), not a bot login, not an automated-bump message, and not a cross-repo broadcast cleanup. Explain **why** all four matter: Dependabot, Mend/Renovate, opensearch-trigger-bot, opensearch-ci-bot, and github-actions commit across many repos on a schedule; engineers occasionally land org-wide cleanups (e.g. CI/lint changes) identically across 50–100+ repos in the same week; and repos using `--no-ff` merge commits would otherwise show a merge-commit "Merge pull request #N from foo/bar" as the latest activity, hiding the broadcast content that the PR actually brought in. State the broadcast thresholds used (default: a `(author, first-line-of-message)` pair appearing in ≥ 5 distinct repos within a 14-day span)
- A short note explaining how "Last new maintainer added" is derived (commits to `MAINTAINERS.md` introducing a new GitHub login)
- A short note explaining the 3-active-maintainer quorum rule

Then one table, sorted as in step 8. Columns:

| Column | Notes |
|---|---|
| Tier | 🔴 / 🟠 / 🟡 / 🟢 |
| Rank | 1-indexed within the whole sorted list |
| Repo | Linked to `https://github.com/opensearch-project/{repo}` |
| Maintainers | Total maintainers from the inactivity snapshot |
| Active | `maintainers − inactive` |
| Inactive | Count where `inactive = true` |
| % Inactive | `inactive / maintainers * 100`, two decimal places, trailing `%` |
| Quorum gap | `max(3 − active, 0)` — number of additional active maintainers needed to reach the quorum of 3 |
| Last new maintainer | `YYYY-MM-DD` of the most recent commit to `MAINTAINERS.md` that introduces a new GitHub login, or `n/a` |
| Years since last new maintainer | Decimal years from "Last new maintainer" to the snapshot date, two decimals; `n/a` if unknown |
| Stars | `stargazers_count` |
| Latest human commit (default branch) | `YYYY-MM-DD` from committer date of the most recent commit that passes all four filters (merge-commit, bot, automated-message, broadcast). `n/a` if every commit in the scanned range is filtered. **Do not** report a date if the latest commit is a merge commit (`parents >= 2`), is from a bot login (Dependabot/Mend/Renovate/opensearch-trigger-bot/opensearch-ci-bot/github-actions or any `*[bot]`), matches an automated-bump message, or is part of an org-wide broadcast cleanup — those land across nearly every repo and make stale repos look freshly active. |
| Why flagged | One sentence per repo explaining the specific reason for the tier (e.g. "0 active maintainers; MAINTAINERS.md last refreshed 4 years ago", "3 active out of 8 listed; no new maintainer added in 2.5 years"). For 🟢 Healthy repos this can be a short positive note. |

## Publishing

Create a **public** gist on the user's GitHub account with `gh gist create --public`. Filename: `opensearch-orphan-repos.md`. Return the gist URL.

If a previous run created a gist for this report and it should be updated rather than replaced, use `gh gist edit <gist-id> <file>` with the same filename.

## Caveats to keep in the report

- "Inactive" is defined by the ingestion pipeline that populates `maintainer-inactivity-*`. The threshold is on the order of months; do not claim a specific number unless it has been verified against the pipeline source.
- The tier thresholds (quorum = 3, 2 / 3 years for stagnation) come from this report's framing, not from formal OpenSearch governance documents. Note that explicitly.
- "Last new maintainer added" is heuristic: it depends on `MAINTAINERS.md` being kept up to date in-tree. A repo that tracks maintainers elsewhere, or that has reformatted `MAINTAINERS.md`, may show a misleading date. Flag, do not conclude.
- A repo flagged Critical or Fragile is not necessarily abandoned — `MAINTAINERS.md` may simply be stale. The action this report suggests is **investigate**, not **deprecate**.
