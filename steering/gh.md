---
inclusion: always
---
# GitHub CLI (`gh`)

General guidance for using the `gh` CLI in this project. Prompts should rely on this file rather than restating auth details.

## Authentication

- The `gh` CLI is **already authenticated** — do not run `gh auth login`.
- Available token scopes: `repo`, `read:org`, `gist` (also `admin:public_key`). This covers reading repos/issues/PRs, listing org members, and creating/editing gists.

## Operational gotcha: `gh` hangs in background shells

`gh` reads its token from the macOS keychain, which is only reachable from a foreground/interactive shell. In a **detached or backgrounded shell** (e.g. `run_in_background`, or a long loop the harness auto-backgrounds) `gh` hangs indefinitely on keychain access and produces no output.

Avoid this two ways:

- **Prefer one fast foreground call over many.** Batch work into a single request — e.g. a GraphQL query with aliases for many repos/users — instead of a per-item loop that runs long enough to get backgrounded.
- **For any backgrounded `gh`, pass the token via env** so it never touches the keychain:
  ```bash
  export GH_TOKEN="$(gh auth token)"   # run this in the foreground first
  # subsequent gh calls in the loop use $GH_TOKEN, no keychain access
  ```

## Fetching org members

Members are maintainers who have explicit membership in the Opensearch-Project.

```bash
# Member data (login, id, avatar_url, html_url, type)
gh api \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  /orgs/opensearch-project/members \
  --paginate
```
