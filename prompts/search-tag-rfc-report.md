Find every RFC across the `opensearch-project` GitHub org that falls within the Search TAG's charter, and report them grouped by repository so the TAG can see its full backlog of proposals in one place.

## Charter (source of truth for scope)

Read the live charter before each run — it can change — rather than relying on the summary below:

```
gh api repos/opensearch-project/technical-steering/contents/technical-advisory-groups/search-tag/charter.md --jq .content | base64 --decode
```

As of this writing, the Search TAG covers: core text search, vector search, semantic search, hybrid retrieval, relevance tuning and ranking, query engine architecture and performance, search at billion-vector scale (ANN/graph updates/filtered search), Lucene lifecycle governance and upstream alignment, agentic search, and ML integration with search (query understanding, embeddings, rerankers, learned ranking models). It explicitly excludes non-search domains (cluster management, ingestion pipelines) except where they directly affect search performance or functionality.

## Repositories

RFCs are **not confined to a fixed repo list** — any repo in the `opensearch-project` org can host one. Discover them org-wide rather than hardcoding a repo set. That said, treat these as near-certain in-scope repos and don't apply the keyword filter as strictly to them: `search-relevance`, `dashboards-search-relevance`, `neural-search`, `k-NN`, `opensearch-jvector`, `neural-sparse-cpp`, `opensearch-learning-to-rank-base`. The core `OpenSearch` repo hosts both search RFCs (query engine, retrievers, ranking) and many non-search ones (cluster coordination, ingestion, security) — always keyword/body-filter it.

## Data sources

