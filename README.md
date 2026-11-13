# renovate-config

The Renovate presets every pitolabs repository extends, as
[`base-repo-standard`](https://github.com/pitolabs/builder-skills/tree/main/skills/base-repo-standard) asks. Public
because a public repository cannot read a private preset.

| File | Extended as | Sets |
|---|---|---|
| [`default.json`](default.json) | `local>pitolabs/renovate-config` | Renovate's `config:best-practices`, plus the rules below |
| [`automerge.json`](automerge.json) | `local>pitolabs/renovate-config:automerge` | part of the standard: patch, pin and digest-pin updates of dependencies at 1.0 or later (in one PR, `automerged updates`), and the weekly lockfile refresh, merge on green CI |
| [`automerge-minor.json`](automerge-minor.json) | `local>pitolabs/renovate-config:automerge-minor` | opt-in: minors too, in the same PR |

## What the default sets, and why

- **`config:best-practices`**: actions and Docker images pinned by digest, config migration PRs, dev dependencies
  pinned, abandoned packages flagged, npm releases held three days, lockfiles refreshed weekly.
- **PyPI releases wait three days too**, like npm: time for a malicious or broken release to be found and pulled
  before Renovate proposes it.
  Everything else waits 12 hours.
- **Automerge is part of the standard**, through `:automerge`: patch, pin and digest-pin updates of dependencies at
  1.0 or later, and the weekly lockfile refresh, merge once CI is green, so routine updates do not pile up. It needs
  CI that runs the repository's tests on pull requests (the base-repo-standard audit checks both). Minors, majors,
  digest updates and 0.x dependencies still wait for a human. `:automerge-minor` is opt-in, only where a minor cannot
  reach production unseen. The updates that merge themselves go in one PR per repository, `automerged updates`: each
  PR costs a CI run, and another each time it is brought up to date with `main`. A group automerges only when every
  update in it does, so it holds only what merges itself; minors without `:automerge-minor` keep their own PRs, and a
  repository's own `groupName` rules (later in its `renovate.json`) win.
- **Monday and Tuesday only (UTC)**: `schedule` and `lockFileMaintenance.schedule` are `* * * * 1-2`, and
  `updateNotScheduled: false` makes Renovate leave open PRs alone on the other days: no update with `main`, and no
  merge either (Renovate's default keeps bringing them up to date, so every push to `main` re-ran CI on every open
  automerge PR; in November 2026 that was over half the CI minutes of the private repositories, which pay for
  Actions). Two days, because Renovate merges one PR per run and the next PR must then be brought up to date and pass
  CI before it merges. To act on a PR on another day, tick its rebase checkbox or the Dependency Dashboard's.
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
  "extends": ["local>pitolabs/renovate-config", "local>pitolabs/renovate-config:automerge"]
}
```

## Change it

`main` takes changes only through a pull request whose `validate` (Renovate's config validator) and `base` checks
pass, and never a direct push, a force push or a deletion; there is no bypass. A change here reaches every repository
on its next Renovate run. This repository follows base-repo-standard and public-repo-standard (`sh scripts/setup.sh`
turns on the hooks and installs the pinned tools).

Merge a pull request locally once its checks pass: push its tested head to `main`, a fast-forward. GitHub records the
pull request as merged, and the commits keep the noreply identity of your git config:

```bash
git fetch origin && git push origin origin/<branch>:main   # the branch must be up to date with main
```

Do not merge on GitHub (merge, squash or rebase) until **Keep my email addresses private** is on
(github.com/settings/emails): GitHub makes that commit and records the merger with this setting, not with your git
config, so with it off your primary address becomes the merge's author or committer, in a public history, and the
`privacy` job refuses it from then on. Renovate's own merges carry its bot identity.
