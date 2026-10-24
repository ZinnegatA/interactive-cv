# ADR-001: Git workflow - squash-merge with Conventional Commit PR titles

## Context

`master` history should be readable and consistent from the start, and usable later for changelog/version automation. This is a solo project, so branch-level commit discipline adds friction without much value.

## Decision

- PRs are merged into `master` via **squash-merge only**; the PR title becomes the commit message.
- PR titles follow **Conventional Commits**: `type(scope)?: description`.
  - Allowed types: `feat`, `fix`, `docs`, `refactor`, `chore`.
  - Scope is optional until module boundaries settle.
- A required CI check validates PR titles. A local `commit-msg` hook may be added later as a convenience.
- Commits on feature branches are not constrained.

## Alternatives considered

- **Rebase-merge with per-commit linting:** every branch commit lands on `master`, so each one must be linted and kept clean.
- **Merge commits:** cluttered history with no benefit for a single contributor.
- **No convention:** inconsistent history and no path to automated changelogs.

## Consequences

- `master` has one conventional commit per PR.
- Granular branch history is lost after merge.
- A PR should represent one logical change, since its title describes everything squashed into it.
- Commits merged before this ADR (the initial one) are not rewritten.
