---
name: coderabbit-triage
description: Triage and fix CodeRabbit review comments on the current PR. Use when the user says "do coderabbit", "address coderabbit", "fix coderabbit comments", "/coderabbit", or otherwise asks to work through CodeRabbit feedback on a pull request.
disable-model-invocation: true
---

# CodeRabbit triage workflow

Walk through every unresolved CodeRabbit comment on the current PR, decide what to do with each, apply fixes grouped by theme, and resolve the threads. Do **not** blindly accept CodeRabbit's suggestions — it hallucinates, over-engineers, and recommends unmaintained packages. Verify every claim against the actual code before acting.

## 1. Gather the comments

Detect the current PR from the branch:

```bash
gh pr view --json number,headRefName,baseRefName,url
```

Fetch CodeRabbit review comments (inline + summary). CodeRabbit posts as `coderabbitai[bot]`:

```bash
# Inline review comments (file + line anchored)
gh api "repos/{owner}/{repo}/pulls/<PR>/comments" --paginate \
  --jq '[.[] | select(.user.login == "coderabbitai[bot]") | {id, path, line, body, in_reply_to_id, html_url}]'

# Top-level review bodies (the "actionable comments" summaries)
gh api "repos/{owner}/{repo}/pulls/<PR>/reviews" --paginate \
  --jq '[.[] | select(.user.login == "coderabbitai[bot]") | {id, state, body, submitted_at, html_url}]'
```

Filter out threads that are already resolved. For GraphQL-resolved threads:

```bash
gh api graphql -f query='
  query($owner:String!, $repo:String!, $pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) {
          nodes { id isResolved comments(first:1){ nodes { author{login} path line body url } } }
        }
      }
    }
  }' -f owner=<owner> -f repo=<repo> -F pr=<PR>
```

## 2. Classify each comment

For every unresolved comment, read the referenced file/line and decide:

- **fix** — legitimate issue, real bug, real footgun, or genuine improvement. Apply it.
- **defer** — valid point but out of scope for this PR. Reply explaining, file a follow-up note in the PR description or a TODO if appropriate, then resolve.
- **reject** — wrong, hallucinated, doesn't apply to this context (e.g. SSR-defensive code on a SPA, recommending an unmaintained package, suggesting a "fix" for code that's already correct). Reply with the reason and resolve.

**Never** apply a fix just because CodeRabbit said so. Re-read the actual code at the cited `path:line` first. If the suggestion conflicts with project context (CLAUDE.md, ADRs, prior decisions), reject it.

Output the classification as a compact table before doing any edits:

```text
#  path:line                          verdict  one-line reason
1  infra/modules/alb/main.tf:42       fix      missing tag breaks cost allocation
2  apps/web/src/loading.tsx:11        reject   SSR guard suggested but this is SPA-only
3  backend/config/settings.py:88      defer    valid but scope creep — open follow-up
```

Wait for the user to confirm the classification before editing — unless the user has explicitly said "just go" / "auto" / similar.

## 3. Apply fixes — grouped by theme, one commit per group

Cluster the `fix` items by concern (e.g. "terraform tagging", "type-safety in loading flow", "doc cleanup"). One commit per cluster, not one commit per comment, not one giant commit mixing themes.

Commit message format:

```text
<area>: <what changed and why>

Addresses CodeRabbit:
- <short summary> (<comment url>)
- <short summary> (<comment url>)
```

## 4. Run the verify gate before pushing

Before committing each themed group, run the project's full verify gate. Prefer verify commands documented in the project's `CLAUDE.md`. Otherwise detect the ecosystem and run its equivalents of:

- typecheck
- lint
- test
- build (only if the project ships a build)

For JavaScript/TypeScript projects, detect the package manager from the lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` → bun, `package-lock.json` → npm) and invoke that manager's `run` command directly. Never assume a wrapper like `@antfu/ni` is installed — use it only if it is already present on the machine.

For Terraform changes: `terraform fmt -check`, `terraform validate`, then `terraform plan` against the relevant root. Eyeball the plan diff — do not push if the plan shows unrelated drift.

Do **not** skip hooks (`--no-verify`) or bypass signing. If a hook fails, fix the underlying problem and create a new commit.

## 5. Reply to and resolve every thread

For each comment, post a reply explaining the resolution, then mark the thread resolved.

```bash
# Reply to an inline comment
gh api "repos/{owner}/{repo}/pulls/<PR>/comments/<comment_id>/replies" \
  -X POST -f body="Fixed in <sha>: <one-line summary>."

# Resolve the thread (GraphQL — needs the thread node id from step 1)
gh api graphql -f query='
  mutation($id:ID!) { resolveReviewThread(input:{threadId:$id}) { thread { isResolved } } }
' -f id=<thread_node_id>
```

Reply templates:

- **fix**: `Fixed in <sha>: <what>.`
- **defer**: `Valid but out of scope for this PR — tracked as <link/TODO>. Resolving.`
- **reject**: `Not applicable here — <one-sentence reason, e.g. "this view is SPA-only, no SSR path">. Resolving.`

## 6. Push

Once every thread has a reply and the gate is green, push. Do not re-run the verify gate before pushing — it just ran.

```bash
git push
```

## Anti-patterns — do not do these

- Don't apply a CodeRabbit "fix" without reading the actual referenced code.
- Don't add an unmaintained or barely-maintained dependency because CodeRabbit suggested one — write the few lines inline.
- Don't mix unrelated themes into one commit just because CodeRabbit raised them on the same PR.
- Don't reach for `--no-verify` to dodge a failing pre-commit hook.
- Don't squash all CodeRabbit fixes into one "address review feedback" commit — the themed history is useful at squash-merge time.
- Don't reply "done" to a thread without resolving it; reply *and* resolve.
