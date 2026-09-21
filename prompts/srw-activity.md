Create a report of recent Search Relevance Workbench (SRW) activity across the two SRW repositories, focused on work done since March 2026 by specific contributors, and identify the tickets (GitHub issues) that work belongs to.

## Repositories

- `opensearch-project/search-relevance` — the SRW backend plugin.
- `opensearch-project/dashboards-search-relevance` — the SRW OpenSearch Dashboards (frontend) plugin.

## Scope

- **Time window:** work landed since **2026-03-01** (inclusive). "Landed" means a pull request merged on or after that date.
- **Contributors of interest (GitHub logins):** `wrigleyDan`, `epugh`, `iprithv`, `venkateshwaracholan`. This list is the single source of truth — every step in Method runs once per login here.
- **"Work"** is any of these contributions by a contributor of interest:
  - **Authored** — a merged PR they opened.
  - **Co-authored** — a merged PR carrying their `Co-authored-by:` trailer.
  - **Reviewed** — a PR where they did *substantial* review in the window: at least one formal review (`APPROVED` / `CHANGES_REQUESTED`) **or** two or more review comments. A single drive-by comment does not count. (This threshold is a heuristic — adjust if results look too noisy or too sparse.)
- Each qualifying PR is traced back to the ticket(s) it closes or references, and the report records which contribution type(s) apply per contributor (someone may author one PR and review another on the same ticket). Reviewed PRs need not be merged — an open PR with substantial review still counts.

## Data sources

- GitHub, via the `gh` CLI (see the `gh.md` steering file for auth, scopes, and the background-shell gotcha).
- Each repo's `CHANGELOG.md` on the default branch.
- Pull requests (merged and open), their reviews and review comments, and the issues (tickets) those PRs close or reference.

## Method

Run every step against **both** repositories and once for **each contributor of interest listed in Scope**. In the commands below, `<login>` is a placeholder for that contributor's GitHub login — substitute each one; do not hard-code the names here (Scope is the single source of truth for the list).

> **Run searches one login at a time.** GitHub does *not* reliably OR repeated qualifiers — combining `author:a author:b …` (or multiple `reviewed-by:`) in a single `--search` can silently return zero. Issue a separate query per contributor and union the results yourself.

> **Validate every login before trusting an empty result.** `gh api /users/<login>` only proves the account *exists*, not that it is the right person — a typo can resolve to a real but unrelated account and then silently match nothing. So: (1) cross-check each Scope login against the repos' `MAINTAINERS.md` (`gh api repos/opensearch-project/<repo>/contents/MAINTAINERS.md --jq .content | base64 --decode`); and (2) treat **zero in-window activity as a red flag, not a finding** — surface it as "possibly wrong login — verify" and reconcile against `MAINTAINERS.md` / recent commits before reporting the contributor as inactive.

1. **Changelog.** Read `CHANGELOG.md` from the default branch and pull out entries added since March 2026, so the report can cross-reference released/landed work:
   ```
   gh api repos/opensearch-project/search-relevance/contents/CHANGELOG.md --jq .content | base64 --decode
   ```

2. **Merged PRs per contributor.** For each contributor, list merged PRs in the window:
   ```
   gh pr list --repo opensearch-project/search-relevance --state merged \
     --search "author:<login> merged:>=2026-03-01" \
     --limit 300 \
     --json number,title,url,author,mergedAt,body,labels
   ```
   Repeat for each contributor and for `opensearch-project/dashboards-search-relevance`.

