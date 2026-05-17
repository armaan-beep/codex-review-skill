# Codex Review Skill

Reusable Codex skill for PR review loops with verified findings, GitHub audit-trail comments, fresh-worktree closeout, and optional Crabbox remote proof runs.

The goal is simple: when you paste a PR into Codex and ask for review, Codex should inspect the code, inspect the PR conversation, verify comments from bots/humans, fix only real issues when asked, and leave a clear trail on GitHub.

## What It Does

- Runs `codex review` against local changes, a branch, a commit, or a PR base.
- Reads PR title/body, review threads, top-level comments, inline comments, reviews, and status checks.
- Treats CodeRabbit and other bot comments as candidate findings, not truth.
- Verifies findings before accepting them.
- Posts append-only GitHub comments for review runs, fixes, tests, Crabbox proof, cleanup, and final status.
- Limits review/fix loops to a default of five.
- Uses a fresh local worktree for final PR closeout.
- Uses Crabbox for optional remote clean-clone proof runs, including private repo support through your local GitHub token.
- Cleans up local worktree metadata and Crabbox resources.

## Requirements

Required:

- Codex CLI and app/CLI skill support
- `git`
- GitHub CLI, authenticated with `gh auth login`
- `jq`

Optional but recommended:

- `crabbox` for remote proof runs
- `shellcheck` for maintaining this repo

No credentials are committed in this repository. Runtime credentials are read from your local environment or tools, such as `gh auth token`, `GH_TOKEN`, `GITHUB_TOKEN`, and Crabbox/Hetzner setup.

## Install As A Global Codex Skill

Clone this repo:

```bash
git clone https://github.com/armaan-beep/codex-review-skill.git
cd codex-review-skill
```

Install the skill into your Codex skills directory:

```bash
scripts/install-skill
```

Restart Codex so it reloads skills.

To update later:

```bash
cd codex-review-skill
git pull
scripts/install-skill --force
```

## Add Repo Wrappers

Repo wrappers make the commands short and stable inside a project:

```bash
cd codex-review-skill
scripts/install-repo-adapter --repo /path/to/your/repo
```

This creates wrapper scripts like:

```bash
scripts/codex-review
scripts/codex-review-pr-context
scripts/codex-review-comment
scripts/codex-review-fresh-worktree
scripts/codex-review-cleanup
scripts/crabbox-pr-proof
```

Commit those wrappers to your project if you want every agent and teammate to have the same command surface. The wrappers do not contain credentials; they locate the installed skill or a vendored `.agents/skills/codex-review` copy at runtime.

## Vendor Into A Repo

For production teams, prefer a pinned repo-local copy over auto-pulling latest at review time:

```bash
mkdir -p /path/to/your/repo/.agents/skills
cp -R skills/codex-review /path/to/your/repo/.agents/skills/codex-review
scripts/install-repo-adapter --repo /path/to/your/repo
```

Update the vendored copy through a normal PR. That keeps review behavior auditable and prevents public upstream changes from silently changing private repo review policy.

## Basic Usage

From a PR branch:

```bash
scripts/codex-review-pr-context
scripts/codex-review --mode branch --github-comment --loop 1 --parallel-tests auto
```

With a PR URL:

```bash
scripts/codex-review-pr-context --pr https://github.com/OWNER/REPO/pull/123
scripts/codex-review --mode branch --pr https://github.com/OWNER/REPO/pull/123 --github-comment --loop 1 --parallel-tests auto
```

Fresh-worktree final review:

```bash
scripts/codex-review-fresh-worktree --pr 123 --github-comment --loop 5
```

Crabbox proof:

```bash
scripts/crabbox-pr-proof --pr 123 --shell "npm test"
```

Python proof:

```bash
scripts/crabbox-pr-proof --pr 123 --python-bootstrap --shell "pytest -q"
```

Cleanup:

```bash
scripts/codex-review-cleanup --require-clean-current --prune-worktrees --crabbox-list --crabbox-cleanup
```

## Recommended Prompt

In a fresh Codex session or fresh worktree, paste:

```text
Use the codex-review skill to review this PR end to end:
https://github.com/OWNER/REPO/pull/123

Read PR context and comments first, run the review loop, verify any findings before accepting them, post the GitHub audit trail, run focused verification, use fresh-worktree closeout, use Crabbox proof if this changes behavior, and clean up review resources.
```

The skill contains the detailed workflow, so you usually do not need a longer prompt.

## Repo-Specific Preflight

If your repo has `scripts/preflight`, `codex-review --parallel-tests auto` uses it.

If your repo does not, the bundled fallback only runs generic checks:

- `git diff --check`
- whitespace checks for untracked text files
- shell syntax checks for common script files

For better test selection, add a repo-local `scripts/preflight` that maps changed paths to lint/type/test commands. You can also set:

```bash
export CODEX_REVIEW_PREFLIGHT="your-command-here"
```

## Crabbox Setup

Install and authenticate Crabbox separately, then verify:

```bash
crabbox doctor
```

For private GitHub repos, `scripts/crabbox-pr-proof` forwards a GitHub token from `GH_TOKEN`, `GITHUB_TOKEN`, or `gh auth token` as a Crabbox secret for the remote clone. It does not write that token to the repo.

Optional templates live in `templates/crabbox/`.

## Maintainer Model

This is an open-source public repo. Anyone can fork and open PRs, but only repository owners or explicitly added collaborators can merge. Keep that default GitHub permission model unless you intentionally add maintainers.

For private/product repos, install or vendor this skill through a reviewed PR instead of live-pulling from `main` during review.

## Contributing

PRs are welcome. See `CONTRIBUTING.md`.

## License

MIT. See `LICENSE`.