- GitHub, via the `gh` CLI (see `gh.md` steering file for auth and the background-shell gotcha).
- **RFC convention:** OpenSearch RFCs are GitHub issues, identified by either the `RFC` label or an `[RFC]` prefix in the title — repos are inconsistent about which they use, so check both. (GitHub's search tokenizer strips brackets, so `"[RFC]" in:title` and `"RFC" in:title` return identical result sets — searching `"RFC" in:title` is sufficient.)
- The Search TAG project board, [github.com/orgs/opensearch-project/projects/45](https://github.com/orgs/opensearch-project/projects/45/), which already curates some proposals. Cross-referencing it requires the `read:project` token scope, which this project's default `gh` auth does **not** have — check with `gh project view 45 --owner opensearch-project`, and if it errors with a missing-scope message, skip that cross-reference and say so in the report rather than asking the user to re-auth mid-run.

## Method

1. **Collect every candidate, org-wide, paginated.** This report only cares about **open** RFCs — filter with `state:open` in the query itself rather than fetching closed ones and discarding them. The search API caps at 1000 results per query — page with `--paginate` and check `total_count` isn't being silently truncated. Run two queries and union the results (do not try to OR the two qualifiers into one query — keep them separate, per the general `gh` search reliability note in `gh.md`):
   ```
   gh api --paginate -X GET search/issues -f q='org:opensearch-project label:RFC state:open' -f per_page=100 \
     --jq '.items[] | {repo: (.repository_url | sub(".*/repos/";"")), number, title, url: .html_url, author: .user.login, labels: [.labels[].name], createdAt: .created_at, updatedAt: .updated_at}'

   gh api --paginate -X GET search/issues -f q='org:opensearch-project "RFC" in:title state:open' -f per_page=100 \
     --jq '.items[] | {repo: (.repository_url | sub(".*/repos/";"")), number, title, url: .html_url, author: .user.login, labels: [.labels[].name], createdAt: .created_at, updatedAt: .updated_at}'
   ```
   De-duplicate by `repo`+`number`.

2. **Prefilter fast, cheaply, on title + repo only** (no per-issue fetches yet — this set can be 500-1000+ items):
   - **Likely in-scope:** repo is one of the near-certain search repos listed above, OR the title contains a search/IR signal word — `search`, `retriev`, `vector`, `semantic`, `hybrid`, `relevan`, `rank`, `rerank`, `query`, `k-NN`/`kNN`/`ANN`, `embed`, `lucene`, `BM25`, `neural`, `sparse`, `agentic`, `RAG`.
   - **Likely out-of-scope:** everything else (e.g. security, ingestion pipelines, dashboards UI chrome, cluster ops, unrelated plugins) — titles like "RFC 2119"/"RFC 10008" appearing only as a citation inside an unrelated title should also fall here.
   - This is a coarse split to bound cost, not the final answer — don't drop the out-of-scope bucket, carry its count into the report (see Report format).

3. **Make the real relevance call on the in-scope + borderline bucket.** Read title and (where the title alone is ambiguous) the issue body to judge fit against the charter's specific responsibility areas — cite which one applies (e.g. "Roadmap Alignment", "Core Primitives and Scale", "Lucene Lifecycle Governance"). Batch body fetches with a single GraphQL query using per-issue aliases (same pattern as `srw-activity.md`) rather than looping `gh issue view` per item, to avoid the backgrounded-shell keychain hang described in `gh.md`:
   ```
   gh api graphql -f query='
   { repository(owner:"opensearch-project", name:"<repo>") {
       i<N>: issue(number:<N>) { title state createdAt url body author { login } }
       # ...one aliased issue(...) line per candidate number in this repo...
     } }'
   ```
   Group aliases per repo (one GraphQL call per repo with candidates, not one call for the whole org).

   **A `null` result for an `issue(number:...)` field means the number no longer resolves as an Issue** — almost always because it was converted to a GitHub Discussion after being indexed by search. Don't treat this as a bug to retry: drop it from the candidate set and list it in the report summary (repo#number) rather than silently vanishing it.

   **At scale (200+ candidates), don't read every body yourself inline** — it blows out context fast and these are independent per-item judgments with no cross-item dependency. Split the in-scope+borderline bucket into chunks of ~40-50 (group by repo, splitting any single repo that exceeds the chunk size), and fan each chunk out to a parallel subagent/fork. Each chunk-worker gets: the charter text (inherited if forked from a conversation that already fetched it, otherwise re-fetch), the chunk's records (repo, number, title, body, url, author, createdAt), and asks for one judged record back per input record — `in_scope` (bool), `charter_area` (string, null if excluded), `exclusion_reason` (short phrase, only when excluded) — written to a JSON file, not relayed through chat. Merge the per-chunk JSON files afterward. This keeps the actual judgment calls (the part that needs full body text) out of the orchestrating context entirely.

   **Domain-overlap calls seen in past runs, for consistency:**
   - `OpenSearch-Dashboards` issues about PPL, observability dashboards, logs/traces tooling, or an "OSD-MCP server" for those domains are **Observability TAG** territory, not Search TAG, even though they may contain the word "query" — exclude explicitly, don't let the keyword prefilter wave them through unchallenged.
   - `opensearch-benchmark` items: the charter explicitly names benchmarking (Responsibility 7, "Best Practices and User Experience"), so lean toward **including** benchmark RFCs — but flag ones that are purely generic benchmarking-tool infrastructure (not search-quality-specific) as borderline in the exclusion/inclusion reason, so a human can weigh in.

4. **Count unique human commenters, excluding the author, for each RFC that survives step 3.** This is a proxy for community engagement/discussion depth, separate from charter fit. Fetch comments in the same per-repo GraphQL batch as step 3 (one extra aliased field per issue, not a separate round-trip):
   ```
   gh api graphql -f query='
   { repository(owner:"opensearch-project", name:"<repo>") {
       i<N>: issue(number:<N>) {
         author { login }
         comments(first: 100) { totalCount nodes { author { login __typename } } }
       }
     } }'
   ```
   - Paginate `comments` with `pageInfo { hasNextPage endCursor }` if `totalCount` exceeds 100 (rare for RFCs, but don't silently truncate).
   - A commenter is **excluded** from the count if: their login matches the issue's own `author.login` (self-replies don't count as outside engagement), OR `__typename == "Bot"`, OR the login is a known bot (case-insensitive, also any login ending in `[bot]`) — reuse the bot list from `orphan-repos-report.md` (`dependabot[bot]`, `mend-for-github-com[bot]`, `renovate[bot]`, `github-actions[bot]`, `opensearch-trigger-bot[bot]`, `opensearch-ci-bot`, `opensearch-changeset-bot[bot]`, etc.).
   - Count **distinct remaining logins** (not comment count — one person commenting 5 times is 1 unique commenter, not 5).

5. **Cross-reference the project board**, if the `read:project` scope check in Data sources succeeds, and mark which RFCs are already tracked there vs. newly surfaced by this sweep.
   ```
   gh project item-list 45 --owner opensearch-project --format json --limit <N>
   ```
   The board can hold 1000+ items (it had 1885 as of this writing) — `gh project item-list` silently caps at whatever `--limit` you pass, and a too-low limit returns a partial list with no error. Check the response's `totalCount` against `items` length and raise `--limit` until they match, rather than trusting a default. Match by issue `url`, not number alone (numbers repeat across repos).

6. **Consolidate and de-duplicate** the final list, sorted within each repo by `createdAt` descending (newest first).

## Report format

Markdown, organized by repository (search-heavy repos like `search-relevance`, `neural-search`, `k-NN` first; then everything else alphabetically). For each RFC:

| Column | Notes |
|---|---|
| RFC | Issue number linked to the issue URL |
| Title | Issue title |
| Author | RFC author's GitHub login |
| Created | `YYYY-MM-DD` |
| Charter area | Which Responsibilities item(s) from the charter it maps to |
| Unique human commenters | Distinct commenter logins, excluding the author and any bots (step 4) — a proxy for how much outside discussion the proposal has drawn |
| On project board? | yes / no / unknown (scope unavailable) |

This report only includes **open** RFCs (see Method step 1) — closed ones (merged/accepted, declined, or stale) are out of scope entirely and not counted anywhere in this report.

Include a summary section above the tables:
- Date the report was generated and which charter revision was read.
- Total open candidates found (label + title sweeps, deduplicated), how many were kept vs. filtered as out-of-scope, and how many needed a body read to decide.
- Total in-scope open RFCs, and per-repo counts.
- Whether the project-board cross-reference ran, and if not, why (missing scope) — tell the user to run `gh auth refresh -s read:project` if they want that enrichment next time.

Add an appendix listing titles that were borderline and excluded, so a human can sanity-check the filter rather than trusting it blindly — this is a judgment call, not a deterministic query, and false negatives are more costly than false positives for a TAG backlog review.

## Publishing

Produce the report as a Markdown file `search-tag-rfcs.md`. If asked to publish, create a **public** gist with `gh gist create --public search-tag-rfcs.md` and return the gist URL. If a previous run created a gist, update it with `gh gist edit <gist-id> search-tag-rfcs.md` instead of creating a new one.

## Caveats to keep in the report

- RFC labeling/prefixing is a per-repo convention, not an org-wide enforced standard — some proposals that function as RFCs may use neither the label nor the title prefix and won't be found by this sweep. If the user knows of one that's missing, that's a signal to extend the search terms, not just add the one issue.
- The keyword prefilter is a recall/cost tradeoff. If the report looks too sparse, lower the bar (treat "likely out-of-scope" titles as borderline instead of dropping them) and re-run step 3 against a larger set.
- "In scope for the Search TAG charter" is inherently a judgment call for proposals that touch both search and another domain (e.g. an ingestion RFC that affects reindexing performance) — when in doubt, include it with a note explaining the overlap rather than silently excluding it.
- A null `issue(number:...)` result in step 3's GraphQL batch means the issue was likely converted to a GitHub Discussion since being indexed by search — not a transient API failure. Drop it and report the count/list, don't retry in a loop.
- Running the charter-fit judgment (step 3) as parallel chunked subagents means each chunk's reasoning is independent — one chunk can't see another's exclusions, so the same (author, title-pattern) pair could in principle be judged inconsistently across chunks. This hasn't caused visible drift so far (chunks are grouped by repo, so near-duplicate titles mostly land in the same chunk), but if the report ever shows two similar RFCs in different repos classified differently without an obvious reason, that's the likely cause — worth a quick manual reconciliation pass rather than assuming it's a charter-scope error.