3. **Co-authored work.** Also capture PRs where a contributor is a co-author but not the primary author (search the PR body / commit trailers for `Co-authored-by:` containing the contributor's login). Note these separately from authored PRs.

4. **Reviewed work.** For each contributor, find PRs they reviewed in the window, excluding their own:
   ```
   gh pr list --repo opensearch-project/search-relevance --state all \
     --search "reviewed-by:<login> -author:<login> updated:>=2026-03-01" \
     --limit 300 --json number,title,url,author,mergedAt,updatedAt
   ```
   Then measure each contributor's review involvement on the candidate PRs and keep only the *substantial* ones (per the threshold in Scope). **Batch this — do not loop `gh pr view` per PR.** A per-PR loop is slow and, when the harness backgrounds it, hangs on the keychain (see `gh.md`). Instead fetch all candidate PRs for a repo in **one GraphQL call** using per-PR aliases, then filter in code:
   ```
   gh api graphql -f query='
   { repository(owner:"opensearch-project", name:"search-relevance") {
       p<N>: pullRequest(number:<N>) {
         number state
         reviews(first:50)  { nodes { author { login } state submittedAt } }
         comments(first:100){ nodes { author { login } createdAt } }
       }
       # ...one aliased pullRequest(...) line per candidate PR number...
     } }'
   ```
   For each contributor, count their reviews (by `state`) and review comments with `submittedAt`/`createdAt` on or after 2026-03-01 (the search's `updated:` filter is coarse, so this date check is required), and record whether they approved or requested changes.

5. **Trace each PR to its ticket(s).** For every PR found (authored, co-authored, or reviewed), resolve the linked issue(s):
   - Prefer the structured link via GraphQL:
     ```
     gh api graphql -f query='
     { repository(owner:"opensearch-project", name:"search-relevance") {
         pullRequest(number: <N>) {
           closingIssuesReferences(first:10) { nodes { number title url state } }
         }
       } }'
     ```
   - Fall back to parsing the PR body for `Closes #N`, `Fixes #N`, `Resolves #N`, and `Related to #N` (including cross-repo `owner/repo#N` references).

6. **Direct tickets.** Also list issues in each repo opened or assigned to a contributor in the window, even if not yet linked to a merged PR, so open work is visible:
   ```
   gh issue list --repo opensearch-project/search-relevance --state all \
     --search "involves:<login> created:>=2026-03-01" --json number,title,url,state,author,assignees
   ```

7. **Consolidate.** De-duplicate tickets across PRs and across the two repos (a ticket may be referenced by several PRs, including a mix of authored and reviewed ones). Drop any PR whose only in-scope activity predates 2026-03-01.

## Report format

Markdown. Organize by repository, then list tickets. For each ticket include:

| Column | Notes |
|---|---|
| Ticket | Issue number linked to the issue URL (or "—" if the PR has no linked issue) |
| Title | Ticket title (or PR title when there is no ticket) |
| Contributor(s) & role | each contributor of interest involved, with their role per PR — `author`, `co-author`, or `reviewer (N reviews/comments, approved\|changes-requested)` |
| PR(s) | PR number(s) linked to the PR URL(s) |
| Merged | `YYYY-MM-DD` of the PR merge (latest, if several) |
| Status | Ticket state (`open` / `closed`), or PR status when there is no ticket |
| Summary | One line describing the work |

Include a summary section above the tables stating:
- Time window used (since 2026-03-01) and the date the report was generated.
- Per repository: per contributor, counts split by role — PRs authored, co-authored, and reviewed (substantial) — and the number of distinct tickets.
- Totals per contributor across both repos.
- Any PRs that could not be traced to a ticket (listed explicitly).

Group co-authored-only and review-only work into their own clearly labeled sub-sections, so authored work stays distinct from work where the contributor only reviewed.

## Publishing

Produce the report as a Markdown file `opensearch-srw-activity.md`. If asked to publish, create a **public** gist with `gh gist create --public opensearch-srw-activity.md` and return the gist URL. If a previous run created a gist, update it with `gh gist edit <gist-id> opensearch-srw-activity.md` instead of creating a new one.

## Caveats to keep in the report

- "Work since March 2026" is measured by PR **merge** date, not when the underlying issue was opened — an older ticket can have recent work, and a recently opened ticket may have no merged work yet.
- Contribution is detected from GitHub records: PR/commit authorship, `Co-authored-by:` trailers, and submitted reviews/review comments. Only GitHub-recorded review counts — informal review via Slack, meetings, or pairing is not captured. The "substantial review" bar is a heuristic threshold (see Scope).
- The two repos are released independently, so a ticket may be "done" in code but not yet reflected in a published `CHANGELOG.md` entry.
