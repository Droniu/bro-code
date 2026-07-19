---
name: ship
description: Ship the current work end-to-end — branch off the correct base (fresh `main`, or a milestone/epic integration branch), run the project's full verify gate, commit per repo conventions, push, and open a PR with a properly-structured description. Use when the user says "/ship", "ship it", "ship this", "open a PR for this", "wrap this up and push", or similar.
disable-model-invocation: true
---

# Ship workflow

One-shot: take the current working state to a pushed branch with an open PR. Be deliberate about destructive operations — when state is ambiguous, ask the user before resetting, switching branches, or stashing.

## 1. Read the current state before doing anything

Run these in parallel:

```bash
git status
git branch --show-current
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || true
git log --oneline -5 origin/main..HEAD 2>/dev/null || true
git diff --stat HEAD
gh pr view --json state,headRefName,baseRefName,url 2>/dev/null || true
```

Classify the branch situation into one of three states:

- **A. On `main` (or `master`)** — clean or with uncommitted changes → create a new branch.
- **B. On a feature branch still in flight** — PR is open (or no PR yet) AND the commits look related to the current uncommitted/staged diff → reuse this branch.
- **C. On a feature branch whose PR is already merged**, OR commits clearly unrelated to the current diff (different feature area, stale work) → switch to the **base branch** (see step 2 — usually `main`, but it may be an epic/integration branch), pull, create a new branch. Stash uncommitted changes first.

If you can't decide between B and C, ask the user with `AskUserQuestion`. Show them: current branch, last 3 commits on it, the diff summary, and the PR state if any.

In every "create a new branch" case (A and C), the new branch is cut from the **base branch determined in step 2** — which is *not* always `main`.

## 2. Determine the base branch, then refresh it

**The base is not always `main`.** A milestone or epic often integrates its tickets on a shared branch (e.g. `feat/<area>`, `epic/<x>`, `integration/*`, `develop`, `release/*`): each ticket branches off and PRs *into* that branch, and the whole thing lands on `main` as a single unit at the end. Shipping such a ticket against `main` strands it ahead of the milestone's shared work and produces dangling cross-references (e.g. to a spec/design doc that only exists on the epic branch). This is a silent, easy-to-miss mistake — detect the base deliberately.

Determine the base **before** branching:

1. **Inspect the prelude/sibling PRs.** If a related ticket (same milestone — often a docs/spec/scaffold prelude) already merged, its base is almost certainly yours too. Do **not** assume a merged PR went to `main`:

   ```bash
   gh pr view <merged-sibling-pr> --json baseRefName,headRefName,state
   gh pr list --state all --search "<milestone keyword or ticket family>" \
     --json number,title,headRefName,baseRefName,state --limit 10
   ```

   A non-`main` base that recurs across the ticket family **is** the integration branch.
2. **Scan origin for an integration branch ahead of `main`:**

   ```bash
   git fetch origin
   git for-each-ref --format='%(refname:short)' refs/remotes/origin \
     | grep -iE 'epic/|feat/|integration/|milestone/|develop|release/' || true
   ```

   For a candidate `origin/<branch>`, confirm it carries related milestone work and how far it leads `main`: `git log --oneline main..origin/<branch>`.
3. **Use conversation / issue-tracker context** — the milestone may name its integration branch explicitly, or the ticket may be one of a numbered family (e.g. `TICKET-200..206`) whose siblings reveal the base.

If a plausible integration branch exists, **confirm the base with the user via `AskUserQuestion`** before proceeding — base choice is consequential and hard to notice once the PR is open. If none exists, the base is `main`.

Call the chosen base **`$BASE`** (it may be `main`). **Only when creating a new branch**, refresh it:

```bash
git checkout "$BASE"
git pull --ff-only
```

If `pull --ff-only` fails (diverged, conflicts, weird state) — **stop and ask the user**. Do not `git reset --hard`, do not `git pull --rebase` automatically.

If there were uncommitted changes on the prior branch, stash with a descriptive message first (before `git checkout "$BASE"`):

```bash
git stash push -m "ship: pre-switch from <old-branch>"
```

After creating the new branch in step 3, pop the stash there.

## 3. Decide ticket, slug, and branch name

**Ticket inference:** scan recent conversation context, the current branch name, and recent `git log` subjects for a ticket ID matching the common tracker pattern `[A-Z]+-\d+` (Jira/Linear style). Learn which scopes this repo actually uses from its history — never assume specific prefixes. If you find a confident match, use it.

If no ticket is in context, ask with `AskUserQuestion`. Offer three options:

1. **Provide ticket ID** — user types e.g. `PROJ-123` via Other
2. **Use inferred candidate** — only include this option if you have a weak/plausible inference to surface
3. **No ticket** — proceed without a ticket prefix

**Slug rules:**

- Lowercase kebab-case derived from the *intent* of the change, not from file names
- ≤ 50 chars
- Branch type prefix: `feature/` for new functionality, `fix/` for bugfixes, `chore/` for tooling/deps/docs, `ci/` for workflow-only. If the repo's existing branches follow a different scheme, match the repo.

**Final branch name:**

- With ticket: `feature/proj-123-add-fuzzy-search` (lowercase ticket)
- Without ticket: `feature/add-fuzzy-search`

```bash
git checkout -b <branch-name>
git stash pop 2>/dev/null || true   # only if you stashed in step 2
```

## 4. Run the full verify gate per touched project

