# Contributing

Thanks for improving `codex-review`.

## Development

Run the local checks before opening a PR:

```bash
find scripts skills/codex-review/scripts -type f -print0 | xargs -0 -n1 bash -n
skills/codex-review/scripts/codex-review-preflight --all --dry-run
```

If `shellcheck` is installed:

```bash
shellcheck scripts/install-skill scripts/install-repo-adapter skills/codex-review/scripts/*
```

## PR Guidelines

- Keep changes focused.
- Do not commit credentials, tokens, local machine paths, private repo names, or private workflow assumptions.
- Update `README.md` when changing installation or user-facing behavior.
- Update `skills/codex-review/SKILL.md` when changing the review workflow.
- Prefer backwards-compatible helper changes; teams may vendor a pinned copy.

## Merge Policy

Anyone can open a PR. Only the repository owner or explicitly added collaborators can merge. External contributions should be reviewed through the normal GitHub PR flow.
