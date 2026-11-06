# renovate-config

The Renovate presets every pitolabs repository extends, as
[`base-repo-standard`](https://github.com/pitolabs/builder-skills/tree/main/skills/base-repo-standard) asks. Public
because a public repository cannot read a private preset.

| File | Extended as | Sets |
|---|---|---|
| [`default.json`](default.json) | `local>pitolabs/renovate-config` | Renovate's `config:best-practices`, plus the rules below; no automerge |
| [`automerge.json`](automerge.json) | `local>pitolabs/renovate-config:automerge` | opt-in: patch, pin and digest-pin updates of dependencies at 1.0 or later, and the weekly lockfile refresh, merge on green CI |
| [`automerge-minor.json`](automerge-minor.json) | `local>pitolabs/renovate-config:automerge-minor` | second opt-in: minors too |

## What the default sets, and why

- **`config:best-practices`**: actions and Docker images pinned by digest, config migration PRs, dev dependencies
  pinned, abandoned packages flagged, npm releases held three days, lockfiles refreshed weekly.
- **PyPI releases wait three days too**, like npm: time for a malicious or broken release to be found and pulled
  before Renovate proposes it.
  Everything else waits 12 hours.
- **No automerge by default.** A repository opts in with `:automerge` only once its CI runs its tests on pull
  requests (the base-repo-standard audit checks it), and after one update went from PR to tests to a manual merge.
  `:automerge-minor` only where a minor cannot reach production unseen.
- **Majors** are labelled `major-update` and wait for a human.
- **Digest updates** (same tag, new content: the upstream re-pushed it) are labelled `upstream-repush` and never
  automerge: in March 2026, `trivy-action`'s tags were re-pushed to other commits.
- **Vulnerability alerts** (from Dependabot alerts, which must be on in the repository) open a PR labelled `security`
  at once, outside the schedule and hourly limit.
- `prHourlyLimit: 0` (the default of 2 held PRs back for hours), `platformAutomerge: false` and `ignoreTests: false`:
  Renovate itself merges, only after the checks pass. It merges one PR per branch per run; the Mend-hosted app runs
  every few hours. To run it now, tick the checkbox at the bottom of the Dependency Dashboard issue.
- `mode: full` takes the repository out of Mend's Silent mode (the default when the app is installed for all
  repositories). If no Dependency Dashboard issue appears after two runs, turn it off by hand: developer.mend.io → the
  repository → Settings → Dependencies → Silent mode off, Automated PRs on.

Repository-specific managers, groups and labels stay in that repository's `renovate.json`.

## Adopt it

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>pitolabs/renovate-config"]
}
```

## Change it

`main` takes changes only through a pull request whose `validate` (Renovate's config validator) and `base` checks
pass, and never a direct push, a force push or a deletion; there is no bypass. A change here reaches every repository
on its next Renovate run. This repository follows base-repo-standard and public-repo-standard (`sh scripts/setup.sh`
turns on the hooks and installs the pinned tools).

Before you merge a pull request here, turn on **Keep my email addresses private** (github.com/settings/emails).
GitHub makes the commit when it merges (merge, squash or rebase) and records the merger with that setting, not with
your git config: with it off, your primary address becomes the merge's author or committer, in a public history, and
the `privacy` job refuses it from then on. Renovate's own merges carry its bot identity.