Detect which subprojects the diff touches (use `git diff --name-only HEAD` or staged paths). For each touched subproject, discover and run its full verify gate sequentially. State the commands you're about to run before running them.

Discovery ladder — first hit wins, but a discovered target is a candidate, not proof of the complete gate; confirm it looks like the project's real gate before relying on it:

1. Verify commands documented in the project's `CLAUDE.md`.
2. Task-runner targets: `Makefile`, `justfile`, `Taskfile.yml` (`lint`, `test`, `typecheck`, `format`).
3. Manifest-native scripts for the detected ecosystem: `package.json` scripts via the package manager detected from the lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` → bun, `package-lock.json` → npm — never assume a wrapper like `nr` exists); `pyproject.toml` / tox / nox for Python; `cargo check && cargo test` for Rust; `go vet ./... && go test ./...` for Go.
4. Pre-commit hook configuration (`.pre-commit-config.yaml`, husky/lint-staged).
5. Ask the user once; suggest recording the answer in the project `CLAUDE.md`.

For Terraform/IaC directories: `terraform fmt -check -recursive` and `terraform validate` per touched root.

If any check fails: **stop**, surface the failure, fix the root cause (or ask the user how to proceed), then re-run only the failing project's gate before continuing.

Never use `--no-verify` or skip hooks. If a pre-commit hook fails, diagnose and fix.

## 5. Stage and commit per repo conventions

Sample the repo's commit style first: `git log --oneline -20 origin/main`. Mirror what you find. If the history shows no consistent convention, fall back to Conventional Commits with an optional ticket suffix:

```text
<type>(<scope>): <imperative summary> (TICKET-N)
```

- **type**: `feat | fix | chore | ci | refactor | docs | test`
- **scope**: subproject or area; omit if the change is genuinely global
- **ticket suffix**: include `(PROJ-123)` only if a ticket exists AND the repo's history uses that pattern
- Imperative mood, subject ≤ 72 chars, body wrapped at 72

Write a body when the change is non-trivial: 1–3 short paragraphs explaining the *why*, not the *what*. Skip the body for tiny/obvious changes.

**Stage explicit paths.** Never `git add -A` or `git add .` — avoid sweeping in `.env`, scratch files, secrets.

```bash
git add path/one path/two
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject> (<TICKET>)

<optional body explaining the why>
EOF
)"
```

If pre-commit hooks (formatters, lint-staged) rewrote files: re-stage the modified files and create a **new** commit. Never `--amend` to absorb hook rewrites.

## 6. Push with upstream tracking

```bash
git push -u origin HEAD
```

If the push is rejected (branch protection, missing permission), surface the error and stop — do not force-push.

## 7. Open the PR

**Title**: the first commit's subject, without any trailing `(#PR)` suffix (GitHub adds that on merge).

**Body** — sample the repo's own merged-PR style first and mirror its structure:

```bash
gh pr list --state merged --limit 5 --json number,title,body
```

If there is no consistent house style, use this structure:

```text
## Summary

<TLDR — exactly 2 sentences, no bullets. Sentence 1: what the problem is. Sentence 2: how this PR solves it.>

## Context

<2-3 paragraphs explaining the context of the change.>

## Test plan

- [x] <verify command that ran green in step 4>
- [x] <next verify command>
- [ ] <manual smoke step, if applicable>
```

Include these sections **only when they apply**:

```text
## Companion PR
- <link to related PR in another repo>

## Merge order
1. <this PR>
2. <follow-up>
```

If a ticket exists, end the body with a tracker link line — but only if the sampled merged PRs use one; copy their exact pattern (e.g. a `Closes PROJ-123` line or a full issue URL) rather than inventing a format.

Create the PR with the base from step 2 (`$BASE` — `main` unless the work belongs to an epic/integration branch):

```bash
gh pr create --base "$BASE" --title "<title>" --body "$(cat <<'EOF'
## Summary
...

## Test plan
- [x] ...
EOF
)"
```

Return the PR URL to the user. If you targeted an integration branch, say so explicitly (e.g. "opened against `feat/schedules`, which lands on `main` with the rest of the milestone").

## Anti-patterns — do not do these

- Don't `git reset --hard` or `git pull --rebase` to "fix" a non-fast-forward main — stop and ask.
- Don't `git add -A` / `git add .` — stage explicit paths.
- Don't `--no-verify`, `--no-gpg-sign`, or otherwise dodge hooks.
- Don't `--amend` after a pre-commit hook rewrote files; create a new commit.
- Don't skip the verify gate "because the change is small".
- Don't fabricate a ticket ID; if not in context, ask or go ticketless.
- Don't include `(#PR)` in the PR title — GitHub appends that on merge.
- Don't push to `main` directly. Ever.
- Don't assume the base is `main`. A milestone's tickets often stack on an epic/integration branch (step 2); detect it and confirm. Shipping onto `main` ahead of the epic strands the work and breaks cross-references to specs/docs that live only on the epic branch.
- Don't infer a merged PR's target from its "MERGED" state — check `gh pr view <pr> --json baseRefName`. A sibling ticket merged into `feat/<area>`, not `main`, is the signal that `feat/<area>` is your base.
- Don't open the PR before the verify gate is green and the push succeeded.
- Don't detect "branch already merged" by vibes — check `gh pr view --json state,baseRefName` and `git log origin/<base>..HEAD`. Ask if ambiguous.
- Don't force-push to recover from a rejected push without explicit user say-so.
