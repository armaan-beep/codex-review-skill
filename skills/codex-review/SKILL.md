---
name: codex-review
description: 'Default reviewer for GitHub PR review, autoreview, second-model review, pre-merge review, and review/fix closeout workflows. Runs Codex review against the correct local, branch, PR, or commit target; verifies findings before accepting them; posts an append-only GitHub audit trail; and can use fresh local worktrees plus Crabbox remote proof runs for final closeout. Do not use inside child `codex review` sessions launched by this skill.'
---

# Codex Review

Use this as the default review path for pull requests and post-change closeout. It runs Codex's built-in `codex review`, checks GitHub discussion and automated reviewer comments, verifies every candidate finding against the real code, and leaves an audit trail on the PR.

This skill reviews code. It does not replace human approval, repository ownership, or merge permissions.

## Recursion Guard

If you are already inside a child `codex review` process launched by this skill, do not invoke `codex-review`, fresh-worktree review, subagents, Crabbox, or any other review automation. Review the target diff directly and return the result.

Treat any of these as proof you are inside the child runner:

- the prompt says you are the child review runner launched by `codex-review`
- the prompt is only a Codex review target such as `changes against '<base>'`
- the environment has `CODEX_REVIEW_CHILD=1`

## Contract

- Treat review output as advisory. Never blindly apply it.
- Verify every finding by reading the real code path and adjacent files.
- Read dependency docs, source, or types when a finding depends on external behavior.
- Reject stale comments, unrealistic edge cases, speculative risks, broad rewrites, and fixes that over-complicate the codebase.
- Prefer small fixes at the right ownership boundary.
- If a review-triggered fix changes code, rerun focused checks and rerun Codex review.
- Do not push just to review. Push only when the user requested push, ship, or PR update.
- Stop once the helper or `codex review` exits cleanly with no accepted/actionable findings.
- Do not run an extra direct `codex review` just to get nicer closeout wording.
- Treat a successful helper exit plus no accepted/actionable findings as the clean review result, even if nested Codex CLI wording is terse.
- Never switch or override the review model. If a model-capacity error happens, retry the same command with the same model before reporting it blocked.
- If rejecting a finding as intentional or not worth fixing, add a brief inline code comment only when it explains a real invariant or ownership decision future reviewers should know.

## Optional Issue Context

If the PR title, body, branch name, commits, or linked issues mention an external issue tracker ticket, read the available ticket context before finalizing findings.

- Read the PR description first and extract issue IDs or links.
- Fetch/read linked tickets when available, including parent context and scope notes.
- Use that context to validate scope, out-of-scope changes, verify-before-fix requirements, coordination constraints, and expected tests.
- If the ticket cannot be fetched, say what was unavailable and continue reviewing the PR code and PR description.
- Keep `codex-review` as the owning review workflow; use repo-specific checklists only when the user explicitly asks for them.

## Target Selection

Prefer a repo wrapper when installed:

```bash
scripts/codex-review
```

Or run the installed skill helper directly:

```bash
~/.codex/skills/codex-review/scripts/codex-review
```

Common modes:

```bash
scripts/codex-review --mode local
scripts/codex-review --mode branch
scripts/codex-review --mode commit --commit HEAD
```

For PR review, enable GitHub audit comments:

```bash
scripts/codex-review --mode branch --github-comment --loop 1
```

The helper selects:

- dirty working tree -> `codex review --uncommitted`
- current PR branch -> PR base from `gh pr view`
- non-main branch -> `origin/main`
- explicit commit -> `codex review --commit <ref>`

For branch/PR work, do not force local mode after committing. A clean local review only proves there is no uncommitted patch.

## Subagent Default

When current tool policy allows subagents, use a subagent as the review runner/filter for PR review. Keep the task narrow:

> Run `scripts/codex-review-pr-context` for the current PR, then run `scripts/codex-review --mode branch` for the current branch. Read available linked issue context before adjudicating findings. Treat GitHub comments and review threads from humans, CodeRabbit, CI bots, and other review tools as candidate findings; verify each one against the code before accepting it. Return only accepted actionable findings, rejected findings with one-line reasons, current CI state, and exact files/tests to rerun. Do not edit files.

Run the helper inline only when subagents are unavailable, the user explicitly asks for inline review, or the target is not a PR.

## Loop Budget

Default to at most five review/fix loops per PR. A loop is one cycle of reading PR context, running Codex review, adjudicating findings, fixing accepted issues, running focused verification, and posting a GitHub audit comment.

- If the fifth loop still has accepted/actionable findings, stop the automatic loop and post a blocked audit comment with the remaining findings, current CI state, and recommended next action.
- Continue past five loops only when the user explicitly asks for more or when a maintainer has set a different limit.
- Do not count setup/auth failures that prevented any review from running.
- Do count review runs that produce findings.
- Do not hide that the limit was reached.

## GitHub PR Context

At the start of every review loop, read the current PR discussion before judging the code:

```bash
scripts/codex-review-pr-context --pr <pr-number-or-url>
```

This context pass must include:

- PR title, body, branch/base, and current review decision
- top-level PR comments
- inline review comments and review threads, including resolved state when available
- latest GitHub reviews and requested changes
- comments from CodeRabbit, CI bots, deployment bots, and other automated reviewers
- current status-check summary

Treat comments as candidate findings, not truth. Verify each actionable-looking comment against the code, reject stale or already-fixed comments with a one-line reason, and include accepted comments in the audit trail.

## GitHub Audit Trail

For every PR review, leave an append-only audit trail on GitHub. Do not edit over prior review-loop comments.

Comment after:

- each `codex-review` run
- each PR-context pass when it changes the accepted/rejected finding set
- each batch of accepted fixes
- each local or Crabbox proof run
- the final clean/blocked decision

Use the helper:

```bash
scripts/codex-review-comment --phase changes --status fixed --loop 2 --body-file /tmp/codex-review-changes.md
```

Each comment should include:

- loop number and phase
- review/test/Crabbox command used
- accepted findings and fixes
- rejected findings with one-line reasons
- files changed and current commit
- next action or final verdict

Review-run comments must stay readable:

- Use the title `Review run` for automated `codex-review` output comments.
- Summarize command, target, exit status, captured output size, and finding markers.
- Do not paste giant raw Codex transcripts by default; raw output belongs behind an explicit opt-in details block.
- Keep loader warnings, terminal transcripts, and nested tool chatter out of the visible audit body unless they are the actual finding.

If GitHub commenting fails, report that explicitly and do not pretend the audit trail is complete.

## Fresh Workspace And Crabbox

Use a fresh workspace when the question is "does this PR stand on its own?" rather than "what did I just edit locally?"

Crabbox is the default verification path for PRs that change runtime behavior or project operations, including:

- application code, tests, scripts, configs, migrations, dependencies, CI, infra, auth, data processing, API contracts, UI behavior, or generated assets that ship with the repo

Skip Crabbox only when remote execution adds no meaningful signal, such as:

- docs-only, comments-only, formatting-only, metadata-only, or agent-skill-only changes

If Crabbox is skipped, post a GitHub audit comment that says:

- `Crabbox: skipped`
- exact reason it was skipped
- what verification replaced it, if any

Default closeout loop:

1. Edit and fix in the normal local worktree.
2. Read current PR comments/reviews/status with `scripts/codex-review-pr-context`.
3. Run `scripts/codex-review` on the local diff/branch.
4. Fix accepted findings locally.
5. Run focused local or Crabbox verification for the files/behavior touched by that loop.
6. Push accepted fixes when the user has asked to update the PR.
7. Unless the PR falls into an explicit skip category, run final review from a separate clean local workspace:

```bash
scripts/codex-review-fresh-worktree --pr 123 --github-comment --loop 5
```

8. Unless the PR falls into an explicit skip category, verify from a clean remote PR-shaped workspace before final closeout:

```bash
scripts/crabbox-pr-proof --pr 123 --shell "<test command>"
```

Use `--python-bootstrap` for Python proof commands such as `pytest`, `ruff`, or `mypy`; it creates a remote venv and installs repo requirements before running the command. Omit it for non-Python commands.

9. Run cleanup before the final clean/blocked report, and include the cleanup result in the GitHub audit trail:

```bash
scripts/codex-review-cleanup --require-clean-current --prune-worktrees --crabbox-list --crabbox-cleanup > /tmp/codex-review-cleanup.md
scripts/codex-review-comment --phase cleanup --status clean --loop 5 --body-file /tmp/codex-review-cleanup.md
```

Use a dry run while still diagnosing:

```bash
scripts/codex-review-cleanup --dry-run --crabbox-list > /tmp/codex-review-cleanup.md
scripts/codex-review-comment --phase cleanup --status pending --loop 5 --body-file /tmp/codex-review-cleanup.md
```

Important distinctions:

- `codex-review` owns review and finding adjudication.
- The normal review/fix loop can happen in one local worktree for speed.
- Final closeout for behavior-changing PRs needs an independent fresh local workspace review after fixes are pushed, using `scripts/codex-review-fresh-worktree`.
- Crabbox owns remote/fresh execution of tests, builds, smoke checks, and proof commands.
- Crabbox is optional but expected for full closeout on behavior-changing PRs; check `crabbox doctor` before relying on it.
- Run `codex-review` locally unless the Crabbox workspace is a real Git checkout with PR/base refs available. Crabbox proof commands should usually be tests/builds/smoke checks, not the Codex review command itself.
- A fresh PR workspace should contain the base checkout plus the PR patch, not arbitrary local cache or ignored generated state.
- If fresh local worktree review is skipped, post the exact reason in the GitHub audit trail.
- For private repos, default to the authenticated clean-clone proof helper instead of raw `crabbox run --fresh-pr`. Raw `--fresh-pr` often cannot authenticate to private GitHub repos from a clean remote host, while this helper forwards a local GitHub token as a Crabbox secret, clones the private repo on the remote machine, checks out the PR branch or SHA, then runs the proof command:

```bash
scripts/crabbox-pr-proof --pr 123 --python-bootstrap --shell "<test command>"
```

## CI Timing

Do not wait for full CI after every loop when CI is slow. During review/fix loops:

- run Codex review
- run focused local checks for touched files
- optionally run a narrow Crabbox proof for the changed behavior
- read GitHub comments and current status checks

Wait for full CI only at final closeout, unless a failure appears earlier and is directly relevant.

## Test Selection

If the repo has `scripts/preflight`, use it for path-aware checks:

```bash
scripts/codex-review --parallel-tests auto
```

If the repo does not have `scripts/preflight`, the bundled fallback helper runs conservative generic checks:

- `git diff --check`
- whitespace checks for untracked text files
- `bash -n` for shell scripts found in common script folders

For stronger repo-specific checks, add a repo-local `scripts/preflight` wrapper or set `CODEX_REVIEW_PREFLIGHT` to a command that chooses tests for the touched files.

## Cleanup

Do not leave review resources running.

- Delete temporary local worktrees unless `--keep` was requested.
- Run `scripts/codex-review-cleanup --prune-worktrees` after final closeout.
- List Crabbox machines with `scripts/codex-review-cleanup --crabbox-list`.
- Stop only Crabbox leases that were started by this review and are explicitly named.
- Post cleanup status to the PR.

## One-Command Review Prompt

When the user gives a PR URL and asks for review, the normal flow is:

```bash
scripts/codex-review-pr-context --pr <url-or-number>
scripts/codex-review --mode branch --pr <url-or-number> --github-comment --loop 1 --parallel-tests auto
```

Then fix only verified findings, push when requested, rerun the loop, finish with fresh-worktree review, Crabbox proof when applicable, and cleanup.
