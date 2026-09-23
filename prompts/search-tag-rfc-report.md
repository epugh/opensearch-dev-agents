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

1. **Collect every candidate, org-wide, paginated.** The search API caps at 1000 results per query — page with `--paginate` and check `total_count` isn't being silently truncated. Run two queries and union the results (do not try to OR the two qualifiers into one query — keep them separate, per the general `gh` search reliability note in `gh.md`):
   ```
   gh api --paginate -X GET search/issues -f q='org:opensearch-project label:RFC' -f per_page=100 \
     --jq '.items[] | {repo: (.repository_url | sub(".*/repos/";"")), number, title, url: .html_url, state, labels: [.labels[].name], createdAt: .created_at, updatedAt: .updated_at}'

   gh api --paginate -X GET search/issues -f q='org:opensearch-project "RFC" in:title' -f per_page=100 \
     --jq '.items[] | {repo: (.repository_url | sub(".*/repos/";"")), number, title, url: .html_url, state, labels: [.labels[].name], createdAt: .created_at, updatedAt: .updated_at}'
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
       i<N>: issue(number:<N>) { title state createdAt url body }
       # ...one aliased issue(...) line per candidate number in this repo...
     } }'
   ```
   Group aliases per repo (one GraphQL call per repo with candidates, not one call for the whole org).

4. **Cross-reference the project board**, if the `read:project` scope check in Data sources succeeds, and mark which RFCs are already tracked there vs. newly surfaced by this sweep.

5. **Consolidate and de-duplicate** the final list, sorted within each repo by `createdAt` descending (newest first).

## Report format

Markdown, organized by repository (search-heavy repos like `search-relevance`, `neural-search`, `k-NN` first; then everything else alphabetically). For each RFC:

| Column | Notes |
|---|---|
| RFC | Issue number linked to the issue URL |
| Title | Issue title |
| State | `open` / `closed`, and if closed whether it looks accepted/implemented/declined (from labels or final comments, best-effort) |
| Created | `YYYY-MM-DD` |
| Charter area | Which Responsibilities item(s) from the charter it maps to |
| On project board? | yes / no / unknown (scope unavailable) |

Include a summary section above the tables:
- Date the report was generated and which charter revision was read.
- Total candidates found (label + title sweeps, deduplicated), how many were kept vs. filtered as out-of-scope, and how many needed a body read to decide.
- Total in-scope RFCs split by state: **open** vs **closed** — and for closed ones, break down further where determinable from labels/final comments (accepted & implemented / accepted, not yet implemented / declined / stale-closed) rather than lumping all closed RFCs together, since "closed" conflates very different outcomes for a backlog review.
- Per-repo counts of in-scope RFCs, each also split open/closed.
- Whether the project-board cross-reference ran, and if not, why (missing scope) — tell the user to run `gh auth refresh -s read:project` if they want that enrichment next time.

Add an appendix listing titles that were borderline and excluded, so a human can sanity-check the filter rather than trusting it blindly — this is a judgment call, not a deterministic query, and false negatives are more costly than false positives for a TAG backlog review.

## Publishing

Produce the report as a Markdown file `search-tag-rfcs.md`. If asked to publish, create a **public** gist with `gh gist create --public search-tag-rfcs.md` and return the gist URL. If a previous run created a gist, update it with `gh gist edit <gist-id> search-tag-rfcs.md` instead of creating a new one.

## Caveats to keep in the report

- RFC labeling/prefixing is a per-repo convention, not an org-wide enforced standard — some proposals that function as RFCs may use neither the label nor the title prefix and won't be found by this sweep. If the user knows of one that's missing, that's a signal to extend the search terms, not just add the one issue.
- The keyword prefilter is a recall/cost tradeoff. If the report looks too sparse, lower the bar (treat "likely out-of-scope" titles as borderline instead of dropping them) and re-run step 3 against a larger set.
- "In scope for the Search TAG charter" is inherently a judgment call for proposals that touch both search and another domain (e.g. an ingestion RFC that affects reindexing performance) — when in doubt, include it with a note explaining the overlap rather than silently excluding it.
