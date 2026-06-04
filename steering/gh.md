---
inclusion: always
---
# Listing Members of Opensearch Project Github Org

Members are maintainers who have explicit membership in the Opensearch-Project.

## Fetching Members

Use the `gh` CLI (already authenticated) to fetch org details for the opensearch-project:

```bash
# Member data (login, id, avatar_url, html_url, type)
gh api \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  /orgs/opensearch-project/members \
  --paginate
```
